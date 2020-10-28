# HSF



## OSGI





## HSF的基本概念

HSF全称为High-Speed Service Framework，旨在为淘系的应用提供一个分布式的服务框架，HSF从分布式应用层面以及统一的发布/调用方式层面为你们提供支持，从而能够很容易的开发分布式的应用以及提供或使用公用功能模块，而不用考虑分布式领域中的各类细节技术，例如远程通信、性能损耗、调用的透明化、同步/异步调用方式的实现等等问题。





## HSF 实现原理



### 提供服务的流程 

- server启动时候向configserver注册
- client启动时候向configserver请求list
- client缓存list，发现不可用的server，从缓存中remove
- configserver经过心跳包维护可用server的list
- list有更新的时候，configserver经过带version的报文通知client更新

![image-20201028113324805](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w4wrzf5j30ql063n0d.jpg)

从以上几个问题出发，看下HSF的实现方式。





### HSF的总体实现方式

![image-20201028113402089](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w5jvqtnj30qp0dygsl.jpg)

从图中能够看出，HSF的实现方式能够理解为是C/S的架构，可是和传统的C/S架构相比仍是有很大的不一样，HSF没有真正的服务器，每一个应用均可以成为服务的调用方和提供方。具体工做方式如下：

- ConfigServer：远程调用对端的地址就是由ConfigServer 来推送的，这样用户只须要配置本身的服务端或者消费端，不须要对本身的地址进行管理。
- Diamond：持久化的配置中心，用于配置服务调用的规则。
- 服务：服务是调用方和提供方交流的依凭，通常是一个接口，表示一个业务行为以及相关的数据含义。经过使用HSFApiProviderBean可以暴露一个服务，将机器的地址注册到configserver，而且可以经过12200端口进行服务提供，经过HSFApiConsumerBean可以包装出一个客户端，它是服务接口的一个代理，而且它从configserver上订阅了服务的地址列表，可以在这个列表上完成随机调用，作到负载均衡与HA((High Available,高可用性群集)。
- 网络通讯：HSF的底层网络通讯是使用netty框架实现的，是基于epoll的NIO的网络通信框架，HSF在此使用的是长链接，经过合理的服务部署及负债均衡，基本不存在I/O方面的限制。







## HSF设计架构

![image-20201028113539594](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w78nd46j30jo0a8q4x.jpg)

Server端除了configServer外还有一个diamond用来保存一些持久化的配置信息，这里不进行过多的介绍。

Client是HSF的重点，下面是各模块的功能介绍：

- Proxy：这一层主要负责接口的代理。基本上全部的RPC框架都会用到代理模式，相信你们不陌生。须要注意的是HSF的代理层还进行了软负载和单元化的处理。
- Remoting：这一层是HSF的应用层协议，定义了报文格式，各个字段的含义等信息，内容比较多，以后单独写一篇文章来介绍。
- Processer：这一层主要是处理HSF自身的业务逻辑，包括埋点、限流、鉴权等。
- Netty：上面三层会将一次服务调用或者服务返回包装成一个报文，而后经过这层传输。





## HSF调用流程

![image-20201028113637606](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w88nh6hj30jz07qjsz.jpg)

上图是HSF整个的调用过程，从左向右看：

上图是HSF整个的调用过程，从左向右看：

- 第一条线路至关于consumer进行服务调用的过程，首先通过proxy层，将请求通过代理类包装出去；而后是Remoting层进行协议的包装，(处理HSF自身的业务逻辑，包括埋点、限流、鉴权等)最后io层发送出去。
- 第二条线路至关于provider将结果返回后解析的过程，与上一流程恰好相反。
- 右边的provider两条调用流程相信你们都能按照上面的过程理解，就不一一讲解了。







## HSF处理请求流程



### HSF提供端初始化

![image-20201028113744087](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w9ecqf0j30j80egjus.jpg)





### HSF消费端初始化

![image-20201028113804976](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4w9rf3vlj30j00e9dji.jpg)





### 消费方请求到提供方，提供方响应一次调用

![image-20201028113829649](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4wa6wgdaj30km0afdkc.jpg)







## 实现细节



### 心跳检测

一、客户端主动关闭链接：客户端第一次与服务端创建连接后，就会周期性（27s）发送心跳包的callback调用，若是连续三次收不到服务端的心跳包回应，客户端主动关闭连接

二、服务端主动关闭链接：当链接59s没有调用（对方网络不可用，或者full gc过久），至关于两次（2*27s）收不到心跳包的时间







## HSF的优势



### 服务的自动注册、发现

经过注册中心，实现服务的注册/注销与服务的发现。当服务启动后，会调用publish来将服务发布到中心，而服务的消费者，经过调用订阅接口传入的监听器来更新服务提供者列表。HSF提供了三种注册中心实现，分别是ConfigServer，Zookeper，和配置文件模式。

![image-20201028114025386](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4wc73tvfj30sj0bxtf5.jpg)

这里仅对Zookeper进行分析，ZookeeperMetadataAddressService 实现了上述接口，在初始化中实例化了一个ZookeeperRegistry来进行管理，其使用了一个封装了zookeperclient的实例-ZkclientZookeeperClient，在注册服务的时候根据url参数中的Constants.DYNAMIC_KEY来肯定建立Persistent节点仍是Ephemeral节点，Ephemeral节点生命周期与本机链接绑定，这样就能够实现本机离线后的服务自动注销的功能。





### 服务提供者与消费者之间长链接

HSF采用长链接方式进行通讯，相比短链接，长链接更具性能优点，避免链接重复建立与销毁带来的缓冲区申请与释放。

HSF抽象了链接AbstractClient(Client)，并采用了netty框架做为底层实现。netty是一个性能很是优秀的通讯框架，基于Reactor模式，内部采用了管线模式来解耦不一样层次的逻辑之间的耦合问题。

HSF为了强化TCP链接的可用性，增长HeartBeat功能，使用了一个Netty提供的 HashedWheelTimer 的定时任务调度器来执行心跳包的发送(补充：此HashedWheelTimer原理采用轮片式的桶结构，避免每次操做对所有任务的迭代操做，只对将要到期的桶进行操做，此原理也可用于缓存系统设计，在须要进行垃圾回收的状况下只须要按照桶为单位进行内存回收)。





### 非侵入性

HSF最大优势是非侵入性，它使用了JAVA的Proxy机制来实现这一特色，在经过xml配置文件配置Consumer的时候，其实是调用了 HSFApiConsumerBean ，在它的初始化方法中，读取了配置的实现接口，并在ProcessComponent中用一个封装了Proxy注册功能，并实现了InvocationHandler接口的类HSFServiceProxy去管理。使用者在本身的代码中无需作任何特殊处理，就像使用本地方法同样去调用其方法。

![image-20201028114219102](https://tva1.sinaimg.cn/large/0081Kckwgy1gk4we6h78wj30t80een95.jpg)





### 版本管理

这一特性能够很灵活的帮助上线运营的服务在升级过程当中避免服务不可用的状况。



### 服务治理

能够经过网页可视化查看、管理、测试服务的可用。



### 扩展灵活

能够接入自动服务降级功能(熔断) - 根据配置或服务的执行结果，在调用级控制服务是否调用执行，避免服务总体瘫痪，提高服务的可用性。















## 参考

[HSF实战](https://blog.csdn.net/tang_MrTang/article/details/80779199)

[HSF原理](https://www.shangmayuan.com/a/300a16e4a5234d8a9b19a4e0.html)

[初识OSGI](https://blog.csdn.net/acmman/article/details/50848595)