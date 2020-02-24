# JMockit



## 概念

使用JMockit可以mock被依赖的代码，从而进行隔离测试。

比如说：

- 类级别整体mock和部分方法重写
- 实例级别整体mock和部分mock
- mock静态方法、私有变量、局部方法





## Maven依赖

Jmockit可以和junit和TestNG配合使用。需要注意的是：

- 如果使用Junit4.5以上，jmockit依赖需要在junit4之前；或者在测试类上添加注解 @RunWith(JMockit.class)。
- 如果是TestNG 6.2+ 或者 JUnit 5+， 没有位置限制

```java
<!-- 如果使用Junit4.5以上 jmockit依赖需要在junit4之前 -->
        <!-- 或者在测试类上添加注解 @RunWith(JMockit.class) -->
        <!-- 如果是TestNG 6.2+ 或者 JUnit 5+， 没有位置限制 -->
        <dependency>
            <groupId>org.jmockit</groupId>
            <artifactId>jmockit</artifactId>
            <version>1.30</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>
```





## 使用

涉及到三个类：

- 测试类：执行测试代码的类。
- CUT（Code Under Test）：被测试的类，测试此类是否能正确地工作。
- 依赖类：CUT会调用依赖类的方法。



CUT如下：

```java
@Data
public class Person {

    private String name;
    private Integer age;
    private Person friend;

    public Person(){}

    public Person(String name, Integer age, Person friend){
        this.age = age;
        this.name = name;
        this.friend = friend;
    }

    @Override
    public boolean equals(Object obj) {
        return this.name.equals(((Person)obj).getName());
    }
}


@Service
public class PersonService {

    public String showName(String name){
        System.out.println("person show name : " + name);
        return name;
    }

    public int showAge(int age) {
        System.out.println("person show age : " + age);
        return age;
    }

    public Person getDefaultPerson(){
        return new Person("miao", 3, null);
    }
}


@Service
public class CoderService {
    @Value("${coder.service.desc}")
    private String desc;

    public String showWork(String work){
        return work;
    }

    public int showSalary(int salary){
        return salary;
    }

    public String getDesc() {
        return desc;
    }

    public String getPersonName(Person person){
        return person.getName();
    }
}
```





### 基本流程

record（录制）---- replay（回放） ---- verify（验证）

```java
record : 设置将要被调用的方法和返回值。

Expections中的方法至少被调用一次，否则会出现missing invocation错误。调用次数和调用顺序不限。
StrictExpectations中方法调用的次数和顺序都必须严格执行。如果出现了在StrictExpectations中没有声明的方法，会出现unexpected invocation错误。
replay：调用（未被）录制的方法，被录制的方法调用会被JMockit拦截并重定向到record阶段设定的行为。

verify：基于行为的验证，测试CUT是否正确调用了依赖类，包括：调用了哪些方法；通过怎样的参数;调用了多少次;调用的相对顺序（VerificationsInOrder）等。可以使用times，minTimes，maxTimes来验证。
```



```java
 @Test
    public void mockProcessTest(final @Mocked PersonService target){
        //录制预期行为
        new Expectations(){
            {
                target.showName(anyString);
                result = "test1";
                target.showAge(anyInt);
                result = -1;
            }
        };

        //测试代码
        Assert.assertTrue("test1".equals(target.showName("test2")));
        Assert.assertTrue(-1 == target.showAge(12));
        Assert.assertTrue(-1 == target.showAge(12));

        //验证
        new Verifications(){
            {
                target.showName("test1");
                times = 0; //执行了0次。参数一致的才会计数
                target.showAge(12);
                times = 2; //执行了2次
            }
        };
    }

    /**
     * Expections中的方法至少被调用一次，否则会出现missing invocation错误.
     * 调用次数和调用顺序不限.
     */
    @Test
    public void mockExpectationsProcessTest(final @Mocked PersonService service){
        new Expectations(){{
            service.showAge(anyInt);
            result = -1;
        }};
        //只调用showName会报错 Missing 1 invocation
        service.showName("hahah");
        service.showAge(12);
    }
    /**
     * StrictExpectations中方法调用的次数和顺序都必须严格执行。如果出现了在StrictExpectations中没有声明的方法，会出现unexpected invocation错误。
     * 没有必要做Verifications验证。
     */
    @Test
    public void mockStrictExpectationsProcessTest(final @Mocked PersonService service){
        new StrictExpectations(){{
            service.showAge(anyInt);
            result = -1;
            service.showName(anyString);
            result = "ok";
        }};

        //1.下面只执行了一个录制方法，报错：unexpected invocation, Missing invocation
       // Assert.assertTrue(-1 == service.showAge(12));

        //2.下面与录制顺序不一致，会报错：unexpected invocation, Missing invocation
       // Assert.assertTrue("ok".equals(service.showName("test")));
       // Assert.assertTrue(-1 == service.showAge(12));

        //3.调用没有录制的方法，报错 Unexpected invocation
       // service.getDefaultPerson();

        //必须全部执行录制的方法，且顺序一致
        Assert.assertTrue(-1 == service.showAge(12));
        Assert.assertTrue("ok".equals(service.showName("test")));
    }
```



### 部分注解说明

```java
RunWith(JMockit.class): 指定单元测试的执行类为JMockit.class。
Tested: 指定被测试类，同时mock实例并注入测试类；依赖的类使用Injectable注入。
Injectable: 将对象进行mock并注入测试类。
Mocked：mock一种类型，并注入测试类。
```

Mocked与Injectable区别：

- Mocked 注入的依赖，类的所有实例都被mock，record的方法，在replay时，按照record的结果返回；没有record的方法返回默认值。
- Injectable 注入的依赖，只mock指定的实例，record的方法，在replay时，按照record的结果返回；没有record的方法返回默认值。没有mock的实例，调用其原始方法。

```java
@RunWith(JMockit.class)
public class MockTest {

    //@Mocked 修饰，所有实例都会被mock
    @Mocked
    private PersonService personService;

    // @Injectable 修饰，只mock指定的实例。
    @Injectable
    private CoderService coderService;

    @Test
    public void testInstance(){
        new Expectations(){
            {
                personService.showAge(anyInt);
                result = -1;

                personService.getDefaultPerson();
                result = new Person("me", 4, null);

                Deencapsulation.invoke(coderService, "showWork", anyString);
                result = "java";

            }
        };

        //record的方法，按照给定的结果返回
        Assert.assertTrue(-1 == personService.showAge(11));
        Assert.assertTrue("java".equals(coderService.showWork("nothing")));
        Assert.assertTrue(4 == personService.getDefaultPerson().getAge());
        //没有录制的方法，返回默认值
        Assert.assertTrue(personService.showName("testName") == null);
        Assert.assertTrue(coderService.showSalary(100) == 0);

        //Mock 所有PersonServiceImpl实例
        PersonService pservice = new PersonService();
        Assert.assertTrue(-1 == pservice.showAge(11));
        Assert.assertTrue(pservice.showName("testName") == null);

        //新生成的CoderService实例没有被mock
        CoderService cservice = new CoderService();
        Assert.assertTrue("something".equals(cservice.showWork("something")));
        Assert.assertTrue(cservice.showSalary(100) == 100);
    }

    /**
     * 可以将参数注入，与类中注入结果一致。
     * 但是不要同时在参数中注入，且在测试类中注入，会影响执行结果。
     */
    @Test
    public void testInjectObj(final @Injectable CoderService coderService){
        new Expectations(){
            {
                coderService.showWork(anyString);
                result = "ok";
            }
        };
        Assert.assertTrue("ok".equals(coderService.showWork("hello")));
        Assert.assertTrue(coderService.showSalary(100) == 0);
    }
```



### 使用示例

1. 部分mock（实例级别）
   在Expectations中传入被mock实例。 则replay的方法在Expectations中被录制时，按照record结果返回；没有被录制，则调用原有代码。
   与之对应的是Injectable 注入的实例，record的方法，在replay时按照record结果返回；没有record的方法，返回默认值。

   ```java
     /**
        * 部分mock，在Expectations中传入被mock实例。
        * replay的方法在Expectations中被录制时，按照record结果返回；
        * 没有被录制，则调用原有代码
        */
       @Test
       public void partiallyMock(){
           new Expectations(personService){
               {
                   personService.showAge(anyInt);
                   result = -1;
               }
           };
           //被录制的方法，按照record结果返回
           Assert.assertTrue(-1 == personService.showAge(11));
           //未录制的方法，调用原有代码
           Assert.assertTrue("testName".equals(personService.showName("testName")));
       }
   ```

   

2. mockUp（类级别）
   mockUp的类，被mock的方法，replay的时候都执行mock的方法；没有被mock的方法，调用原有代码。
   与之对应的事Mocked注入的类，所有record的方法按照record结果返回；没有record的方法，返回默认值。

   ```java
   /**
        * mockUp类，被mock的方法，replay的时候都执行mock的方法；
        * 没有被mock的方法，调用原有代码
        */
       @Test
       public void mockUpTest(){
           
           new MockUp<PersonService>(){
               @Mock
               public String showName(String name){
                   return "mocked";
               }
           };
   
           Assert.assertTrue("mocked".equals(new PersonService().showName("test")));
           Assert.assertTrue(1 == new PersonService().showAge(1));
       }
   ```

   

3. mock 静态方法

   ```java
   @Test
       public void testStaticMethod(){
           new Expectations(CollectionUtils.class){{
               CollectionUtils.isEmpty((Collection<?>) any);
               result = true;
           }};
           List<Integer> list = Lists.newArrayList(1,2,3);
           Assert.assertTrue(list.size() == 3);
           Assert.assertTrue(CollectionUtils.isEmpty(list));
       }
   ```

   

4. mock 私有变量

5. 局部方法

6. mock 参数匹配问题
   参数为基本类型时，若mock方法参数设置为anyXXX，则任意此类型参数都可mock成功；若mock方法参数为具体值，则实际参数 equals mock参数时，才能mock成功。
   参数为非基本类型时，mock参数不可以为any，执行报错；若mock参数为具体值，只有传递的参数 equals mock参数时，才能mock成功。

   ```java
   public class CoderServiceTest {
   
       @Tested
       private CoderService coderService;
       @Injectable
       private PersonService personService;
   
       @Test
       public void testMockCase(){
           new Expectations(coderService){{
               //mock私有变量
               Deencapsulation.setField(coderService, "desc", "coderDesc");
               //mock 方法
               Deencapsulation.invoke(coderService, "showWork", anyString);
               result = "noWork";
           }};
   
           //mock 私有变量成功
           Assert.assertTrue(coderService.getDesc().equals("coderDesc"));
           //mock 私有方法
           Assert.assertTrue(coderService.showWork("coder").equals("noWork"));
       }
       
       @Test
       public void testParamCase(){
           new Expectations(coderService){{
               //基本类型，mock参数为anyXXX
               Deencapsulation.invoke(coderService, "showWork", anyString);
               result = "mocked";
   
               //基本类型，mock参数为实际值
               Deencapsulation.invoke(coderService, "showSalary", 12);
               result = -1;
   
               //非基本类型，mock参数不可以为anyXXX，会报错 java.lang.IllegalArgumentException: Invalid null value passed as argument 0
           //    Deencapsulation.invoke(coderService, "getPersonName", (Person)any);
           //    result = "mocked";
   
               //基本类型，mock参数为实际值
               Deencapsulation.invoke(coderService, "getPersonName", new Person("me", 3, null));
               result = "mocked";
           }};
   
           //基本类型，mock参数为anyXXX, 实际参数为任意值mock成功
           Assert.assertTrue(coderService.showWork("java").equals("mocked"));
   
           //基本类型，mock参数为具体值, 实际参数 equals mock参数时，mock成功
           Assert.assertTrue(coderService.showSalary(12) == -1);
           Assert.assertTrue(coderService.showSalary(100) == 100);
   
           //基本类型，mock参数为实际值,实际参数 equals mock参数时，mock成功
           Assert.assertTrue("mocked".equals(coderService.getPersonName(new Person("me", 4, null))));
           Assert.assertFalse("mocked".equals(coderService.getPersonName(new Person("you", 3, null))));
       }
   
   }
   ```

   



### 注意事项

Tested指定的被测试类，必须是实现类，而非接口。否则不能正确实例化，报错NPE。





## 适用场景

- 被测试类所依赖的类比较复杂，难以构造，这时候可以使用mock
- 被测试类所依赖的接口还没有开发完，可以使用mock，保证被测试类的逻辑正确





## 原理

jmockit先通过Mocked注解标记需要Mock掉的类，jmockit使用ASM来修改原来的class文件，在JVM运行的时候，通过JDK6之后的动态Instrumentation特性监听类加载事件，并在目标类加载之前，用魔改后的字节码换掉真货（用mocked的实现掉包原来的代码）。

虽然Java是门静态类型语言，不过幸亏有字节码和JVM作为中间层，使得mock实现起来相对容易。









## 参考

[JMockit使用总结](https://www.cnblogs.com/shoren/p/jmokit-summary.html)

[mock适用场景](https://cloud.tencent.com/developer/article/1388155)

[Mock的实现原理](https://segmentfault.com/a/1190000003718149)

[Instrumentation原理](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/)

[Instrument原理](https://www.cnblogs.com/wade-luffy/p/6078301.html)

