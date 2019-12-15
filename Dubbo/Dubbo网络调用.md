# Dubbo网络调用



## 背景

我们知道Dubbo远程调用（消费过程）的大致流程如下：

- 从Dirctory中获取该方法的invoker列表
- 经过router路由的筛选，得到满足条件的invoker列表
- 经过Cluster容错调用invoker
- 经过loadBalance筛选出最终执行的invoker
- 经过消费端的filter链
- 网络请求及序列化
- .....提供者方执行请求，返回结果
- 用户线程获取结果



这篇文章讲介绍网络请求的几种方式，以及用户线程如果获取到最终的结果。







## 网络调用方式

Dubbo使用Netty作为底层通讯框架，Netty是基于NIO实现的通讯框架，那么Dubbo是如何基于Netty进行网络通讯的呢？

Dubbo为一种RPC通信框架，提供进程间的通信，在使用dubbo协议+Netty作为传输层时，提供三种API调用方式：

1. 同步接口
2. 异步带回调接口
3. 异步不带回调接口



Dubbo里面通过参数isOneway、isAsync来控制调用方式：

- isOneway=true 表示异步不带回调
- isAsync=true 表示异步带回调
- 上述两种情况都不满足，使用同步API







### 同步接口



#### 概念

 同步接口适用在大部分环境，通信方式简单、可靠，客户端发起调用，等待服务端处理，调用结果同步返回。

这种方式下，在高吞吐、高性能（响应时间很快）的服务接口场景中最为适用，可以减少异步带来的额外的消耗，也方便客户端做一致性保证。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9x81lx6nuj31180qe78g.jpg" alt="image-20191215105731790" style="zoom:50%;" />





#### 源码

同步情况下，客户端发起请求，并通过get()方法阻塞等待服务端的响应结果:

```java
RpcContext.getContext().setFuture(null);
return (Result) currentClient.request(inv, timeout).get();

// 代码位置: org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
```



我们进入get()方法中：

```java
public Object get(int timeout) throws RemotingException {
  if (timeout <= 0) {
    timeout = Constants.DEFAULT_TIMEOUT;
  }

  // 如果没有收到响应，则进行循环内，循环进行判断
  if (!isDone()) {

    long start = System.currentTimeMillis();
    lock.lock();
    try {
      while (!isDone()) {
        // 在等待队列中阻塞，等待被唤醒
        done.await(timeout, TimeUnit.MILLISECONDS);
        if (isDone() || System.currentTimeMillis() - start > timeout) {
          break;
        }
      }
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    } finally {
      lock.unlock();
    }

    if (!isDone()) {
      throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
    }
  }

  // 从response对象中获取结果返回
  return returnFromResponse();
}

// 代码位置： org.apache.dubbo.remoting.exchange.support.DefaultFuture#get(int)
```



用户线程什么时候被唤醒呢？

```java
private void doReceived(Response res) {
  lock.lock();
  try {
    response = res;
    if (done != null) {
      // 当收到响应的时候，唤醒正在等待的客户端线程
      done.signal();
    }
  } finally {
    lock.unlock();
  }
  if (callback != null) {
    invokeCallback(callback);
  }
}

// 代码位置：org.apache.dubbo.remoting.exchange.support.DefaultFuture#doReceived
```



#### 流程

- 用户线程发起网络请求
- 用户线程调用ResponseFuture.get()方法进入阻塞状态
- 提供方执行请求，返回结果
- 唤醒用户线程
- 用户线程从await()方法返回，得到结果







### 异步带回调接口



#### 概念

异步带回调接口，用在任务处理时间较长，客户端应用线程不愿阻塞等待，而是为了提高自身处理能力希望服务端处理完成后可以异步通知应用线程。这种方式可以大大提升客户端的吞吐量，避免因为服务端的耗时问题拖死客户端。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9x8hg9imzj31000qs0wp.jpg" alt="image-20191215111248130" style="zoom:50%;" />



#### 实战

首先设置referenceConfig为async参数为异步调用：

```xml
<dubbo:reference
            id="demoService"
            interface="com.huang.yuan.api.service.DemoService"
            version="1.0"
            async="true"   
            timeout="1000000">
</dubbo:reference>
```

提供方代码如下:

```java
@Override
public ModelResult<String> test(String param) {

  spLogger.warn("远程方法被执行了，进入睡眠");

  try {
    Thread.sleep(1000);
  } catch (Exception e) {
    e.printStackTrace();
  }

  spLogger.warn("远程方法结束了");

  return new ModelResult<>(param);
}
```

消费方代码如下：

```java
public void test() throws Exception {
  ModelResult<String> modelResult = demoService.test("huangyuan");
  System.out.println("立即获取到结果= " + modelResult);

  System.out.println("用户线程先做点别的事...");

  Future future = RpcContext.getContext().getFuture();
  System.out.println("用户线程通过future获取到结果= " + future.get());
}
```

结果如下：

![image-20191215113139544](https://tva1.sinaimg.cn/large/006tNbRwgy1g9x912dk5wj31b603ggmq.jpg)



#### 源码

异步请求的情况下，用户线程发起请求后，放置一个Future到RpcContext中，返回立即返回一个空的结果。

```java
ResponseFuture future = currentClient.request(inv, timeout);
RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
return new RpcResult();
// 代码位置: com.alibaba.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
```



用户线程这时可以去干点别的事，当用户线程想要获取结果的时候，可以调用Future.get()方法尝试获取结果，代码会进入 com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#get(int)，如果这时候提供方还没有返回结果，则用户线程进入阻塞状态



同样的，接收到提供方的结果以后，回调com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#doReceived，这时候用户线程就可以从阻塞状态中返回，获取到结果。







#### 流程

引用官网的图解，大概流程如下：

![image-20191215110750389](https://tva1.sinaimg.cn/large/006tNbRwgy1g9x8caa8uaj30xo0aowl9.jpg)

- 1、用户线程调用接口
- 2、发出网络请求
- 3、设置future到RpcContext上下文中
- 4、用户线程从RpcContext上下文中获取future
- 5、用户线程调用get()方法等待结果，进入阻塞状态
- 6、远程调用返回响应
- 7、唤醒用户线程，获取结果

****

**可以看到，Dubbo目前虽然实现了异步调用，但是获取结果还是需要同步阻塞等待，这个问题在apache dubbo中通过CompletableFuture得到解决，用户线程可以真正的不用管结果何时返回，只要dubbo回调用户线程，用户线程去拿结果即可**







### 异步不带回调接口



#### 概念

异步不带回调接口，一些场景为了进一步提升客户端的吞吐能力，只需发起一次服务端调用，不需关心调用结果，可以使用此种通信方式。

一般在不需要严格保证数据一致性或者有其他补偿措施的情况下，选用这种，可以最小化远程调用带来的性能损耗。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9x9gr0n5jj30za0oyad1.jpg" alt="image-20191215114643497" style="zoom:50%;" />





#### 实战

这种调用方式目前Dubbo的配置文件似乎还不支持，可以通过自定义FIlter，改写Dubbo参数的方式使用这种调用方式：

```java
<dubbo:reference
  id="demoService"
  interface="com.huang.yuan.api.service.DemoService"
  version="1.0"
  timeout="1000000"
  filter="testFilter">  // 自定义filter
</dubbo:reference>
```

自定义Filter如下：

```java
public class Filter2 implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        
        Map<String, String> attachments = invocation.getAttachments();

        // 通过RETURN_KEY这个参数，表示调用不需要返回值
        attachments.put(Constants.RETURN_KEY, "false");

        return invoker.invoke(invocation);
    }
}
```

输出结果：

![image-20191215115754195](https://tva1.sinaimg.cn/large/006tNbRwgy1g9x9sdnnibj316603kdgo.jpg)



#### 源码

异步不带回调接口的调用方式，源码非常简单，就是在发起请求之后，立即返回一个空结果

```java
boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
currentClient.send(inv, isSent);
RpcContext.getContext().setFuture(null);
```













## 参考

[非阻塞通讯下的同步API实现原理](https://www.cnblogs.com/yaohonv/p/rpc_sycn_nio.html)

