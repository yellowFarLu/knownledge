# CGLib

CGLIB是一个强大的、高性能的代码生成库。其被广泛应用于AOP框架（Spring、dynaop）中，用以提供方法拦截操作。

**CGLIB代理主要通过对字节码的操作，为对象引入代理，以控制对象的访问**。我们知道Java中有一个动态代理也是做这个事情的，那我们为什么不直接使用Java动态代理，而要使用CGLIB呢？答案是**CGLIB相比于JDK动态代理更加强大**，JDK动态代理虽然简单易用，但是其有一个致命缺陷是，只能对接口进行代理。如果要代理的类为一个普通类、没有接口，那么Java动态代理就没法使用了。

## CGLIB组成结构

![image-20191027151854602](https://tva1.sinaimg.cn/large/006y8mN6gy1g8cs8h5hy2j30pu0g8dst.jpg)

CGLIB底层使用了ASM（一个短小精悍的字节码操作框架）来操作字节码生成新的类。除了CGLIB库外，脚本语言（如Groovy何BeanShell）也使用ASM生成字节码。ASM使用类似SAX的解析器来实现高性能。我们不鼓励直接使用ASM，因为它需要对Java字节码的格式足够的了解。



## 示例

```java
public class Dao {

    public void update() {
        System.out.println("PeopleDao.update()");
    }

    public void select() {
        System.out.println("PeopleDao.select()");
    }

}
```

```java
public class DaoProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy proxy)
            throws Throwable {

        System.out.println("Before Method Invoke");

        proxy.invokeSuper(object, objects);

        System.out.println("After Method Invoke");

        return object;
    }

}
```

```java
public class CglibTest {

    public static void main(String[] args) {

        // 生成代理类
        DaoProxy daoProxy = new DaoProxy();

        Enhancer enhancer = new Enhancer();
        // 代理类的父类
        enhancer.setSuperclass(Dao.class);
        // 设置代理类的调用处理器
        enhancer.setCallback(daoProxy);

        // 获取代理类
        Dao dao = (Dao)enhancer.create();
        dao.update();
        dao.select();
    }

}
```



## 原理

使用字节码技术，动态生成被代理类的子类。





## 源码



### Enhancer

Enhancer可能是CGLIB中最常用的一个类，和Java1.3动态代理中引入的Proxy类差不多。和Proxy不同的是，**Enhancer既能够代理普通的class，也能够代理接口**

**Enhancer创建一个被代理对象的子类并且拦截所有的方法调用（包括从Object中继承的toString和hashCode方法）**

Enhancer不能够拦截final方法，例如Object.getClass()方法，这是由于Java final方法语义决定的。基于同样的道理，Enhancer也不能对fianl类进行代理操作。这也是Hibernate为什么不能持久化final class的原因。

Enhancer.setSuperclass用来设置父类型，从toString方法可以看出，使用CGLIB生成的类为**被代理类的一个子类**，形如：SampleClass$$EnhancerByCGLIB$$e3ea9b7



有些时候我们可能只想对特定的方法进行拦截，对其他的方法直接放行，不做任何操作，使用Enhancer处理这

种需求同样很简单,只需要一个CallbackFilter即可：

```java
public class TestCallbackFilterDemo {


    @Test
    public void testCallbackFilter() throws Exception{

        Enhancer enhancer = new Enhancer();

        CallbackHelper callbackHelper = new CallbackHelper(SampleClass.class, new Class[0]) {

            @Override
            protected Object getCallback(Method method) {

                if((method.getDeclaringClass() != Object.class) &&
                        (method.getReturnType() == String.class)){

                    return new FixedValue() {
                        @Override
                        public Object loadObject() throws Exception {
                            return "Hello cglibxxxxxxx";
                        }
                    };

                }else{
                    return NoOp.INSTANCE;
                }
            }
        };

        enhancer.setSuperclass(SampleClass.class);
        enhancer.setCallbackFilter(callbackHelper);
        enhancer.setCallbacks(callbackHelper.getCallbacks());
        SampleClass proxy = (SampleClass) enhancer.create();

        // 拦截
        System.out.println(proxy.test("abc"));

        // 放行
        System.out.println(proxy.hashCode());
    }

}
```





### ImmutableBean

通过名字就可以知道，不可变的Bean。ImmutableBean**允许创建一个原来对象的包装类，这个包装类是不可变的**，任何改变底层对象的包装类操作都会抛出IllegalStateException。但是我们可以通过直接操作底层对象来改变包装类对象。这有点类似于Guava中的不可变视图

```java
@Test
public void testImmutableBean() throws Exception {

  SampleBean bean = new SampleBean();
  bean.setValue("Hello world");

  //创建不可变类
  SampleBean immutableBean = (SampleBean) ImmutableBean.create(bean);
  Assert.assertEquals("Hello world",immutableBean.getValue());

  // 可以通过底层对象来进行修改
  bean.setValue("Hello world, again");
  Assert.assertEquals("Hello world, again", immutableBean.getValue());

  // 直接修改将throw exception
  immutableBean.setValue("Hello cglib");
}

public class SampleBean {

    private String value;

    public SampleBean() {
    }

    public SampleBean(String value) {
        this.value = value;
    }
    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}

```









## 参考

https://blog.csdn.net/danchu/article/details/70238002