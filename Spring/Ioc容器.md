#IoC



## IoC初始化过程

- 资源定位，资源解析（这个资源一般指的是配置文件）
   - 找到资源文件，载入。
   - 把资源文件转化成DOM元素
- 注册到IoC中
- 关键类：org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions
   - 这时候bean还没有实例化
- 注册的过程主要是DOM元素转化为BeanDefinition，并且存放到Map中。把DOM元素转化为BeanDefinition主要由NamespaceHandler决定，这里也体现了Spring的扩展性
- 调用所有注册的 BeanFactoryPostProcessor;
- 注册所有的 BeanPostProcessor;
- 单例Bean的实例化（开始Bean的生命周期） org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
- 调用beanPostProcessor的地方org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
   - 依赖注入



## 关键类

- BeanDefinition

  XML中bean配置的Java映射, 也就是说XML配置了一个Bean, 就会有一个BeanDefinition对象与之对应, 它保存了Bean的各种信息, 包括scope/lazy-init等.

- BeanFactory

  Bean工厂, 保存了所有的Bean并管理它们的生命周期和依赖关系.

- ApplicationContext

  更高级的Bean工厂, 除了BeanFactory提供的能力之外, 还支持包括国际化/事件消息等.

  ApplicationContext本身就是一个资源加载器





## Resource定位,载入和注册

- 入口

```xml
<context-param>

        <param-name>contextConfigLocation</param-name>

        <param-value>classpath*:spring/applicationContext.xml</param-value>

    </context-param>

    <listener>

        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>

    </listener>

```

ContextLoaderListener 实现了 javax.servlet.ServletContextListener，这个是JavaEE中Servlet的一个类。当tomcat启动的时候，加载web.xml文件，然后初始化Servlet。在初始化Servlet上下文之后, 会执行org.springframework.web.context.ContextLoaderListener的 `contextInitialized`方法. 这里是Spring初始化的入口。

- 创建WebApplicationContext

默认是 XmlWebApplicationContext.

- refresh [重点]

**org.springframework.context.support.AbstractApplicationContext#refresh**

不论是BeanFactory还是ApplicationContext, 都会执行 `org.springframework.context.support.AbstractApplicationContext#refresh` 进行真正的初始化工作.

其中 `obtainFreshBeanFactory`是进行 **Resource定位,载入和注册**, 执行整个方法之后, XML中所有Bean的配置都被转化成BeanDefinition, 并保存在 DefaultListableBeanFactory 的 beanDefinitionNames 和 beanDefinitionMap 中;

\- Resource定位

  **org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.xml.XmlBeanDefinitionReader)**

  这个过程就是Spring找到XML文件, 这个文件可能在文件系统/类路径/网络路径中.

  **ResourceLoader**, 如果我们使用的是 FileSystemXmlApplicationContext, 那么 ResourceLoader 就是 FileSystemXmlApplicationContext, 它本身就是一个DefaultResourceLoader. 如果我们使用的是XmlWebApplicationContext, 那就是使用 XmlWebApplicationContext了.

- 载入(Bean解析)

  **org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource...)**

  这个过程就是把找到的XML中的Bean定义转化为BeanDefinition.

  真正的解析工作在 **org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions**, 这里有个比较重要的点: XML的命名空间.

  如果是Spring默认的命名空间 **http://www.springframework.org/schema/beans**, 这个Bean的定义的命名空间, 明显Spring是知道怎么去解析的. 但是有一些自定义的命名空间, 比如Dubbo的配置文件, 这种的话, Spring是不知道要怎么去解析的. 所有Spring扩展了解析器, 自定义的命名空间就交给自定义的解析器去解析成BeanDefinition. 那么Spring是怎么找到这个自定义命名空间的解析器的呢? `META-INF/spring.handlers`中 `http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler`. 这里有个约定的规范, 比如spring的context命名空间, 那么它的解析器类名是 ContextNamespaceHandler, aop命名空间的是AopNamespaceHandler, dubbo的是DubboNamespaceHandler.

- 解析Bean

  **org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement**

- 注册

**org.springframework.beans.factory.support.BeanDefinitionReaderUtils#registerBeanDefinition**

把BeanName和BeanDefinition注册到 DefaultListableBeanFactory.



单例类的初始化

**org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization**

\- 遍历容器中所有的beanDefinitionNames, 逐个初始化.







##Bean的生命周期

入口:org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

在实例化Bean的时候，开启Bean的生命周期。初始化的过程，简单来说就是**构造器初始化Bean，进行依赖注入，在初始化的过程中调用一些扩展**，其流程如下：



- 调用构造器初始化

- 调用InstantiationAwareBeanPostProcessor#postProcessPropertyValues

- 进行依赖注入 org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean
  - 触发@Autowired/@Value/@Inject 依赖注入
  - 触发@Resouce的依赖注入 
  - 真正进行注入org.springframework.context.annotation.CommonAnnotationBeanPostProcessor

会触发getBeanName

- BeanNameAware#setBeanName  让实现这个接口的bean **可以获取到自己在spring容器里的名字**
  - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean

- BeanFactoryAware#setBeanFactory 
  - 通过该方法可以获取容器，因此可以做到通过容器获取bean

- BeanPostProcessor#postProcessBeforeInitialization
- @PostConstruct  
  - @PostConstruct注解的方法将会在依赖注入完成后被自动调用
  - @PostConstruct修饰的方法在构造函数之后执行，在init-method设置的方法之前执行
- InitializingBean#afterPropertiesSet
  - 也是扩展的一种，可以针对bean做配置
- 自定义init-method（通过在bean标签的init-method方法定义）
- BeanPostProcessor#postProcessAfterInitialization

**这样子bean就初始化好了**

...

**当容器停止的时候，依次调用**

1. @PreDestroy

   @PreDestroy修饰的方法在被bean被销毁之前执行

2. DisposableBean#destroy

3. destory-method





##依赖注入

遍历Ioc容器中的对象，遍历对象的属性，然后如果该属性是对象，则把Ioc容器中的对应对象的引用赋值到属性上。



### 构造器注入

```java
public class DemoC {

    private DemoA demoA;

    public DemoC(DemoA demoA) {

    }
    
}
```

```xml
<bean id="demoC" class="com.model.DemoC">
  <constructor-arg ref="demoA"/>
</bean>
```







### set注入

```java
public class DemoA {

    private DemoB demoB;

    public void setDemoB(DemoB demoB) {
        this.demoB = demoB;
    }
}
```

```xml
// 注入demoB对象
<bean id="demoA" class="com.model.DemoA">
  	<property name="demoB" ref="demoB"/>
</bean>
```







###注解注入

如常见的@Resource、@Controller





## 附录

### 三级缓存

三级缓存: 

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)

https://blog.csdn.net/github_38687585/article/details/82317674

https://blog.csdn.net/herzog_/article/details/88690759



### Spring如何解决循环依赖的问题

- Setter依赖 [可以解决]

  Spring容器提前暴露完成构造器注入但是未完成其他步骤（如set注入）的bean，然后当双方再进入set注入的时候，对象已经在Ioc容器里面了，因此可以直接注入，从而避免循环依赖的问题。

- 构造器依赖 [无法解决]

  比如A的构造器注入了B，B的构造器注入了A，从而造成构造器循环依赖。

  此循环依赖无法解决，只能抛出 BeanCurrentlyInCreationException 表示循环依赖；

- prototype范围的依赖 [无法解决]

  无法解决，因为Spring容器不进行缓存“prototype”作用域的bean，因此无法提前暴露一个创建中的bean；对于“singleton”作用域的bean，可以通过设置“setAllowCircularReferences(false)”来禁用循环依赖；
  
  （singleton 只有一个实例，也即是单例模式。prototype访问一次创建一个实例，相当于new）







### Autowired与Resource注入的区别；

- @Autowired（或@Inject）和@Resource都同样适用。 但是意义上存在概念差异或差异
- @Resource意味着按名称注入一个已知的资源。 该名称是从带注解的setter或字段的名称中提取的，或者是从name参数中获取的。实现类为`CommonAnnotationBeanPostProcessor`
- @Inject或@Autowired尝试按类型注入适当的其他组件。实现类为`AutowiredAnnotationBeanPostProcessor`
- 所以，基本上这些是两个截然不同的概念。 不幸的是，@Resource的Spring实现有一个内置的fallback，当按名称解析失败时会启动。 在这种情况下，它会回退到@Autowired类型的解决方式。 虽然这种回退方法很方便，但它引起了很多混淆，因为人们不了解概念上的差异，并倾向于使用@Resource进行基于类型的自动装配。



- @Autowired 和 @Inject

1. Matches by Type

2. Restricts by Qualifiers

3. Matches by Name

- @Resource

1. Matches by Name
2. Matches by Type
3. Restricts by Qualifiers (ignored if match is found by name)



- 最佳实践

  - 用@Component("beanName")显式声明你的bean；

  - 使用@Resource并带上name属性，@Resource(name="beanName")

  - 除非您想创建一个类似的bean列表，否则请避免使用@Qualifier注解。 例如，您可能想要使用特定的@Qualifier注释来标记一组规则。 这种方法使得将一组规则类插入可用于处理数据的列表变得简单。

  - 使用扫描组件的特定包<contextcomponent-scan base-package="com.sourceallies.person"/>。 虽然这会导致更多的组件扫描配置，但它会减少您向Spring上下文添加不必要组件的可能性。









### BeanPostProcessor 与 BeanFactoryPostProcessor

BeanFactoryPostProcessor：BeanFactory后置处理器，是对BeanDefinition对象进行修改。BeanFactoryPostProcessor接口是针对bean容器的，它的实现类可以在当前BeanFactory初始化（spring容器加载bean定义文件）后，bean实例化之前修改bean的定义属性，达到影响之后实例化bean的效果。



BeanPostProcessor：Bean后置处理器，是对生成的Bean对象进行修改。BeanPostProcessor能在spring容器实例化bean之后，在执行bean的初始化方法前后，添加一些自己的处理逻辑

执行逻辑：  postProcessBeforeInitialization --->  init-method（bean标签里面的） ---> postProcessAfterInitialization



https://blog.csdn.net/zhanyu1/article/details/83114684





### BeanFacotry 与 FactoryBean

https://www.jianshu.com/p/05c909c9beb0





###作用域（scope）

beanFactory除了拥有ioc的职责外，还有着对象生命周期管理。

scope用来声明容器中对象所应该处的限定场景或者该对象的存活时间，即容器在对象进入其相应的scope之前，生成并装配这些对象，在对象不再处于scope的限定之后，容器通常会销毁这些对象。

**spring容器提供了几种scope类型？**

- singleton：在spring容器中只存在一个实例，所有对象的引用将共享这个实例。（注：不要和单例模式搞混）
- prototype：容器每次都会生成一个新的对象实例给请求方。
- request （限定在web应用中使用）：为每个http请求创建一个全新的request-processor对象供当前请求使用，请求结束，实例生命周期即结束。
- session （限定在web应用中使用）：为每个独立的session创建一个全新的对象实例。
- global session （限定在web应用中使用）：只有应用在基于portlet的Web应用程序中才有意义，它映射到portlet的global范围的*session。*如果在普通的基于servlet的Web应用中使用了这个类型的scope，容器会将其作为普通的session类型的scope对待。







## 参考文档

- [archerda: spring-framework](https://github.com/archerda/spring-framework)
- [Spring IoC容器分析](http://www.importnew.com/27469.html)
- https://www.cnblogs.com/chenpt/p/9896618.html

  