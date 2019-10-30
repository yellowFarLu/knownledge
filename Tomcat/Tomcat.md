# Tomcat

Tomcat是开源的 Java Web 应用服务器，实现了 Java EE规范，如果Servlet、JSP、webSocket。



## Tomcat组成

![image-20191006124211822](https://tva1.sinaimg.cn/large/006y8mN6gy1g7odoxsz68j30yz0u048m.jpg)

**注意**: *Server*是整个Tomcat组件的容器，包含一个或多个Service。 *Service*：Service是包含Connector和Container的集合，Service用适当的Connector接收用户的请求，再发给相应的Container来处理。



**Server**：指的就是整个**Tomcat 服务器**，负责管理和 启动各个 Service（服务），同时监听 8005 端口发过来的 shutdown 命令，用 于关闭整个容器

**Service**：即**web服务**， 包含Connectors、Container 两个 核心组件，以及多个功能组件，各个Service 之间是独立的，但是共享同一JVM 的资源

**Connector**：**Tomcat 与外部世界的连接器**。负责监听固定端口，接收外部请求，传递给 Container，并 将 Container 处理的结果返回给外部

**Container**：即**Servlet 容器（Catalina）**，内部有多层容器组成，用于管理 Servlet 生命周期，调用 servlet 相关方法。

**Loader**：封装了Java ClassLoader，**用于Container加载类文件**

**Realm**：Tomcat 中为 web 应用程序提供访问认证和角色管理的机制

**JMX**：Java SE 中定义技术规范，是一个为应用程序、设备、系统等植入管理功能的框架，**通过 JMX 可以远程监控 Tomcat 的运行状态**。相当于tomcat的管理后台

**Jasper**：Tomcat 的 Jsp 解析引擎，**用于将 Jsp 转换成 Java 文件，并编译成 class 文件**

**Session**：负责管理和创建 session，以及 Session 的持久化(可自定义)，支持 session 的集群

**Pipeline**：在容器中充当管道的作用，管道中可以设置各种 valve(阀门)，请求和响应在经由管道中各个阀门处理，提供了一种灵活可配置的处理请求和响应的机制。相当于tomcat提供的扩展点。使用责任链模式。

**Naming**：命名服务。**命名服务将名称和对象联系起来，使得我们可以用名称访问对象**。目录服务也是一种命名 服务，对象不但有名称，还有属性。Tomcat 中可以使用 JNDI 定义数据源、配置信息，用于开发与部署的分离。（JNDI， Java命名和目录接口，是一组在 Java 应用中访问命名和目录服务的 API）

**Manager**：是被Container用来管理Session池。



## Container组成

![image-20191006125636422](https://tva1.sinaimg.cn/large/006y8mN6gy1g7oe3vtcv9j310a0t0q7c.jpg)

**Engine**：Servlet的顶层容器，包含一个或多个Host子容器。Engine包含Host和Context，接到请求后仍给相应的Host在相应的Context里处理。

**Host**：虚拟主机，负责 web 应用的部署和 Context 的创建

**Context**：Web应用上下文，包含多个 Wrapper，负责web配置的解析、管理所有的 Web 资源

**Wrapper**：最底层的容器，是对 Servlet 的封装，负责 Servlet 实例的创建、执行和销毁





##生命周期管理

Tomcat 为了方便管理组件和容器的生命周期，定义了从创建、启动、到停止、销毁共12种状态，tomcat 生命周期管理了内部状态变化的规则控制，组件和容器只需实现相应的生命周期方法即可完成各生命周期内的操作。

比如执行初始化操作时，会判断当前状态是否 New，如果不是则抛出生命周期异常；是的 话则设置当前状态为 Initializing，并执行 initInternal 方法，由子类实现，方法执行成功则设置当前状态为 Initialized，执行失败则设置为 Failed 状态

![image-20191006130016213](https://tva1.sinaimg.cn/large/006y8mN6gy1g7oe7p8aroj31080h0grd.jpg)

**Tomcat 的生命周期管理引入了事件机制**，在容器的生命周期状态发生变化时会通知事件监听器，监听器通过判断事件的类型来进行相应的操作。事件监听器的添加可以在 server.xml 文件中进行配置

Tomcat 各类容器的配置过程就是通过添加 listener 的方式来进行的，从而达到配置逻辑与容器的解耦。如下， 

**EngineConfig**：主要启动和停止打印日志
**HostConfig**：主要处理部署应用，解析应用 META-INF/context.xml 并创建应用的 Context 

**ContextConfig**：主要解析并合并web.xml，扫描应用的各类 web 资源 (filter、servlet、listener)



###Tomcat 的启动过程

![image-20191006130835860](https://tva1.sinaimg.cn/large/006y8mN6gy1g7oegcyjvgj30zk0rcagq.jpg)

启动从 Tomcat 提供的 start.sh 脚本开始，shell 脚本会调用 Bootstrap 的 main 方法，实际调用了 Catalina 相应的 load、start 方法。

load方法会通过 Digester 进行 config/server.xml 的解析，在解析的过程中会根据xml中的关系和配置信息来创建容器，并设置相关的属性。接着 Catalina 会调用 StandardServer 的 init 和 start 方法进行容器的初始化和启动。

按照 xml 的配置关系，server 的子元素是 service，service 的子元素是顶层容器 Engine，每层容器有持有自己的子容器，而这些元素都实现了生命周期管理 的各方法，因此就很容易的完成整个容器的启动、关闭等生命周期的管理。

StandardServer 完成 init 和 start 方法调用后，会一直监听来自8005端口(可配置)，如果接收到shutdown命令，则会退出循环监听，执行后续的 stop 和 destroy 方法，完成Tomcat 容器的关闭。同时也会调用 JVM 的 Runtime.getRuntime(﴿.addShutdownHook 方法，在虚拟机意外退出的时候来关闭容器。

所有容器都是继承自 ContainerBase，基类中封装了容器中的重复工作，负责启动容器相关的组件Loader、Logger、Manager、Cluster、Pipeline，启动子容器(线程池并发启动子容器，通过线程池submit 多个线程，调用后返回 Future 对象，线程内部启动子容器，接着调用 Future 对象 的 get 方法来等待执行结果)。





###Servlet 生命周期

![image-20191008170339243](https://tva1.sinaimg.cn/large/006y8mN6gy1g7qwhlmwt1j30hy0eutb8.jpg)

Servlet 生命周期可被定义为从创建直到毁灭的整个过程。以下是 Servlet 遵循的过程：

1.被创建：执行init方法，只执行一次

　　1.1Servlet什么时候被创建？

　　--默认情况下，第一次被访问时，Servlet被创建，然后执行init方法；

　　--可以配置执行Servlet的创建时机；

2.提供服务：执行service方法，执行多次

3.被销毁：当服务器正常关闭时，执行destroy方法，只执行一次





**load on startup**

当值为 0 或者大于 0 时，表示容器在应用启动时就加载这个 servlet; 当是一个负数时或者没有指定时，则指示容器在该 servlet 被选择时才加载; 正数的值越小，启动该 servlet 的优先级越高;



**single thread model**

每次访问 servlet，新建 servlet 实体对象，但并不能保证线程安全，同时 tomcat 会限制 servlet 的实例数目
最佳实践：不要使用该模型，servlet 中不要有全局变量。







## 扩展



###Servlet和Spring

spring-mvc的前置控制器DispatcherServlet实际上就是一个servler，前置控制器能够拦截请求，将其分发给Controller处理



### 参考

https://www.jianshu.com/p/d74eef07487f

https://www.jianshu.com/c/37a659fb59ca

https://juejin.im/post/58eb5fdda0bb9f00692a78fc