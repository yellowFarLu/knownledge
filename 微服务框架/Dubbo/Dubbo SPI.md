# SPI扩展



## 前言

站在一个框架作者的角度来说，定义一个接口，自己默认给出几个接口的实现类，同时 **允许框架的使用者也能够自定义接口的实现**。现在一个简单的问题就是：**如何优雅的根据一个接口来获取该接口的所有实现类呢？**

JDK SPI 正是为了优雅解决这个问题而生，**SPI 全称为 (Service Provider Interface)，即服务提供商接口**，是JDK内置的一种服务提供发现机制。目前有不少框架用它来做服务的扩展发现，**简单来说，它就是一种动态替换服务实现者的机制**。

所以，Dubbo如此被广泛接纳的其中的 **一个重要原因就是基于SPI实现的强大灵活的扩展机制**，开发者可自定义插件嵌入Dubbo，实现灵活的业务需求。



## Java SPI

JDK为SPI的实现提供了工具类，即java.util.ServiceLoader，ServiceLoader中定义的SPI规范没有什么特别之处，**只需要有一个提供者配置文件（provider-configuration file），该文件需要在resource目录`META-INF/services`下，文件名就是服务接口的全限定名**。

1. **文件内容是提供者Class的全限定名列表**，显然提供者Class都应该实现服务接口；
2. **文件必须使用UTF-8编码**；



### 示例

```java
public interface Cmand {
    public void execute();
}

public class ShutdownCommand implements Cmand {

    public void execute() {
        System.out.println("shutdown....");
    }

}

public class StartCommand implements Cmand {
    public void execute() {
        System.out.println("start....");
    }
}

public class SPIMain {

    public static void main(String[] args) {

        ServiceLoader<Cmand> loader = ServiceLoader.load(Cmand.class);

        System.out.println(loader);

        for (Cmand Cmand : loader) {
            Cmand.execute();
        }
    }
}
```

![Snip20191027_24](https://tva1.sinaimg.cn/large/006y8mN6gy1g8d3abrd4hj31gn0u00yq.jpg)



### 缺点

- 虽然ServiceLoader也算是使用的延迟加载，**但是只能通过遍历获取，也就是遍历的时候，接口的实现类会全部加载并实例化一遍**。如果你并不想用某些实现类，它也被加载并实例化了，这就造成了浪费。

- 获取某个实现类的方式不够灵活，**只能通过Iterator形式获取**，不能根据某个参数来获取对应的实现类。







## Dubbo SPI

Dubbo对JDK SPI进行了扩展，对服务提供者配置文件中的内容进行了改造，**由原来的提供者类的全限定名列表改成了KV形式的列表，这也导致了Dubbo中无法直接使用JDK ServiceLoader**，所以，与之对应的，在Dubbo中有ExtensionLoader。

**ExtensionLoader是扩展点载入器，用于载入Dubbo中的各种可配置组件**，比如：负载均衡策略（LoadBalance）、拦截器（Filter）、集群方式（Cluster）等。

总之，Dubbo为了应对各种场景，**它的所有内部组件都是通过这种SPI的方式来管理的**，这也是为什么Dubbo需要将服务提供者配置文件设计成KV键值对形式，**这个K就是我们在Dubbo配置文件或注解中用到的K，Dubbo直接通过服务接口（上面提到的ProxyFactory、LoadBalance、Protocol、Filter等）和配置的K从ExtensionLoader拿到服务提供的实现类**。



SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将**接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类**。正因此特性，我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。SPI 机制在第三方框架中也有所应用，比如 Dubbo 就是通过 SPI 机制加载所有的组件。不过，Dubbo 并未使用 Java 原生的 SPI 机制，而是对其进行了增强，使其能够更好的满足需求。在 Dubbo 中，SPI 是一个非常重要的模块。基于 SPI，我们可以很容易的对 Dubbo 进行拓展。如果大家想要学习 Dubbo 的源码，SPI 机制务必弄懂。



### 扩展功能介绍

Dubbo对SPI的扩展是 **通过ExtensionLoader来实现的**。

**Dubbo通过SPI注解定义了可扩展的接口**，如LoadBalance、Filter、Transporter等。**每个类型的扩展对应一个ExtensionLoader。SPI的value参数决定了默认的扩展实现**。



查看ExtensionLoader的源码，可以看到Dubbo对JDK SPI 做了三个方面的扩展：

- **方便获取扩展实现**：JDK SPI仅仅通过接口类名获取所有实现，**而ExtensionLoader则通过接口类名和key值获取一个实现**；

- **IOC依赖注入功能：Adaptive实现**，就是生成一个代理类，**这样就可以根据实际调用时的一些参数动态决定要调用的类了**。

  - 举例来说：接口A，实现者A1、A2。接口B，实现者B1、B2。

    现在实现者A1含有setB()方法，会自动注入一个接口B的实现者，此时注入B1还是B2呢？都不是，**而是注入一个动态生成的接口B的实现者 B$Adpative，该实现者能够根据参数的不同，自动引用B1或者B2来完成相应的功能**；

- **采用装饰器模式进行功能增强，自动包装实现，这种实现的类一般是自动激活的**，常用于包装类，比如：Protocol的两个实现类：ProtocolFilterWrapper、ProtocolListenerWrapper。

  - 还是第2个的例子，**接口A的另一个实现者AWrapper1**。大体内容如下：

    ```java
    private A a;
    AWrapper1（A a）{
       this.a=a;
    }
    ```

    因此，**当在获取某一个接口A的实现者A1的时候，已经自动被AWrapper1包装了**。



### 扩展源码分析

通过ExtensionLoader加载一个实现类的流程，大致如下：

- 获取ExtensionLoader
  - 在Dubbo中每一个扩展接口都对应有一个ExtensionLoader
- 加载所有扩展类的具体实现类
  - dubbo会扫描META-INF/dubbo/internal、META-INF/dubbo、META-INF/services三个目录下的配置文件
  - 然后加载配置文件中的扩展实现类，放到一个map中，键就是SPI的key，值为扩展实现类的class
- 通过SPI的key获取扩展实现类
- 实例化扩展实现类
- 对扩展实现类进行依赖注入
- 如果有装饰类型，则包上一层装饰类型
- 返回扩展类实例

![image-20191028105112990](https://tva1.sinaimg.cn/large/006y8mN6gy1g8dq47izxrj30xc0i9n09.jpg)



















## 参考

[SPI示例及概念](https://www.jianshu.com/p/7daa38fc9711)

[Dubbo SPI原理流程](https://www.cnblogs.com/heart-king/p/5632524.html)