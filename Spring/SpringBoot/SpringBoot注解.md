# SpringBoot注解



## 前言

在spring boot中，摒弃了spring以往项目中大量繁琐的配置，遵循约定大于配置的原则，通过自身默认配置，极大的降低了项目搭建的复杂度。同样在spring boot中，大量注解的使用，使得代码看起来更加简洁，提高开发的效率。这些注解不光包括spring boot自有，也有一些是继承自spring的。

本文中将spring boot项目中常用的一些核心注解归类总结，并结合实际使用的角度来解释其作用。







## 项目配置注解



### @SpringBootApplication 注解

查看源码可发现，@SpringBootApplication是一个复合注解，包含了@SpringBootConfiguration，@EnableAutoConfiguration，@ComponentScan这三个注解。

**这三个注解的作用分别为：**

- @SpringBootConfiguration：标注当前类是配置类，这个注解继承自@Configuration。并会将当前类内声明的一个或多个以@Bean注解标记的方法的实例纳入到spring容器中，并且实例名就是方法名。
- @EnableAutoConfiguration：是自动配置的注解，这个注解会根据我们添加的组件jar来完成一些默认配置，比如：我们做微服务时会添加spring-boot-starter-web这个组件jar的pom依赖，这样配置会默认配置springmvc 和tomcat。
- @ComponentScan：扫描**被注解类的 当前包及其子包下**被@Component，@Controller，@Service，@Repository注解标记的类并注入到spring容器中进行管理。等价于<context:component-scan>的xml配置文件中的配置项。

大多数情况下，这3个注解会被同时使用，因此，这三个注解就被做了包装，成为了@SpringBootApplication注解。



### @ServletComponentScan

使用@ServletComponentScan 注解后：

- Servlet可以直接通过@WebServlet注解自动注册
- Filter可以直接通过@WebFilter注解自动注册
- Listener可以直接通过@WebListener 注解自动注册



### @MapperScan

作用：@MapperScan指定要变成实现类的接口所在的包，然后包下面的所有接口在编译之后都会生成相应的实现类。

添加位置：是在Springboot启动类上面添加。

**例子**：

```java
@SpringBootApplication
@MapperScan("com.winter.dao")
public class SpringbootMybatisDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootMybatisDemoApplication.class, args);
    }
}
// 添加@MapperScan(“com.winter.dao”)注解以后，com.winter.dao包下面的接口类，在编译之后都会生成相应的实现类
```

它和@mapper注解是一样的作用，不同的地方是扫描入口不一样。@mapper需要加在每一个mapper接口类上面。所以大多数情况下，都是在规划好工程目录之后，通过@MapperScan注解配置路径完成mapper接口的注入。

添加mybatis相应组建依赖之后。就可以使用该注解。

![image-20201026115814385](https://tva1.sinaimg.cn/large/0081Kckwgy1gk2lm5uis8j30h603zgoa.jpg)



**使用@MapperScan注解多个包**
（实际用的时候根据自己的包路径进行修改）

```java
@SpringBootApplication  
@MapperScan({"com.kfit.demo","com.kfit.user"})  
public class App {  
    public static void main(String[] args) {  
       SpringApplication.run(App.class, args);  
    }  
} 
```

MapperScan注解的使用参考：https://blog.csdn.net/manchengpiaoxue/article/details/84937257



### **资源导入注解**

@ImportResource @Import @PropertySource 这三个注解都是用来导入自定义的一些配置文件。



#### @ImportResource

@ImportResource用于导入其他xml配置文件，可以看到其参数指定的是配置文件路径。

注意我们的配置文件一般都在resources文件夹下面，所以路径上面最好加上classpath:前缀，这样子就会从java环境变量中查找了，也就会从resources文件夹下面查找了

```java
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ImportResource;

@MapperScan("com.example.mapper") //扫描的mapper
@ImportResource(value = "classpath:provider.xml")
@SpringBootApplication
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }

}
```



#### @Import

其参数指定的是Class信息，也就是指向配置类。

 我们来创建一个类，用来生成Book相关Bean，如下：

```java
// 创建BookConfiguration
@Configuration
public class BookConfiguration {
 
    @Bean
    public Book book(){
        return new Book();
    }
}
 
// 替换上文的@ImportResource方式为@Import
@Configuration
@Import(value = BookConfiguration.class)
public class SpringConfiguration {
 
    @Bean(name = "stu",autowire = Autowire.BY_TYPE)
    @Scope(value = "singleton")
    public Student student(){
        return new Student(11,"jack",22);
    }
}
 
// 测试
AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(SpringConfiguration.class);
Book book = (Book) applicationContext.getBean("book");
System.out.println(book);
 
// 结果
Book(id=0, bookName=null, desc=null, author=null)
```



#### @PropertySource

当应用比较大的时候，如果所有的内容都当在一个配置文件中，就会显得比较臃肿，同时也不太好理解和维护，此时可以将一个文件拆分为多个，使用 @PropertySource 注解加载指定的配置文件。

下面演示使用 @PropertySource 注解 加载类路径下的 user.properties 配置文件，为 User.java POJO 对象的属性赋值。

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;
import java.util.Date;
import java.util.List;
import java.util.Map;

/**
 * @ConfigurationProperties 表示 告诉 SpringBoot 将本类中的所有属性和配置文件中相关的配置进行绑定；
 * prefix = "user" 表示 将配置文件中 key 为 user 的下面所有的属性与本类属性进行一一映射注入值，如果配置文件中不存在 "user" 的 key，则不会为 POJO 注入值，属性值仍然为默认值
 
 * @Component 将本类标识为一个 Spring 组件，因为只有是容器中的组件，容器才会为 @ConfigurationProperties 提供此注入功能
 
 * @PropertySource (value = { " classpath : user.properties " }) 指明加载类路径下的哪个配置文件来注入值
 */
@PropertySource(value = {"classpath:user.properties"})
@Component
@ConfigurationProperties(prefix = "user")
public class User {
    private Integer id;
    private String lastName;
    private Integer age;
    private Date birthday;
    private List<String> colorList;
    private Map<String, String> cityMap;
    private Dog dog;
 
    //省略 getter、setter 、toString 方法没粘贴
}
```

```java
/**
 * 狗 实体类-----------普通 Java Bean。 
 * 被依赖项可以不加 @ConfigurationProperties 注解，但是必须提供 getter、setter 方法
 */
public class Dog {
    private Integer id;
    private String name;
    private Integer age;
    //省略 getter、setter 、toString 方法
}
```

在类路径下提供 user.properties 配置文件如下:

```properties
user.id=111
user.lastName=张无忌
user.age=120
user.birthday=2018/07/11
user.colorList=red,yellow,green,blacnk
user.cityMap.mapK1=长沙市
user.cityMap.mapK2=深圳市
user.dog.id=7523
user.dog.name=狗不理
user.dog.age=3
```

测试运行：

```java
import com.wmx.yuanyuan.pojo.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import javax.annotation.Resource;
@RunWith(SpringRunner.class)
@SpringBootTest
public class YuanyuanApplicationTests {
    /**
     * 因为 POJO类 使用了 Component 注解，就是 Spring 一个组件，交由了 Spring 容器注入值
     * 所以使用 @Autowired 或者 @Resource，DI 注入在取值即可
     */
    @Resource
    private User user;
    @Test
    public void contextLoads() {
        System.out.println("------------------------------********************--------------------------------------------");
        System.out.println("·······················" + user);
    }
}
```

参考：https://blog.csdn.net/wangmx1993328/article/details/81005170









## controller层注解



### @Controller

表明这个类是一个控制器类，和@RequestMapping来配合使用拦截请求，如果不在method中注明请求的方式，默认是拦截get和post请求。这样请求会完成后转向一个视图解析器。但是在大多微服务搭建的时候，前后端会做分离。所以请求后端只关注数据处理，后端返回json数据的话，需要配合@ResponseBody注解来完成。

这样一个只需要返回数据的接口就需要3个注解来完成，大多情况我们都是需要返回数据。也是基于最佳实践，所以将这三个注解进一步整合。



### @RestController

@RestController 是@Controller 和@ResponseBody的结合，一个类被加上@RestController 注解，数据接口中就不再需要添加@ResponseBody。更加简洁。

同样的情况,@RequestMapping(value="",method= RequestMethod.GET ),我们都需要明确请求方式。这样的写法又会显得比较繁琐，于是乎就有了如下的几个注解：

| **普通风格**                                                | ***\*Rest风格\****            |
| ----------------------------------------------------------- | ----------------------------- |
| **@RequestMapping(value=“”,method = RequestMethod.GET)**    | **@GetMapping(value =“”)**    |
| **@RequestMapping(value=“”,method = RequestMethod.POST)**   | **@PostMapping(value =“”)**   |
| **@RequestMapping(value=“”,method = RequestMethod.PUT)**    | **@PutMapping(value =“”)**    |
| **@RequestMapping(value=“”,method = RequestMethod.DELETE)** | **@DeleteMapping(value =“”)** |

这几个注解是 @RequestMapping(value="",method= RequestMethod.xxx )的最佳实践。为了代码的更加简洁。



### **@CrossOrigin**

@CrossOrigin(origins = "", maxAge = 1000) 这个注解主要是为了解决跨域访问的问题。这个注解可以为整个controller配置启用跨域，也可以在方法级别启用。

我们在项目中使用这个注解是为了解决微服务在做定时任务调度编排的时候，会访问不同的spider节点而出现跨域问题。



### **@Autowired**

这是个最熟悉的注解，是spring的自动装配，这个个注解可以用到构造器，变量域，方法，注解类型上。当我们需要从bean 工厂中获取一个bean时，Spring会自动为我们装配该bean中标记为@Autowired的元素。



### **@EnablCaching**

@EnableCaching 这个注解是spring framework中的注解驱动的缓存管理功能。自spring版本3.1起加入了该注解。其作用相当于spring配置文件中的cache manager标签。



### **@PathVariable**

@PathVariable：标记一个方法参数，该参数的值将使用URI模板中对应的变量的值来赋值

![image-20201026164331175](https://tva1.sinaimg.cn/large/0081Kckwgy1gk2tuzps09j30k502i764.jpg)

同样可以支持变量名加正则表达式的方式，变量名:[正则表达式]。

![image-20201026164437600](https://tva1.sinaimg.cn/large/0081Kckwgy1gk2tw3kkujj30k803ojtu.jpg)









## **servcie层注解**



### **@Service**

@Service 这个注解用来标记业务层的组件，我们会将业务逻辑处理的类都会加上这个注解交给spring容器。事务的切面也会配置在这一层。



### **@Resource**

@Resource和@Autowired一样都可以用来装配bean，都可以标注字段上，或者方法上。 @resource注解不是spring提供的，是属于J2EE规范的注解。两个之前的区别就是匹配方式上有点不同，@Resource默认按照名称方式进行bean匹配，@Autowired默认按照类型方式进行bean匹配。







## **持久层注解**



### **@Repository**

**@Repository注解类作为DAO对象，管理操作数据库的对象。**

总的来看，@Component, @Service, @Controller, @Repository是spring注解，注解后，类可以被spring框架所扫描并注入到spring容器来进行管理。



### **@Component**

@Component是通用注解，其他三个注解是这个注解的拓展，并且具有了特定的功能。

通过这些注解的分层管理，就能将请求处理，义务逻辑处理，数据库操作处理分离出来，为代码解耦，也方便了以后项目的维护和开发。

所以我们在正常开发中，如果能用@Service, @Controller, @Repository其中一个标注这个类的定位的时候，就不要用@Component来标注。



### **@Transactional**

@Transactional 通过这个注解可以声明事务，可以添加在类上或者方法上。

一般情况是我们会在servcie层添加了事务注解，即可开启事务。要注意的是，事务的开启只能在public 方法上。并且主要事务切面的回滚条件。正常我们配置rollbackfor exception时 ，如果在方法里捕获了异常就会导致事务切面配置的失效。







## 参考

[SpringBoot常用注解](https://www.cnblogs.com/nihaorz/p/10528121.html)