### 背景

注入普通本地服务JavaBean是注入该bean实例本身，那么，注入远程的Dubbo服务Bean呢？

其实就是一个proxy引用，该proxy引用由动态代理生成。



### 解析

首先看dubbo中引用bean的配置

<dubbo:reference id="configRemoteService"

interface="combiz.ConfigRemoteService"

version="1.0" />



它使用dubbo:reference对应org.apache.dubbo.config.spring.ReferenceBean类。在Ioc容器初始化的时候，把这个配置对应的BeanDefinition注册到Ioc容器中。该类实现了org.springframework.beans.factory.FactoryBean接口，在依赖注入的时候，调用FactoryBean的getObject()方法获取proxy实例。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean



在Spring容器中，会利用BeanDefinition对象信息来初始化创建之后的Bean对象，但如果你想插手“创建对象”这一步，有两种方法：

- 使用工厂方法模式来指定某个工厂方法来创建Bean

- Bean的类实现FactoryBean接口



Dubbo中就是用第二种方法来插手Bean对象创建这一步，ReferenceBean实现了FactoryBean接口，而getObject()正是FactoryBean接口中定义的

ReferenceBean类继承了com.alibaba.dubbo.config.ReferenceConfig，getObject()的具体实现如下：

```java
@Override
    public Object getObject() throws Exception {
        return get();
    }
```

仅仅是简单调用父类的get()方法，接下来继续看get()方法的实现

```java
public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            // 初始化工作
            init();
        }
        // 将代理工厂创建的代理对象返回
        return ref;
    }
```

init方法中，会通过com.alibaba.dubbo.rpc.proxy.AbstractProxyFactory的getProxy方法生成代理对象。



AbstractProxyFactory#getProxy的通用实现，其实主要是将该代理需要实现的接口给确定好，该接口存储在invoker对象的url的interfaces参数中，是一个字符串，用逗号分隔，我们不需要关心interfaces中存储的到底是哪些接口，只需要知道的是，invoker.getInterface()会加入到动态代理中，而另一个重载的getProxy方法显然是丢给子类来实现了。AbstractProxyFactory有两个实现类：JdkProxyFactory和JavassistProxyFactory，顾名思义，一个是使用JDK的动态代理，一个是使用Javaassist来实现动态代理，我们直接看JdkProxyFactory的实现：



**JdkProxyFactory的实现**

```java
public class JdkProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")

    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {

        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));

    }

 

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {

        return new AbstractProxyInvoker<T>(proxy, type, url) {

            @Override

            protected Object doInvoke(T proxy, String methodName,

                                      Class<?>[] parameterTypes,

                                      Object[] arguments) throws Throwable {

                Method method = proxy.getClass().getMethod(methodName, parameterTypes);

                return method.invoke(proxy, arguments);

            }

        };

    }

}
```

在JdkProxyFactory的getProxy方法中我们看到了熟悉的类：InvokerInvocationHandler，很显然它实现了JDK中的InvocationHandler接口



### 总结一下

如果一个类的属性是dubbo接口，会触发getBean方法进行依赖注入，会触发org.apache.dubbo.config.spring.ReferenceBean的getObject()方法获取proxy引用，并且把该proxy注入到属性上面。

```java
<dubbo:reference id="demoServiceYuan"

interface="com.spring.DemoServiceYuan"

version="1.0" />

```



![Snip20181208_2](https://ws2.sinaimg.cn/large/006tNbRwgy1fxzg009ykmj30vm0d2tan.jpg)









### 扩展

在Dubbo中，没有使用CGLib进行代理，而是使用JDK和Javassist来进行动态代理！我们知道，动态代理是无法用反射做的，只能靠动态生成字节码，这就需要使用字节码工具包，比如asm和Javassist等，在Spring3.2.2之前版本的源码中，我们可以看到是有单独spring-asm的模块的，但在Spring3.2.2版本开始，就没有spring-asm模块了，不是不使用了，而是spring-asm已经整合到spring-core中了，可见asm在Spring中的地位（CGLib使用的就是asm），至于Dubbo为什么不使用CGLib，当我们选择动态代理实现时，无非考虑的是如下因素：

- 使用的难易程度。

- 功能，比如JDK的代理只能通过接口实现，而CGLib则只能通过类来实现

- 生成的字节码的效率，包括创建的效率和运行的效率。

- 生成的字节码大小，为什么说这也比较重要？这和生成的Class和其实例的大小相关。

至于具体的原因，可以参考Dubbo作者的博客：http://javatar.iteye.com/blog/814426



那么，接下来的问题就是，什么时候会使用JDK的动态代理，而什么时候使用Javassist的动态代理呢？在dubbo-rpc-api的模块中，有个com.alibaba.dubbo.rpc.ProxyFactory文件，里面定义了代理的几种方式：

stub=com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper

jdk=com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory

javassist=com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory



StubProxyFactoryWrapper是存根（stub）的代理工厂，主要用于暴露服务提供者的本地服务给远端消费者来调用。可以直接通过xml的配置来手动选择代理，比如：

<dubbo:provider proxy="jdk" />或<dubbo:consumer proxy="jdk" />

如果不配置，由于ProxyFactory接口上有@SPI("javassist")注解，所以默认是使用Javassist来实现动态代理！