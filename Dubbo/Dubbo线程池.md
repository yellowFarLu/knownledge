# 线程模型



## Netty的线程模型

![image-20191029094828398](https://tva1.sinaimg.cn/large/006y8mN6gy1g8etx8e1i8j30aj0c8jso.jpg)

Netty线程模型基于NIO实现，即非阻塞IO及多路复用。在netty中存在两种线程：boss线程和worker线程。



### boss线程

作用：

- accept（接收）客户端的连接；
- 将接收到的连接注册到一个worker线程上

个数：

- 通常情况下，服务端每绑定一个端口，开启一个boss线程





### worker线程

作用：

- 处理注册在其身上的connection（连接）的各种io事件

个数：

- 默认是：核数+1

注意：

- 一个worker线程可以注册多个connection
- 一个connection只能注册在一个worker线程上





## Dubbo 线程池模型

![image-20191029102434267](https://tva1.sinaimg.cn/large/006y8mN6gy1g8euys7n7pj30of09omze.jpg)

**整体步骤：（受限于派发策略，以默认的all为例, 以netty4为例）**

1. 客户端的用户线程发出一个请求后获得future，在执行get时进行阻塞等待；
2. 服务端使用worker线程（netty通信模型）接收到请求后，将请求提交到server线程池中进行处理
3. server线程处理完成之后，将相应结果返回给客户端的worker线程池（netty通信模型），最后，worker线程将响应结果提交到client线程池进行处理
4. client线程将响应结果填充到future中，然后唤醒等待的主线程，主线程获取结果，返回给客户端





### 服务端

两种线程池：

- io线程池：netty的boss和worker线程池。
  - boss：建立connection
  - worker：处理注册在其身上的连接connection上的各种io事件
  - IO线程负责处理连接、各种IO事件
- 业务线程池：**fixedThreadPool()**
  - 与worker配合处理各种请求



### 客户端

两种线程池：

- io线程池：netty的boss和worker线程池
  - 同上
- 业务线程池：**cached**ThreadPool
  - 与worker配合处理各种响应，最后得到响应后唤醒被阻塞的主线程



### **事件派发策略**

如果事件处理的逻辑能迅速完成，并且不会发起新的 IO 请求，比如只是在内存中记个标识，则直接在 IO 线程上处理更快，因为减少了线程池调度。

但如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。

如果用 IO 线程处理事件，又在事件处理过程中发起新的 IO 请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。

因此，需要通过不同的派发策略和不同的线程池配置的组合来应对不同的场景:

```xml
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```



1、dubbo基于netty。有5种派发策略：

- 默认是all：所有消息都派发到业务线程池，包括请求，响应，连接事件，断开事件，心跳等。
- direct：所有消息都不派发到业务线程池，全部在 IO 线程上直接执行。
- message：只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO线程上执行
- execution：只请求消息派发到线程池，不含响应（客户端线程池），响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行
- connection：将连接断开事件放入队列，在 IO 线程上有序逐个执行，其它消息派发到线程池。 



2、业务线程池：

- fixed：固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
  - coresize：200
  - maxsize：200
  - 队列：SynchronousQueue
  - 回绝策略：AbortPolicyWithReport - 打印线程信息jstack，之后抛出异常
- cached：缓存线程池，空闲一分钟自动删除，需要时重建。
- limited：可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
- `eager` ：优先创建`Worker`线程池。在任务数量大于`corePoolSize`但是小于`maximumPoolSize`时，优先创建`Worker`来处理任务。当任务数量大于`maximumPoolSize`时，将任务放入阻塞队列中。阻塞队列充满时抛出`RejectedExecutionException`。(相比于`cached`:`cached`在任务数量超过`maximumPoolSize`时直接抛出异常而不是将任务放入阻塞队列)



### 异步调用

从v2.7.0开始，Dubbo的所有异步编程接口开始以[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)为基础

基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。

![image-20191029131228622](https://tva1.sinaimg.cn/large/006y8mN6gy1g8ezthm40jj30h105fq4w.jpg)

- 如果是同步调用，返回一个future给用户线程，并且调用get()方法，阻塞业务线程直到完成调用

  ```java
  RpcContext.getContext().setFuture(null);
  return (Result) currentClient.request(inv, timeout).get();
  ```

- 如果异步调用，会将Future保存在RpcContext中，返回一个空结果给用户线程。当提供方返回结果的时候，

  ```java
  ResponseFuture future = currentClient.request(inv, timeout) ;
  RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
  return new RpcResult();
  ```

  









## 参考

https://www.cnblogs.com/java-zhao/p/7822766.html