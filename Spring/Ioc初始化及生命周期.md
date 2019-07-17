#IoC初始化



## 概览

IoC初始化主要包括:

- Resource定位,载入DOM解析和注册到IoC中;

   关键类：org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

   这时候bean还没有实例化，只是信息都存放在beanDefinition中

- 调用所有注册的 BeanFactoryPostProcessor;

- 注册所有的 BeanPostProcessor;

- 单例Bean的实例化; org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization

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

ContextLoaderListener 实现了 javax.servlet.ServletContextListener, 这个是JavaEE中Servlet的一个类, 在初始化Servlet上下文之后的时候, 会执行它的 `contextInitialized`方法. 这里是Spring初始化的入口.

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

### Bean的生命周期

入口:org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

1. Constructor 调用构造器初始化

2. InstantiationAwareBeanPostProcessor#postProcessPropertyValues (org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean)

- 触发@Autowired/@Value/@Inject 依赖注入

- 触发@Resouce的依赖注入 

  真正进行注入org.springframework.context.annotation.CommonAnnotationBeanPostProcessor

  会触发getBeanName

3. BeanNameAware#setBeanName  让Bean获取自己在BeanFactory配置中的名字(**org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)**)

4. BeanFactoryAware#setBeanFactory 

5. ApplicationContextAware#setApplicationContext

6. BeanPostProcessor#postProcessBeforeInitialization

7. @PostConstruct  注释用于在依赖关系注入完成之后需要执行的方法上（通过调用beanPostProcessor调用）

8. InitializingBean#afterPropertiesSet

9. 自定义init-method（通过在bean标签的init-method方法定义）

10. BeanPostProcessor#postProcessAfterInitialization


Bean is Ok

...

Container shutdown

1. @PreDestroy

2. DisposableBean#destroy

3. destory-method



## 附录

### Spring如何解决循环依赖的问题

- 三级缓存: org.springframework.beans.factory.support.DefaultSingletonBeanRegistry

- Setter依赖 [可以解决]

  通过提前暴露一个单例工厂方法(ObjectFactory)，从而使其他bean能引用到该bean；

- 构造器依赖 [无法解决]

  此循环依赖无法解决，只能抛出 BeanCurrentlyInCreationException 表示循环依赖；

- prototype范围的依赖 [无法解决]

  无法解决，因为Spring容器不进行缓存“prototype”作用域的bean，因此无法提前暴露一个创建中的bean；对于“singleton”作用域的bean，可以通过设置“setAllowCircularReferences(false)”来禁用循环依赖；

### Autowired与Resource注入的区别；

- @Autowired（或@Inject）和@Resource都同样适用。 但是意义上存在概念差异或差异

- @Resource意味着按名称注入一个已知的资源。 该名称是从带注解的setter或字段的名称中提取的，或者是从name参数中获取的。

- @Inject或@Autowired尝试按类型注入适当的其他组件。

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

### BeanFacotry 与 FactoryBean

## 参考文档

- [archerda: spring-framework](https://github.com/archerda/spring-framework)

- [Spring IoC容器分析](http://www.importnew.com/27469.html)