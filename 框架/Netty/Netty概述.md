# Netty



## 概念

Netty是一个高性能、异步事件驱动的NIO框架，它提供了对TCP、UDP和文件传输的支持，作为一个异步NIO框架，Netty的所有IO操作都是**异步非阻塞**的，通过Future-Listener机制，用户可以方便的主动获取或者通过通知机制获得IO操作结果。

作为当前最流行的NIO框架，Netty在互联网领域、大数据分布式计算领域、游戏行业、通信行业等获得了广泛的应用，一些业界著名的开源组件也基于Netty的NIO框架构建。如：Dubbo、 RocketMQ、Hadoop的Avro、Spark等。

Netty需要学习的内容: 编解码器、TCP粘包/拆包及Netty如何解决、ByteBuf、Channel和Unsafe、ChannelPipeline和ChannelHandler、EventLoop和EventLoopGroup、Future等。

Netty使用的IO模型是异步IO（AIO）。





## Netty逻辑架构

![image-20200504200157948](https://tva1.sinaimg.cn/large/007S8ZIlgy1gego5jbqkrj312s0q2quh.jpg)

### Reactor通信调度层

它由一系列辅助类组成，包括Reactor线程NioEventLoop、NioSocketChannel、NioServerSocketChannel等。

该层的职责就是监听网络的连接事件、读写事件，负责将网络层的数据读取到内存缓冲区中，然后触发各种网络事件，例如连接创建、连接激活、读事件、写事件，将这些事件触发到PipeLine中，由PipeLine管理的责任链来进行后续的处理。



### 责任链 ChannelPipeLine

它负责事件在责任链中的有序传播，同时负责动态的编排责任链。

责任链可以选择监听和处理自己关心的事件。它可以拦截处理和向后传播事件。

通常情况下，往往会开发编解码Handler用于消息的编解码，它可以将外部的消息转换成内部的POJO对象，这样上层业务只需要关心处理业务逻辑即可。



### 业务逻辑编排层

业务逻辑编排层通常有两类，一类是纯粹的业务逻辑编排，一类是应用层协议插件，用于特定协议相关的会话和链路管理。

由于应用层协议栈往往是开发一次到处运行，并且变动较小，故而将应用协议到 POJO 的转变和上层业务放到不同的 ChannelHandler 中，就可以实现协议层和业务逻辑层的隔离，实现架构层面的分层隔离。









## 编解码器

Java序列化的目的主要有两个：

- 网络传输
- 对象持久化

Java序列化仅仅是Java编解码技术的一种，由于他的种种缺陷，衍生出多种编解码技术和框架。



Java序列化的缺点：

- 无法跨语言
- 序列化后的码流太大
- 序列化性能太低





### 业界主流的编解码框架

- Google Protobuf：支持Java、C++、Python三种语言，高效的编码性能，结构化数据存储格式（XML,JSON等）
- Facebook Thrift：适用于静态的数据交换，需要先确定好它的数据结构。结构变化后需要重新编译IDL文件，这也是Thrift的弱项。
- JBoss Marshalling：是一个Java对象的序列化API，修正了JDK自带的序列化包的很多问题，但又保持跟java.io.Serializable接口的兼容。
- Hessian：一个轻量级的remoting onhttp工具，使用简单的方法提供了RMI的功能，采用的是二进制RPC协议。





## TCP粘包/拆包

TCP是个“流”协议，所谓流，就是没有界限的一串数据。TCP底层并不了解上层业务数据的具体含义，它会根据TCP缓存区的实际情况进行包的划分，所以在业务上的一个完整的包，可能被TCP拆分成多个包进行发送，也可能把多个小包封装成一个大的数据包发送，这就是TCP粘包和拆包问题。

有如下几种情况：

- 正常情况，一个包一个包的发送
  - ![image-20200413215512969](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshewhqrvj30zo092q5z.jpg)
- 把小包合并成大包发送
  - ![image-20200413215538863](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshfb3713j310409wn03.jpg)
- 发送拆包和粘包
  - ![image-20200413215655254](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshgmwwcfj30z00f2n45.jpg)





## Netty线程模型

在JAVA NIO方面Selector给Reactor模式提供了基础，Netty结合Selector和Reactor模式设计了高效的线程模型。

关于Java NIO 构造Reator模式，Doug Lea在《Scalable IO in Java》中给了很好的阐述，这里截取PPT对Reator模式的实现进行说明。



### Reactor单线程模型

![image-20200413215907111](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshixdz64j30xu0hgk50.jpg)

这是最简单的Reactor单线程模型，由于Reactor模式使用的是异步非阻塞IO，所有的IO操作都不会被阻塞，理论上一个线程可以独立处理所有的IO操作。这时Reactor线程是个多面手，负责**多路分离套接字，Accept新连接，并分发请求到处理链中**。



对于一些小容量应用场景，可以使用到单线程模型。但对于高负载，大并发的应用却不合适，主要原因如下：

- **当一个NIO线程同时处理成百上千的链路，性能上无法支撑**，即使NIO线程的CPU负荷达到100%，也无法完全处理消息。
- 当NIO线程负载过重后，处理速度会变慢，会导致大量客户端连接超时，超时之后往往会重发，更加重了NIO线程的负载。
- **可靠性低，一个线程意外死循环，会导致整个通信系统不可用。**

为了解决这些问题，出现了Reactor多线程模型。







### Reactor多线程模型

![image-20200413220330128](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshnh868vj30z80rktr6.jpg)

相比上一种模式，该模型在**处理链部分采用了多线程（线程池）**。

在绝大多数场景下，该模型都能满足性能需求。但是，在一些特殊的应用场景下，如服务器会对客户端的握手消息进行安全认证。这类场景下，**单独的一个Acceptor线程可能会存在性能不足的问题**。为了解决这些问题，产生了第三种Reactor线程模型。





### **Reactor主从模型** 

![image-20200413220709660](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshratu5gj31060sc7on.jpg)

该模型相比第二种模型，是将Reactor分成两部分：

- **mainReactor负责监听server socket，accept新连接；并将建立的socket分派给subReactor。**
- subReactor**负责多路分离已连接的socket，读写网络数据**，对业务处理功能，其扔给worker线程池完成。通常，subReactor个数上可与CPU个数等同。

利用主从NIO线程模型，可以解决一个服务端监听线程无法有效处理所有客户端连接的性能不足问题，因此，在Netty的官方Demo中，推荐使用该线程模型。





### Netty模型

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gdsi6k449jj30zg0e0n5g.jpg" alt="image-20200413222150123" style="zoom:200%;" />

其实Netty的线程模型是Reactor模型的变种，那就是去掉线程池的第三种形式的变种，这也是Netty NIO的默认模式。

Netty的线程模型并不是一成不变，它实际取决于用户的启动参数配置。通过设置不同的启动参数，Netty可以同时支持Reactor单线程模型、多线程模型和主从模型。

前面介绍完 Netty 相关一些理论，下面从功能特性、模块组件来介绍 Netty 的架构设计。

















## Netty功能特性

![image-20200413221022868](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshumvq6uj31200negud.jpg)









## Netty模块组件

Netty主要有下面一些组件：

Selector、NioEventLoop、NioEventLoopGroup、ChannelHandler、ChannelHandlerContext、ChannelPipeline



### Selector

Netty 基于 Selector 对象实现 I/O 多路复用，通过 Selector 一个线程可以监听多个连接的 Channel 事件。



### NioEventLoop及NioEventLoopGroup

当系统在运行过程中，如果频繁的进行线程上下文切换，会带来额外的性能损耗。

多线程并发执行某个业务流程，业务开发者还需要时刻对线程安全保持警惕，哪些数据可能会被并发修改，如何保护？这不仅降低了开发效率，也会带来额外的性能损耗。

为了解决上述问题，Netty采用了串行化设计理念

从消息的读取、编码以及后续Handler的执行，始终都由IO线程EventLoop负责，这就意味着整个流程不会进行线程上下文的切换，数据也不会面临被并发修改的风险。这也解释了为什么Netty线程模型去掉了Reactor主从模型中线程池。

EventLoopGroup是一组EventLoop的抽象，EventLoopGroup提供next接口，可以从一组EventLoop里面按照一定规则获取其中一个EventLoop来处理任务

对于EventLoopGroup这里需要了解的是在Netty中，在Netty服务器编程中我们需要BossEventLoopGroup和WorkerEventLoopGroup两个EventLoopGroup来进行工作。

通常一个服务端口即一个ServerSocketChannel对应一个Selector和一个EventLoop线程，也就是说BossEventLoopGroup的线程数参数为1。

BossEventLoop负责接收客户端的连接并将SocketChannel交给WorkerEventLoopGroup来进行IO处理。

EventLoop的实现充当Reactor模式中的分发（Dispatcher）的角色。





### ChannelPipeline

ChannelPipeline其实是担任着Reactor模式中的请求处理器这个角色。

ChannelPipeline的默认实现是DefaultChannelPipeline，DefaultChannelPipeline本身维护着一个用户不可见的tail和head的ChannelHandler，他们分别位于链表队列的头部和尾部。tail在更上层的部分，而head在靠近网络层的方向。

在Netty中关于ChannelHandler有两个重要的接口，ChannelInBoundHandler和ChannelOutBoundHandler。

inbound可以理解为网络数据从外部流向系统内部，而outbound可以理解为网络数据从系统内部流向系统外部。

用户实现的ChannelHandler可以根据需要实现其中一个或多个接口，将其放入Pipeline中的链表队列中，ChannelPipeline会根据不同的IO事件类型来找到相应的Handler来处理。

同时链表队列是责任链模式的一种变种，自上而下或自下而上所有满足事件关联的Handler都会对事件进行处理。

ChannelInBoundHandler对从客户端发往服务器的报文进行处理，一般用来执行半包/粘包，解码，读取数据，业务处理等；

ChannelOutBoundHandler对从服务器发往客户端的报文进行处理，一般用来进行编码，发送报文到客户端。

下图是对ChannelPipeline执行过程的说明：

![image-20200413222646432](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdsibp3b7rj31nm0g4doy.jpg)







### ChannelHandler

是一个接口，处理 I/O 事件或拦截 I/O 操作，并将其转发到其 ChannelPipeline(业务处理链)中的下一个处理程序。



### ChannelHandlerContext

保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象。

ChannelPipeline 是保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作。实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互。

ChannelPipeline对事件流的拦截和处理流程：

![image-20200413221402551](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshyg2okkj30u00y5kfm.jpg)

作者：CleverApe
链接：https://www.jianshu.com/p/a618adef427c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

Netty中的事件分为Inbond事件和Outbound事件。

Inbound事件通常由I/O线程触发，如TCP链路建立事件、链路关闭事件、读事件、异常通知事件等。

Outbound事件通常是用户主动发起的网络I/O操作，如用户发起的连接操作、绑定操作、消息发送等。



在 Netty中，Channel 、ChannelHandler、ChannelHandlerContext、 ChannelPipeline的关系如下图：

![image-20200413221438105](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdshz2fz9qj31ug0kah1d.jpg)

一个 Channel 包含了一个 ChannelPipeline，而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表，并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler。

入站事件和出站事件在一个双向链表中，入站事件会从链表 head 往后传递到最后一个入站的 handler，出站事件会从链表 tail 往前传递到最前一个出站的 handler，两种类型的 handler 互不干扰。





### Buffer

Netty提供的经过扩展的Buffer相对NIO中的有个许多优势，作为数据存取非常重要的一块，我们来看看Netty中的Buffer有什么特点。



#### ByteBuf读写指针

1. 在ByteBuffer中，读写指针都是position，而在ByteBuf中，读写指针分别为readerIndex和writerIndex
2. 直观看上去ByteBuffer仅用了一个指针就实现了两个指针的功能，节省了变量，但是当对于ByteBuffer的读写状态切换的时候必须要调用flip方法，而当下一次写之前，必须要将Buffe中的内容读完，再调用clear方法。
3. 每次读之前调用flip，写之前调用clear，这样无疑给开发带来了繁琐的步骤，而且内容没有读完是不能写的，这样非常不灵活。
4. 相比之下我们看看ByteBuf，读的时候仅仅依赖readerIndex指针，写的时候仅仅依赖writerIndex指针，不需每次读写之前调用对应的方法，而且没有必须一次读完的限制。



#### 零拷贝

![image-20200413222904343](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdsie3d741j30za0ncn2r.jpg)

1. Netty的接收和发送ByteBuffer采用direct buffer，**使用直接内存进行Socket读写**，不需要进行字节缓冲区的二次拷贝。
2. 如果使用传统的堆内存（HEAP BUFFERS）进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
3. Netty提供了组合Buffer对象，可以聚合多个ByteBuffer对象，用户可以像操作一个Buffer那样方便的对组合Buffer进行操作，避免了传统通过内存拷贝的方式将几个小Buffer合并成一个大的Buffer。
4. **Netty的文件传输采用了transferTo方法**，它可以直接将文件缓冲区的数据发送到目标Channel，避免了传统通过循环write方式导致的内存拷贝问题。
   1. 本来数据是从内核——》用户空间——》socket缓冲区，使用transferTo之后，数据直接从内核——》socket缓冲区。从而提高性能。





### 引用计数与池化技术

![image-20200413223619330](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdsiln08cjj30z80lmah7.jpg)

1. 在Netty中，每个被申请的Buffer对于Netty来说都可能是很宝贵的资源，因此为了获得对于内存的申请与回收更多的控制权，Netty自己根据引用计数法去实现了内存的管理。
2. Netty对于Buffer的使用都是基于直接内存（DirectBuffer）实现的，大大提高I/O操作的效率
3. 然而DirectBuffer和HeapBuffer相比之下除了I/O操作效率高之外还有一个天生的缺点，即对于DirectBuffer的申请相比HeapBuffer效率更低（**为什么？？**）
4. 因此Netty结合引用计数实现了PolledBuffer，即池化的用法，当引用计数等于0的时候，Netty将Buffer回收致池中，在下一次申请Buffer的没某个时刻会被复用。





## 总结

Netty其实本质上就是Reactor模式的实现，Selector作为多路复用器，EventLoop作为转发器，Pipeline作为事件处理器。

但是和一般的Reactor不同的是，Netty使用串行化实现，并在Pipeline中使用了责任链模式。

Netty中的buffer相对有NIO中的buffer又做了一些优化，大大提高了性能。





## 问题



**netty如何避免的NIO空循环?**

当应用程序发起IO请求的时候，应用程序立即返回，有内核等待IO请求，并且得到数据以后，将数据写入应用程序的缓冲区，然后再通知应用程序进行读取，从而避免NIO空循环。















## 参考

[一文看懂netty原理](https://zhuanlan.zhihu.com/p/70970558)

[Netty原理](https://www.jianshu.com/p/a618adef427c)

