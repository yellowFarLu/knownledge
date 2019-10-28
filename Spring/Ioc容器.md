# IoC

首先了解一下控制反转：

什么是控制？控制就是对对象进行创建、销毁、依赖对象这些类操作。

什么是“反转”（叫“转移”更为贴切）？“反转”的意思就是**将“控制”的操作交由容器来处理，自己只管用，只管要，其他的都不管。**





## IoC容器初始化过程

- **资源定位，资源解析**
   
   - 找到资源文件，载入。
   - 把资源文件转化成DOM元素
   - 这个资源一般指的是配置文件
   
- **把Bean定义注册到IoC容器中**

   - 把BeanDefinition加入到 org.springframework.beans.factory.support.DefaultListableBeanFactory#**beanDefinitionMap**里面，这个的BeanDefinition相当于类，这时候还没有把该类进行实例化。
   - 关键类：org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions
   - 注册的过程主要是DOM元素转化为BeanDefinition，并且存放到Map中。把DOM元素转化为BeanDefinition主要由NamespaceHandler决定，这里也体现了Spring的扩展性

- **注册并且调用所有的BeanFactory的后置处理器——BeanFactoryPostProcessor**

   - 这一阶段程序员可以对BeanDefinition进行修改

- **注册所有Bean的后置处理器——BeanPostProcessor**

- **实例化单例Bean**    

   - **依赖注入**：将对象的属性赋值。这个值就已经是实例化之后的对象了。

   - org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization

   - 这个阶段开始Bean的生命周期，**为bean分配内存空间，初始化，调用各种扩展，如：beanPostProcessor、afterPropertySet等**

   - 调用beanPostProcessor的地方org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization

      



## 关键类

- BeanDefinition

  **XML中bean配置的Java映射**, 也就是说XML配置了一个Bean, 就会有一个BeanDefinition对象与之对应, 它保存了Bean的各种信息, 包括scope/lazy-init等。有点像于**类**。

- BeanFactory

  Bean工厂， **保存了所有的Bean**并**管理它们的生命周期**和**依赖关系**.

- ApplicationContext

  更高级的Bean工厂，管理Bean的生命周期和依赖关系，还支持包括国际化/事件消息等.

  ApplicationContext本身就是一个资源加载器。





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
  - 默认是 XmlWebApplicationContext.

- refresh [重点]
  - **org.springframework.context.support.AbstractApplicationContext#refresh**
  - 不论是BeanFactory还是ApplicationContext, 都会执行 `org.springframework.context.support.AbstractApplicationContext#refresh` 进行真正的初始化工作.
  - 其中 `obtainFreshBeanFactory`是进行 **Resource定位,载入和注册**, 执行整个方法之后, XML中所有Bean的配置都被转化成BeanDefinition, 并保存在 DefaultListableBeanFactory 的 beanDefinitionNames 和 beanDefinitionMap 中;



### Resource定位

-  这个过程就是**Spring找到XML文件, 这个文件可能在文件系统/类路径/网络路径中.**
- **ResourceLoader**, 如果我们使用的是 FileSystemXmlApplicationContext, 那么 ResourceLoader 就是 FileSystemXmlApplicationContext, 它本身就是一个DefaultResourceLoader. 如果我们使用的是XmlWebApplicationContext, 那就是使用 XmlWebApplicationContext了.
- 代码：org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.xml.XmlBeanDefinitionReader)



### 载入(Bean解析)

- **这个过程就是把找到的XML中的Bean定义转化为BeanDefinition**

- 真正的解析工作在 **org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions**, 这里有个比较重要的点: XML的命名空间.

- 如果是Spring默认的命名空间 **http://www.springframework.org/schema/beans**, 这个Bean的定义的命名空间, 明显Spring是知道怎么去解析的. 但是有一些自定义的命名空间, 比如Dubbo的配置文件, 这种的话, Spring是不知道要怎么去解析的. 所有Spring扩展了解析器, 自定义的命名空间就交给自定义的解析器去解析成BeanDefinition. 那么Spring是怎么找到这个自定义命名空间的解析器的呢? `META-INF/spring.handlers`中 `http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler`. 这里有个约定的规范, 比如spring的context命名空间, 那么它的解析器类名是 ContextNamespaceHandler, aop命名空间的是AopNamespaceHandler, dubbo的是DubboNamespaceHandler.

- 代码：
  - org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement
  - org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource...)



### 注册

- **把BeanName和BeanDefinition注册到 DefaultListableBeanFactory**

- org.springframework.beans.factory.support.BeanDefinitionReaderUtils#registerBeanDefinition





## 单例类的初始化

**org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization**

\- 遍历容器中所有的beanDefinitionNames, 逐个初始化.







##Bean的生命周期

入口:org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

在实例化Bean的时候，开启Bean的生命周期。初始化的过程，简单来说就是**构造器初始化Bean，进行依赖注入，在初始化的过程中调用一些扩展**，其流程如下：



- **调用构造器初始化**
- 调用InstantiationAwareBeanPostProcessor#postProcessPropertyValues
- **进行依赖注入 **
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean
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





## 扩展



### 循环依赖的问题



#### 理论依据

Spring解决循环依赖的理论依据是：**当我们获取到对象的引用时，对象的属性是可以延后设置的**

Spring中单例对象的初始化可以分为三步：

- 调用构造方法构造对象
- 填充属性
- 调用配置文件中init方法初始化

第一步和第二步是可能发生循环依赖的。



#### 三级缓存

Spring为了解决循环依赖问题，使用了三级缓存。如下：

```java
/**
 * 单例对象的缓存（一级缓存）
 **/
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/**
 * 提前曝光的单例对象的缓存（二级缓存）
 **/
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/**
 * 单例对象工厂的缓存（三级缓存）
 **/
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```



**单例对象获取流程**

- Spring首先从singletonObjects（一级缓存）中尝试获取
- 如果获取不到并且对象在创建中，则尝试从earlySingletonObjects(二级缓存)中获取
- 如果还是获取不到并且允许从三级获取，则通过singletonFactory.getObject()(三级缓存)获取。如果获取到了则将单例对象从三级缓存提升到二级缓存中



**Spring解决循环依赖的诀窍就在于singletonFactories这个缓存。**

当单例这个对象已经被生产出来了，虽然还不完美（还没有进行属性填充、初始化），但是已经能被人认出来了（**根据对象引用能定位到堆中的对象**），所以Spring此时将这个对象提前曝光出来让大家认识



#### 如何解决

- Setter依赖 [可以解决]

  **Spring容器提前暴露完成构造器注入但是未完成其他步骤（如set注入）的bean**，然后当双方再进入set注入的时候，对象已经在Ioc容器里面了，因此可以直接注入，从而避免循环依赖的问题。

- 构造器依赖 [无法解决]

  比如A的构造器注入了B，B的构造器注入了A，从而造成构造器循环依赖。

  此循环依赖无法解决，只能抛出 BeanCurrentlyInCreationException 表示循环依赖；

- prototype范围的依赖 [无法解决]

  无法解决，**因为Spring容器不进行缓存“prototype”作用域的bean，因此无法提前暴露一个创建中的bean**，；对于“singleton”作用域的bean，可以通过设置“setAllowCircularReferences(false)”来禁用循环依赖；
  



**例子**

“A的某个field或者setter依赖了B的实例对象，同时B的某个field或者setter依赖了A的实例对象”这种循环依赖的情况。A首先完成了初始化的第一步，并且将自己提前曝光到singletonFactories中，此时进行初始化的第二步，发现自己依赖对象B，此时就尝试去get(B)，发现B还没有被create，所以走create流程，B在初始化第一步的时候发现自己依赖了对象A，于是尝试get(A)，尝试一级缓存singletonObjects(肯定没有，因为A还没初始化完全)，尝试二级缓存earlySingletonObjects（也没有），尝试三级缓存singletonFactories，由于A通过ObjectFactory将自己提前曝光了，所以B能够通过ObjectFactory.getObject拿到A对象(虽然A还没有初始化完全，但是总比没有好呀)，B拿到A对象后顺利完成了初始化阶段1、2、3，完全初始化之后将自己放入到一级缓存singletonObjects中。此时返回A中，A此时能拿到B的对象顺利完成自己的初始化阶段2、3，最终A也完成了初始化，长大成人，进去了一级缓存singletonObjects中，而且更加幸运的是，由于B拿到了A的对象引用，所以B现在hold住的A对象也蜕变完美了！一切都是这么神奇！！




### Autowired与Resource区别

- **@Autowired（或@Inject）和@Resource都同样适用。 但是意义上存在概念差异**
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

- BeanFactoryPostProcessor：
  - BeanFactory后置处理器，是对BeanDefinition进行修改。
  - BeanFactoryPostProcessor接口是针对bean容器的，它的实现类可以在**当前BeanFactory初始化（spring容器加载bean定义文件）后，bean实例化之前**修改BeanDefinition，达到影响之后实例化bean的效果。

- BeanPostProcessor：
  - Bean后置处理器，是对生成的Bean对象进行修改。
  - BeanPostProcessor能在spring容器实例化bean之后，在执行bean的初始化方法前后，添加一些自己的处理逻辑

- BeanPostProcessor执行逻辑： 
  - postProcessBeforeInitialization --->  init-method（bean标签里面的） ---> postProcessAfterInitialization





### BeanFactory 与 FactoryBean

- BeanFactory是Beean的容器，它负责管理Bean的生命周期和依赖关系

- FactoryBean是一个Bean，但是这个Bean可以产生其他Bean实例。程序员可以通过FactoryBean按照自己的想法去产生Bean。

  - ```java
    public class Student implements Serializable {
    
        public Student(String abc) {
            this.abc = abc;
        }
    
        private String abc;
    
        public String getAbc() {
            return abc;
        }
    
        public void setAbc(String abc) {
            this.abc = abc;
        }
    
        @Override
        public String toString() {
            return "Student{" +
                    "abc='" + abc + '\'' +
                    '}';
        }
    }
    ```

    ```java
    @Component
    public class StudentFactoryBean implements FactoryBean<Student> {
    
        @Override
        public Class<?> getObjectType() {
            return Student.class;
        }
    
        @Override
        public Student getObject() throws Exception {
            return new Student("黄远");
        }
    
        @Override
        public boolean isSingleton() {
            return true;
        }
    }
    ```

    ```java
    public class StudentFactoryBeanTest extends SpringTestCase {
    
        /**
         * 注意这里使用Student，也就是创建出来的Bean类型
         */
        @Resource
        Student studentFactoryBean;
    
        @Test
        public void func() {
            System.out.println(studentFactoryBean);
        }
    
    }
    ```









## 参考文档

- [archerda: spring-framework](https://github.com/archerda/spring-framework)

- [Spring IoC容器分析](http://www.importnew.com/27469.html)

- https://www.cnblogs.com/chenpt/p/9896618.html

- [Spring通过三级缓存解决循环依赖](https://blog.csdn.net/github_38687585/article/details/82317674)

- [BeanPostProcessor 与 BeanFactoryPostProcessor](https://blog.csdn.net/zhanyu1/article/details/83114684)

- [BeanFacotry 与 FactoryBean](https://www.jianshu.com/p/05c909c9beb0)

  