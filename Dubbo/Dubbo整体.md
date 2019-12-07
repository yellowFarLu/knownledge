# Dubbo



## RPC由来

​	在校期间大家都写过不少程序，比如写个hello world服务类，然后本地调用下，如下所示。这些程序的特点是服务消费方和服务提供方是本地调用关系。

　　而一旦踏入公司，尤其是大型互联网公司就会发现，公司的系统都由成千上万大大小小的服务组成，各服务部署在不同的机器上，由不同的团队负责。这时就会遇到两个问题：

　　(1) 要搭建一个新服务，免不了需要依赖他人的服务，而现在他人的服务都在远端，怎么调用？

　　(2) 其它团队要使用我们的服务，我们的服务该怎么发布以便他人调用？　　

​	

​	简要流程就是：

​		- 服务消费方把调用信息转成二进制，并且传给服务提供方，进行远程调用。

​		- 服务提供方执行函数，获得结果，再把结果序列化成二进制，并且传输给服务消费方

![11977583-e76bbbe6ecc1df4a](https://ws2.sinaimg.cn/large/006tNbRwgy1fxz8p7wwxhj31do0tzq5l.jpg)

　　由于各服务部署在不同的机器上，服务间的调用免不了网络通信过程，服务消费方每调用一个服务都要写一坨网络通信相关的代码（比如说图中Client Stub、Server Stub），不仅复杂而且极易出错。

​	如果有一种方式 **能让我们像调用本地服务一样调用远程服务**，而让调用者对网络通信这些细节透明，那么将大大提高生产力，比如服务消费方在执行helloWorldService.sayHello(“test”)时，实质上调用的是远端的服务。

　　这种方式其实就是**RPC**(Remote Procedure Call Protocol)，在各大互联网公司中被广泛使用，如阿里巴巴的hsf、Dubbo(开源)、Facebook的thrift(开源)、Google grpc(开源)等。





## 基础概念

Dubbo是alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合），比如表现层和业务层就需要解耦合。

从面向服务的角度来看，Dubbo采用的是一种非常简单的模型，要么是提供方提供服务，要么是消费方消费服务，所以基于这一点可以抽象出服务提供方（Provider）和服务消费方（Consumer）两个角色。

除了以上两个角色，它还有注册中心和监控中心。它可以通过注册中心对服务进行注册和订阅；可以通过监控中心对服务进行监控，这样的话，就可以知道哪些服务使用率高、哪些服务使用率低。对使用率高的服务增加机器，对使用率低的服务减少机器，达到合理分配资源的目的。

Dubbo就是SOA服务治理方案的核心框架。用于分布式调用，其重点在于分布式的治理。



## 整体架构

![image-20191207161601067](https://tva1.sinaimg.cn/large/006tNbRwgy1g9o8aj012yj30pm0gitfo.jpg)

### 节点角色说明

| 节点        | 角色说明                               |
| ----------- | -------------------------------------- |
| `Provider`  | 暴露服务的服务提供方                   |
| `Consumer`  | 调用远程服务的服务消费方               |
| `Registry`  | 服务注册与发现的注册中心               |
| `Monitor`   | 统计服务的调用次数和调用时间的监控中心 |
| `Container` | 服务运行容器                           |



### 调用关系说明

1. 服务容器负责启动，加载，运行服务提供者。
2. 服务提供者在启动时，向注册中心注册自己提供的服务。
3. 服务消费者在启动时，向注册中心订阅自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6. 服务消费者和提供者，**在内存中累计调用次数和调用时间**，定时每分钟发送一次统计数据到监控中心。





### 特性

Dubbo 架构具有以下几个特点，分别是连通性、健壮性、伸缩性、以及向未来架构的升级性。

#### 连通性

- 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
- 监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示
- 服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销
- 服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销
- 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
- 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者
- 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
- 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者



#### 健壮性

- 监控中心宕掉不影响使用，只是丢失部分采样数据
- 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
- 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
- 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
- 服务提供者无状态，任意一台宕掉后，不影响使用
- 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复



#### 伸缩性

- 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
- 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者



#### 升级性

当服务集群规模进一步扩大，带动IT治理结构进一步升级，需要实现动态部署，进行流动计算，现有分布式服务架构不会带来阻力。下图是未来可能的一种架构：

![image-20191207161935836](https://tva1.sinaimg.cn/large/006tNbRwgy1g9o8e7izcyj30xk0m6awv.jpg)

##### 节点角色说明

| 节点         | 角色说明                               |
| ------------ | -------------------------------------- |
| `Deployer`   | 自动部署服务的本地代理                 |
| `Repository` | 仓库用于存储服务应用发布包             |
| `Scheduler`  | 调度中心基于访问压力自动增减服务提供者 |
| `Admin`      | 统一管理控制台                         |
| `Registry`   | 服务注册与发现的注册中心               |
| `Monitor`    | 统计服务的调用次数和调用时间的监控中心 |







## Dubbo核心功能

远程通讯：

​	远程通讯，提供对多种NIO框架抽象封装，包括“同步转异步”和“请求-响应”模式的信息交换方式。



集群容错：

​	服务框架，提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。



服务注册：

​	服务注册，基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。





## 框架设计

### 整体设计

![image-20191207162549759](https://tva1.sinaimg.cn/large/006tNbRwgy1g9o8kp2mxzj315l0u0u0x.jpg)

图例说明：

- 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。
- 图中从下至上分为**十层**，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。
- 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
- 图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。



### 各层说明

- **service层**：消费者使用的接口，在代码里，我们直接使用这个
- **config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
- **proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`
- **registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`
- **cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`
- **monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`
- **protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`
- **exchange 信息交换层**：**封装请求响应模式，同步转异步**，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`
- **transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`
- **serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`



### 关系说明

- 在 RPC 中，Protocol 是核心层，也就是只要有 Protocol + Invoker + Exporter 就可以完成非透明的 RPC 调用，然后在 Invoker 的主过程上 Filter 拦截点。
- 图中的 Consumer 和 Provider 是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用 Client 和 Server 的原因是 Dubbo 在很多场景下都使用 Provider, Consumer, Registry, Monitor 划分逻辑拓普节点，保持统一概念。
- 而 Cluster 是外围概念，所以 Cluster 的目的是将多个 Invoker 伪装成一个 Invoker，这样其它人只要关注 Protocol 层 Invoker 即可，加上 Cluster 或者去掉 Cluster 对其它层都不会造成影响，因为只有一个提供者时，是不需要 Cluster 的。
- Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
- 而 Remoting 实现是 Dubbo 协议的实现，如果你选择 RMI 协议，整个 Remoting 都不会用上，Remoting 内部再划为 Transport 传输层和 Exchange 信息交换层，Transport 层只负责单向消息传输，是对 Mina, Netty, Grizzly 的抽象，它也可以扩展 UDP 传输，而 Exchange 层是在传输层之上封装了 Request-Response 语义。
- Registry 和 Monitor 实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。





### 领域模型

在 Dubbo 的核心领域模型中：

- Protocol 是服务域，它是 Invoker 暴露和引用的主功能入口，它负责 Invoker 的生命周期管理。
- Invoker 是实体域，**它是 Dubbo 的核心模型**，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
- Invocation 是会话域，**它持有调用过程中的变量**，比如方法名，参数等。



### 基本设计原则

- 采用 Microkernel + Plugin 模式，Microkernel 只负责组装 Plugin，**Dubbo 自身的功能也是通过扩展点实现的，也就是 Dubbo 的所有功能点都可被用户自定义扩展所替换。**
- **采用 URL 作为配置信息的统一格式**，所有扩展点都通过传递 URL 携带配置信息。











## 名词补充解释



### Invoker

**概念**
Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。它里面有一个很重要的方法 Result invoke(Invocation invocation)。

**Invocation**是会话域，它持有调用过程中的变量，比如方法名，参数等重要信息。



#### 2种类型的Invoker

##### 本地执行类的Invoker

本地执行方法，比如说单元测试执行本项目的方法。举个例子，比如有一个Dubbo接口demoService.sayHello，在本项目中执行 demoService.sayHello，就通过InjvmExporter来进行反射执行demoService.sayHello就可以了。



##### 远程通信类的Invoker

client端：要执行 demoService.sayHello，它封装了DubboInvoker进行远程通信，发送要执行的接口给server端。
server端：采用了AbstractProxyInvoker执行了DemoServiceImpl.sayHello,然后将执行结果返回发送给client.



**按服务提供、服务消费分类**

引用官方文档：分为服务提供 Invoker 和服务消费 Invoker 
![4181053155-5be6d41891122_articlex](https://ws1.sinaimg.cn/large/006tNbRwgy1fxz867tz01j30h20b6t9o.jpg)

为了更好的解释上面这张图，我们结合服务消费和提供者的代码示例来进行说明：

**服务消费者代码**

```java
public class DemoClientAction {
 
    private DemoService demoService;
 
    public void setDemoService(DemoService demoService) {
        this.demoService = demoService;
    }
 
    public void start() {
        String hello = demoService.sayHello("world" + i);
    }
}
```

上面代码中的 DemoService 就是上图中服务消费端的 proxy，用户代码通过这个 proxy 调用其对应的 Invoker，而该 Invoker 实现了真正的远程服务调用。



**服务提供者代码**

```java
public class DemoServiceImpl implements DemoService {
 
    public String sayHello(String name) throws RemoteException {
        return "Hello " + name;
    }
}
```

上面这个类会被封装成为一个 AbstractProxyInvoker 实例，并新生成一个 Exporter 实例。这样当网络通讯层收到一个请求后，会找到对应的 Exporter 实例，并调用它所对应的 AbstractProxyInvoker 实例，从而真正调用了服务提供者的代码。



**Invoker继承关系**

![2909755045-5be6d4186fdda_articlex](https://ws4.sinaimg.cn/large/006tNbRwgy1fxz7yh7yvhj307l097749.jpg)



### Directory

**概念**
简单来说，Directory就是装载invoker的文件目录



**两个重要Directory**

#### StaticDirectory

​	静态目录服务，他的Invoker是固定的。



#### RegistryDirectory

​	注册目录服务，他的Invoker集合数据来源于zk注册中心的，他实现了NotifyListener接口，这个接口中的notify方法就是注册中心的回调，也就是它之所以能根据注册中心动态变化的根源所在.。
整个过程有一个重要的map变量，methodInvokerMap（它是数据的来源；同时也是notify的重要操作对象，重点是写操作。）



### Router

**概念**
利用Router，可以从多个服务提者方中选择一个进行调用
**分类**
主要是3个实现类：
ConditionRouter（条件路由）：条件路由主要就是根据Dubbo管理控制台配置的路由规则来过滤相关的invoker
MockInvokersSelector：主要根据参数，判断是否需要筛选出正常的（非mock的）invoker 或者 mock的invoker
ScriptRouter（脚本路由）

**例子**
参考：org.apache.Dubbo.rpc.cluster.router.script.ScriptRouterTest



### LoadBalance

**概念**
负载均衡策略的实现。与Router功能类似，利用负载均衡策略（random,roundrobin,leastactive），从多个服务提者方中选择一个进行调用





### ProxyFactory

在服务提供者端，ProxyFactory主要把服务的真正实现统一包装成一个Invoker，Invoker通过反射来执行具体的Service实现对象的方法。默认的实现是JavassistProxyFactory，代码如下：

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper类不能正确处理带$的类名
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                  Class<?>[] parameterTypes, 
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```



### Protocol

负责服务的发布和订阅。

Protocol是Dubbo中的服务域，只在服务启用时加载，无状态，线程安全，是实体域Invoker暴露和引用的主功能入口，负责Invoker的生命周期管理，是Dubbo中远程服务调用层。

Protocol根据指定协议对外公布服务，当客户端根据协议调用这个服务时，Protocol会将客户端传递过来的Invocation参数交给Invoker去执行。

Protocol加入了远程通信协议，会根据客户端的请求来获取参数Invocation。

```java
@Extension("dubbo")
public interface Protocol {

    int getDefaultPort();

    // 对于服务提供端，将本地执行类的Invoker通过协议暴漏给外部
    // 外部可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    // 这个是针对服务消费端的，服务消费者从注册中心获取服务提供者发布的服务信息
    // 通过服务信息得知服务提供者使用的协议，然后服务消费者仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
    // 执行时，需要将执行信息通过指定协议发送给服务提供者，服务提供者接收到参数Invocation，然后交给服务提供者的本地Invoker来执行
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```



#### Exporter

负责invoker的生命周期，包含一个Invoker对象，可以撤销服务。



#### Exchanger

负责数据交换和网络通信的组件。每个Invoker都维护了一个ExchangeClient的 引用，并通过它和远端server进行通信。



## 扩展

### Dubbo的接口定义

dubbo传参数：

- 参数类必须实现Serializable序列化接口
- 参数的属性也必须能够序列化
- 属性不能是Class类型