- [深入理解动态代理](#1)
    - [代理模式](#1.1)
        - [静态代理](#1.1.1)
        - [动态代理](#1.1.2)
    - [Java动态代理](#1.2)
        - [类加载机制](#1.2.1)
        - [动态代理加载机制](#1.2.2)
        - [JDK](#1.2.3)
        - [Cglib](#1.2.4)
        - [Javassist](#1.2.5)
        - [ASM](#1.2.6)
        - [性能比较](#1.2.7)
    - [参考文档](#1.3)
    - [附录](#1.4)
        - [公用类](#1.4.1)
        - [静态代理 demo](#1.4.2)
        - [JDK demo](#1.4.3)
        - [JDK动态代理字节码源码](#1.4.4)
        - [Cglib demo](#1.4.5)
        - [Cglib动态代理字节码源码](#1.4.6)
        - [Javassist demo](#1.4.7)
        - [Javassist动态代理字节码源码](#1.4.8)
    <!-- TOC -->

# 深入理解动态代理

## 代理模式

在某些情况下，一个客户不想或者不能直接引用一个对
象，此时可以通过一个称之为"代理"的第三者来实现
间接引用。代理对象可以在客户端和目标对象之间起到中介的作用，并且可以通过代理对象去掉客户不能看到
的内容和服务或者添加客户需要的额外服务。

通过引入一个新的对象（如小图片和远程代理
对象）来实现对真实对象的操作或者将新的对
象作为真实对象的一个替身，这种实现机制即
为代理模式，通过引入代理对象来间接访问一
个对象，这就是代理模式的模式动机。

代理模式(Proxy Pattern) ：给某一个对象提供一个代
理，并由代理对象控制对原对象的引用。代理模式的英文叫做Proxy或Surrogate，它是一种对象结构型模式。

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxwzsi18sbj30ir0f3q42.jpg)

###  <h id="1.1.1"/>静态代理

-   程序运行前就已经存在代理类的字节码文件,代理类和委托类的关系在运行前就已经确定了;
-   优点
    -   业务类只需要关注业务逻辑本身，保证了业务类的重用性;
-   缺点
    -   代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理，静态代理在程序规模稍大时就无法胜任了;
    -   如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度;

###  动态代理

动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。

优点:

 - 一个动态代理可以服务于多种不同类型的对象。
 - 接口增加方法，动态代理无需变动

## Java动态代理

### 类加载机制

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxwxytw2u6j30j00dnwfy.jpg)

### 动态代理加载机制

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxwy05udwlj30pn0e7abx.jpg)

### JDK

关键代码

-   [**创建调用处理类**\] 实现 java.lang.reflect.InvocationHandler接口,
    实现它的 **java.lang.reflect.InvocationHandler\#invoke(Object proxy,
    Method method, Object\[\] args) throws Throwable**接口;
-   \[**创建动态代理类实例**\] 调用
    **java.lang.reflect.Proxy\#newProxyInstance(ClassLoader
    loader,Class\<?\>\[\] interfaces,InvocationHandler h) throws
    IllegalArgumentException** 方法,
    传递上面创建的InvocationHandler实例, 创建动态代理实例;
-   jdk动态代理类: **public final class \$Proxy0 extends Proxy
    implements TargetInterface**

jdk动态代理的机制特点

-   包：如果所代理的接口都是 public
    的，那么它将被定义在顶层包（即包路径为空），如果所代理的接口中有非
    public 的接口（因为接口不能被定义为 protect 或 private，所以除
    public 之外就是默认的 package
    访问级别），那么它将被定义在该接口所在包（假设代理了
    com.ibm.developerworks 包中的某非 public 接口
    A，那么新生成的代理类所在的包就是
    com.ibm.developerworks），这样设计的目的是为了最大程度的保证动态代理类不会因为包管理的问题而无法被成功定义并访问；
-   类修饰符：该代理类具有 final 和 public
    修饰符，意味着它可以被所有的类访问，但是不能被再度继承；
-   类名：格式是"\$ProxyN"，其中 N 是一个逐一递增的阿拉伯数字，代表
    Proxy 类第 N 次生成的动态代理类，值得注意的一点是，并不是每次调用
    Proxy 的静态方法创建动态代理类都会使得 N
    值增加，原因是如果对同一组接口（包括接口排列的顺序相同）试图重复创建动态代理类，它会很聪明地返回先前已经创建好的代理类的类对象，而不会再尝试去创建一个全新的代理类，这样可以节省不必要的代码重复生成，提高了代理类的创建效率;

代理类实例的特点

-   每个实例都会关联一个调用处理器对象，可以通过 Proxy 提供的静态方法
    getInvocationHandler 去获得代理类实例的调用处理器对象;
-   在代理类实例上调用其代理的接口中所声明的方法时，这些方法最终都会由调用处理器的
    invoke 方法执行，此外，值得注意的是，委托类的根类 java.lang.Object
    中有三个方法也同样会被分派到调用处理器的 invoke 方法执行，它们是
    hashCode，equals 和 toString，可能的原因有：一是因为这些方法为
    public 且非 final
    类型，能够被代理类覆盖；二是因为这些方法往往呈现出一个类的某种特征属性，具有一定的区分度，所以为了保证代理类与委托类对外的一致性，这三个方法也应该被分派到委托类执行;
-   当代理的一组接口有重复声明的方法且该方法被调用时，代理类总是从排在最前面的接口中获取方法对象并分派给调用处理器，而无论代理类实例是否正在以该接口（或继承于该接口的某子接口）的形式被外部引用，因为在代理类内部无法区分其当前的被引用类型;

被代理的一组接口的特点

-   首先，要注意不能有重复的接口，以避免动态代理类代码生成时的编译错误;
-   其次，这些接口对于类装载器必须可见，否则类装载器将无法链接它们，将会导致类定义失败;
-   再次，需被代理的所有非 public
    的接口必须在同一个包中，否则代理类生成也会失败;
-   最后，接口的数目不能超过 65535，这是 JVM 设定的限制;

异常处理方面的特点

-   从调用处理器接口声明的方法中可以看到理论上它能够抛出任何类型的异常，因为所有的异常都继承于
    Throwable
    接口，但事实是否如此呢？答案是否定的，原因是我们必须遵守一个继承原则：即子类覆盖父类或实现父接口的方法时，抛出的异常必须在原方法支持的异常列表之内。所以虽然调用处理器理论上讲能够，但实际上往往受限制，除非父接口中的方法支持抛
    Throwable 异常。那么如果在 invoke
    方法中的确产生了接口方法声明中不支持的异常，那将如何呢？放心，Java
    动态代理类已经为我们设计好了解决方法：它将会抛出
    UndeclaredThrowableException 异常。这个异常是一个 RuntimeException
    类型，所以不会引起编译错误。通过该异常的 getCause
    方法，还可以获得原来那个不受支持的异常对象，以便于错误诊断。

动态代理的优点

-   动态代理与静态代理相比较，最大的好处是接口中声明的所有方法都被转移到调用处理器一个集中的方法中处理（InvocationHandler.invoke）。这样，在接口方法数量比较多的时候，我们可以进行灵活处理，而不需要像静态代理那样每一个方法进行中转;

动态代理的不足

-   诚然，Proxy
    已经设计得非常优美，但是还是有一点点小小的遗憾之处，那就是它始终无法摆脱仅支持
    interface
    代理的桎梏，因为它的设计注定了这个遗憾。回想一下那些动态生成的代理类的继承关系图，它们已经注定有一个共同的父类叫
    Proxy。Java 的继承机制注定了这些动态代理类们无法实现对 class
    的动态代理，原因是多继承在 Java 中本质上就行不通。

### Cglib

cglib是针对类来实现代理的，基于ASM的包装，他的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对final修饰的类进行代理。

cglib 创建某个类A的动态代理类的模式是：

1.  查找A上的所有非final 的public类型的方法定义；

2.  将这些方法的定义转换成字节码；

3.  将组成的字节码转换成相应的代理的class对象；

4.  实现 MethodInterceptor接口，用来处理
    对代理类上所有方法的请求（这个接口和JDK动态代理InvocationHandler的功能和角色是一样的）

差异

-   JDK动态代理
    -   其代理的对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理。
    -   通过反射来实现对目标方法的调用，效率低一些；
-   CGLIG动态代理
    -   无法通知（advice）final方法，因为它不能被覆写；
    -   真实对象的引用，可以直接调用，效率更高；

### <h id="1.2.5"/>Javassist

javassist是[jboss](http://baike.baidu.com/view/309533.htm)的一个子项目，其主要的优点，在于简单，而且快速。直接使用java编码的形式，而不需要了解[虚拟机](http://baike.baidu.com/view/1132.htm)指令，就能动态改变类的结构，或者动态生成类。

使用Javassist有两种方式来实现动态代理，一种是使用代理工厂创建,和普通的JDK动态代理和CGLIB类似,另一种则可以使用字节码技术创建。

### <h id="1.2.6"/>ASM

ASM 是一个 Java
字节码操控框架。它能够以二进制形式修改已有类或者动态生成类。ASM
可以直接产生二进制 class 文件，也可以在类被加载入 Java
虚拟机之前动态改变类行为。ASM
从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。

**不过ASM在创建class字节码的过程中，操纵的级别是底层JVM的汇编指令级别，这要求ASM使用者要对class组织结构和JVM汇编指令有一定的了解。**

### <h id="1.2.7"/>性能比较

ASM \> Javassist Bytecode \> Cglib \> JDK \> Javassist ProxyFactory

-   差异原因
    -   各方案生成的字节码不一样， 
        像JDK和CGLIB都考虑了很多因素，以及继承或包装了自己的一些类， 
        所以生成的字节码非常大，而我们很多时候用不上这些， 而手工生成的字节码非常小，所以速度快。





## Spring Aop

spring开启动态代理的配置

```xml
<aop:aspectj-autoproxy/>
```

通过aop命名空间的<aop:aspectj-autoproxy />声明自动为spring容器中那些配置@aspectJ切面的bean创建代理，织入切面

有一个proxy-target-class属性，默认为false

- 当配为<aop:aspectj-autoproxy  poxy-target-class="true"/>时，表示使用CGLib动态代理技术织入增强。
- 当配为<aop:aspectj-autoproxy  poxy-target-class="false"/>时，表示使用jdk动态代理织入增强
- 不过即使proxy-target-class设置为false，如果目标类没有声明接口，则spring将自动使用CGLib动态代理。



### 排除其他类

```xml
<aop:aspect id="logMonitor" ref="logAspect" order="1">
            <aop:pointcut id="monitor"
                          expression="execution(* com.facishare.eservice.cases.provider.service.impl..*.*(..))
                          and !execution(* com.facishare.eservice.cases.provider.service.impl.MultipleVersionsServiceImpl.canExecuteByVersion(..))" />
            <aop:around pointcut-ref="monitor" method="around"/>
        </aop:aspect>
```

如图，排除了，com.facishare.eservice.cases.provider.service.impl.MultipleVersionsServiceImpl.canExecuteByVersion接口，该接口将不执行aop。



### 控制aop执行顺序

使用order参数，如:

```xml
<aop:config>
        <aop:aspect id="logMonitor" ref="logAspect1" order="1">
            <aop:pointcut id="monitor"
                          expression="execution(* com.huang.yuan.dubbo.service.impl.DemoServiceImpl.test(..))" />
            <aop:around pointcut-ref="monitor" method="around"/>
        </aop:aspect>
    </aop:config>

    <aop:config>
        <aop:aspect id="logMonitor" ref="logAspect2" order="2">
            <aop:pointcut id="monitor"
                          expression="execution(* com.huang.yuan.dubbo.service.impl.DemoServiceImpl.test(..))" />
            <aop:around pointcut-ref="monitor" method="around"/>
        </aop:aspect>
    </aop:config>

    <aop:config>
        <aop:aspect id="logMonitor" ref="logAspect3" order="3">
            <aop:pointcut id="monitor"
                          expression="execution(* com.huang.yuan.dubbo.service.impl.DemoServiceImpl.test(..))" />
            <aop:around pointcut-ref="monitor" method="around"/>
        </aop:aspect>
    </aop:config>
```

配置了3个aop，程序执行流程如：

![Snip20190414_2](https://ws2.sinaimg.cn/large/006tNc79gy1g21ysu4pr3j31b20twwk3.jpg)



### execution表达式覆盖

```
<aop:pointcut id="monitor"  
```

如果存在id相同的monitor，则会覆盖，即以最后一个monitor为准。









## 参考文档

-   [图解设计模式：代理模式](https://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/proxy.html)
-   [动态代理方案性能对比](http://javatar.iteye.com/blog/814426)
-   [Java动态代理机制详解（JDK
    和CGLIB，Javassist，ASM）](https://blog.csdn.net/luanlouis/article/details/24589193)

## <h id="1.4"/>附录

### <h id="1.4.1"/>公用类

``` {.java}
package com.github.archerda.designpattern.proxy;

/**
 * @author archerda
 * @date 2018/7/9
 */
public interface TargetInterface {
    void sayHi();
}
```

``` {.java}
package com.github.archerda.designpattern.proxy;

/**
 * @author archerda
 * @date 2018/04/24
 */
public class TargetClass {

    public void sayHi() {
        System.out.println("Hi");
    }
}
```

### <h id="1.4.2"/>静态代理 demo

``` {.java}
package com.github.archerda.designpattern.proxy;

/**
 * @author archerda
 */
public class StaticMain {

    public static void main(String[] args) {
        TargetInterface targetInterface = new TargetInterfaceImpl();

        TargetInterface target = new ProxyClass(targetInterface);

        // Client
        target.sayHi();
    }

    private static class TargetInterfaceImpl implements TargetInterface {

        @Override
        public void sayHi() {
            System.out.println("Hi");
        }
    }

    public static class ProxyClass implements TargetInterface {

        // 存储真实对象
        private TargetInterface targetInterface;

        public ProxyClass(TargetInterface targetInterface) {
            this.targetInterface = targetInterface;
        }

        @Override
        public void sayHi() {
            System.out.println("Invoke start.");
            targetInterface.sayHi();
            System.out.println("Invoke end.");
        }
    }
}
```

### <h id="1.4.3"/>JDK demo

``` {.java}
package com.github.archerda.designpattern.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * JDK动态代理Demo；
 *
 * @author archerda
 * @date 2018/01/08
 */
public class JdkMain {

    public static void main(String[] args) {

        /*
        生成$Proxy0的class文件
         */
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

        TargetInterface targetInterface = new TargetInterfaceImpl();

        /*
        生成一个代理类，这个代理类签名如下：public final class $Proxy0 extends Proxy implements TargetInterface
        它继承了 Proxy 类（这也是JDK代理不能代理类的原因），并实现了 TargetInterface 接口；
        并且这个类是 final 的，不能被继承；

        类名：$Proxy0；（$Proxy + 自增数字）
        */
        TargetInterface proxy = (TargetInterface) Proxy.newProxyInstance(targetInterface.getClass().getClassLoader(),
                targetInterface.getClass().getInterfaces(), new Handler(targetInterface));

        /*
        调用ProxyHolder的sayHi方法，JVM会把这个方法转交给 InvocationHandler 的 invoke 方法；
        实际的调用方法如下：
        public final void sayHi() throws  {
            try {
                super.h.invoke(this, m3, (Object[])null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }
         */
        proxy.sayHi();

    }

    private static class TargetInterfaceImpl implements TargetInterface {

        @Override
        public void sayHi() {
            System.out.println("Hi");
        }
    }

    private static class Handler implements InvocationHandler {

        /**
         * JDK的代理，Handler要持有一个实例的引用；
         */
        private TargetInterface targetInterface;

        public Handler(TargetInterface targetInterface) {
            this.targetInterface = targetInterface;
        }

        /**
         * @param proxy 代理对象实例，比如"@Proxy0"
         * @param method 原始方法（被拦截的方法），比如"public abstract void com.github.archerda.designpattern.proxy.TargetInterface.sayHi()"
         * @param args 参数集（被拦截方法的参数）；
         * @return Object
         * @throws Throwable 异常；
         */
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            System.out.println("Invoke start.");

            // 调用 ProxyInstance 的方法；
            // 注意，这里 invoke 方法的 第一个参数，必须是ProxyInstance，如果是ProxyHolder，将会出现死循环；
            // method.invoke(proxy, args); 死循环，因为JVM会一直委托给该方法；
            method.invoke(targetInterface, args);

            System.out.println("Invoke end.");

            return null;
        }
    }

}
```

### <h id="1.4.4"/>JDK动态代理字节码源码

``` {.java}
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.sun.proxy;

import com.github.archerda.designpattern.proxy.TargetInterface;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements TargetInterface {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHi() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.github.archerda.designpattern.proxy.TargetInterface").getMethod("sayHi");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

### <h id="1.4.5"/>Cglib demo

``` {.java}
package com.github.archerda.designpattern.proxy;


import net.sf.cglib.core.DebuggingClassWriter;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * CGLIB动态代理Demo；
 *
 * @author archerda
 * @date 2018/04/24
 */
public class CglibMain {

    public static void main(String[] args) {

        /*
        生成代理类的class文件；
         */
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/Archerda/Desktop/java-example/com/sun/proxy");

        /*
        生成Cglib的增强器；
         */
        Enhancer enhancer = new Enhancer();

        /*
        给增强器设置父类；
         */
        enhancer.setSuperclass(TargetClass.class);

        /*
        给增强器设置会拦截器；
         */
        enhancer.setCallback(new Interceptor());

        /*
        使用增强器创建代理类；
         */
        TargetClass target = (TargetClass) enhancer.create();

        /*
        调用代理类方法；
         */
        target.sayHi();
    }

    private static class Interceptor implements MethodInterceptor {

        /**
         *
         * @param o 代理对象，比如：TargetClass$EnhancerByCGLIB$72cfe419
         * @param method 原始方法（要被拦截的方法）；比如：public void com.github.archerda.designpattern.proxy.TargetClass.sayHi()
         * @param objects 参数集；
         * @param methodProxy 代理方法（要触发父类的方法对象）；
         * @return Object
         * @throws Throwable 异常
         */
        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("Invoke start");

            /*
            这里不能使用 method#invoke 或者 methodProxy#invoke，否则可能会导致死循环；
             */
            methodProxy.invokeSuper(o, objects);
            System.out.println("Invoke end.");
            return null;
        }
    }
}
```

### <h id="1.4.6"/>Cglib动态代理字节码源码

``` {.java}
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.github.archerda.designpattern.proxy;

import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class TargetClass$$EnhancerByCGLIB$$92457a4a extends TargetClass implements Factory {
    private boolean CGLIB$BOUND;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static final Method CGLIB$sayHi$0$Method;
    private static final MethodProxy CGLIB$sayHi$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$finalize$1$Method;
    private static final MethodProxy CGLIB$finalize$1$Proxy;
    private static final Method CGLIB$equals$2$Method;
    private static final MethodProxy CGLIB$equals$2$Proxy;
    private static final Method CGLIB$toString$3$Method;
    private static final MethodProxy CGLIB$toString$3$Proxy;
    private static final Method CGLIB$hashCode$4$Method;
    private static final MethodProxy CGLIB$hashCode$4$Proxy;
    private static final Method CGLIB$clone$5$Method;
    private static final MethodProxy CGLIB$clone$5$Proxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("com.github.archerda.designpattern.proxy.TargetClass$$EnhancerByCGLIB$$92457a4a");
        Class var1;
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"finalize", "()V", "equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$finalize$1$Method = var10000[0];
        CGLIB$finalize$1$Proxy = MethodProxy.create(var1, var0, "()V", "finalize", "CGLIB$finalize$1");
        CGLIB$equals$2$Method = var10000[1];
        CGLIB$equals$2$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$2");
        CGLIB$toString$3$Method = var10000[2];
        CGLIB$toString$3$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$3");
        CGLIB$hashCode$4$Method = var10000[3];
        CGLIB$hashCode$4$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$4");
        CGLIB$clone$5$Method = var10000[4];
        CGLIB$clone$5$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$5");
        CGLIB$sayHi$0$Method = ReflectUtils.findMethods(new String[]{"sayHi", "()V"}, (var1 = Class.forName("com.github.archerda.designpattern.proxy.TargetClass")).getDeclaredMethods())[0];
        CGLIB$sayHi$0$Proxy = MethodProxy.create(var1, var0, "()V", "sayHi", "CGLIB$sayHi$0");
    }

    final void CGLIB$sayHi$0() {
        super.sayHi();
    }

    public final void sayHi() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$sayHi$0$Method, CGLIB$emptyArgs, CGLIB$sayHi$0$Proxy);
        } else {
            super.sayHi();
        }
    }

    final void CGLIB$finalize$1() throws Throwable {
        super.finalize();
    }

    protected final void finalize() throws Throwable {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$finalize$1$Method, CGLIB$emptyArgs, CGLIB$finalize$1$Proxy);
        } else {
            super.finalize();
        }
    }

    final boolean CGLIB$equals$2(Object var1) {
        return super.equals(var1);
    }

    public final boolean equals(Object var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            Object var2 = var10000.intercept(this, CGLIB$equals$2$Method, new Object[]{var1}, CGLIB$equals$2$Proxy);
            return var2 == null ? false : (Boolean)var2;
        } else {
            return super.equals(var1);
        }
    }

    final String CGLIB$toString$3() {
        return super.toString();
    }

    public final String toString() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? (String)var10000.intercept(this, CGLIB$toString$3$Method, CGLIB$emptyArgs, CGLIB$toString$3$Proxy) : super.toString();
    }

    final int CGLIB$hashCode$4() {
        return super.hashCode();
    }

    public final int hashCode() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            Object var1 = var10000.intercept(this, CGLIB$hashCode$4$Method, CGLIB$emptyArgs, CGLIB$hashCode$4$Proxy);
            return var1 == null ? 0 : ((Number)var1).intValue();
        } else {
            return super.hashCode();
        }
    }

    final Object CGLIB$clone$5() throws CloneNotSupportedException {
        return super.clone();
    }

    protected final Object clone() throws CloneNotSupportedException {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? var10000.intercept(this, CGLIB$clone$5$Method, CGLIB$emptyArgs, CGLIB$clone$5$Proxy) : super.clone();
    }

    public static MethodProxy CGLIB$findMethodProxy(Signature var0) {
        String var10000 = var0.toString();
        switch(var10000.hashCode()) {
        case -2012941911:
            if (var10000.equals("sayHi()V")) {
                return CGLIB$sayHi$0$Proxy;
            }
            break;
        case -1574182249:
            if (var10000.equals("finalize()V")) {
                return CGLIB$finalize$1$Proxy;
            }
            break;
        case -508378822:
            if (var10000.equals("clone()Ljava/lang/Object;")) {
                return CGLIB$clone$5$Proxy;
            }
            break;
        case 1826985398:
            if (var10000.equals("equals(Ljava/lang/Object;)Z")) {
                return CGLIB$equals$2$Proxy;
            }
            break;
        case 1913648695:
            if (var10000.equals("toString()Ljava/lang/String;")) {
                return CGLIB$toString$3$Proxy;
            }
            break;
        case 1984935277:
            if (var10000.equals("hashCode()I")) {
                return CGLIB$hashCode$4$Proxy;
            }
        }

        return null;
    }

    public TargetClass$$EnhancerByCGLIB$$92457a4a() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
        CGLIB$THREAD_CALLBACKS.set(var0);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
        CGLIB$STATIC_CALLBACKS = var0;
    }

    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        TargetClass$$EnhancerByCGLIB$$92457a4a var1 = (TargetClass$$EnhancerByCGLIB$$92457a4a)var0;
        if (!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if (var10000 == null) {
                var10000 = CGLIB$STATIC_CALLBACKS;
                if (var10000 == null) {
                    return;
                }
            }

            var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
        }

    }

    public Object newInstance(Callback[] var1) {
        CGLIB$SET_THREAD_CALLBACKS(var1);
        TargetClass$$EnhancerByCGLIB$$92457a4a var10000 = new TargetClass$$EnhancerByCGLIB$$92457a4a();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Callback var1) {
        CGLIB$SET_THREAD_CALLBACKS(new Callback[]{var1});
        TargetClass$$EnhancerByCGLIB$$92457a4a var10000 = new TargetClass$$EnhancerByCGLIB$$92457a4a();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Class[] var1, Object[] var2, Callback[] var3) {
        CGLIB$SET_THREAD_CALLBACKS(var3);
        TargetClass$$EnhancerByCGLIB$$92457a4a var10000 = new TargetClass$$EnhancerByCGLIB$$92457a4a;
        switch(var1.length) {
        case 0:
            var10000.<init>();
            CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
            return var10000;
        default:
            throw new IllegalArgumentException("Constructor not found");
        }
    }

    public Callback getCallback(int var1) {
        CGLIB$BIND_CALLBACKS(this);
        MethodInterceptor var10000;
        switch(var1) {
        case 0:
            var10000 = this.CGLIB$CALLBACK_0;
            break;
        default:
            var10000 = null;
        }

        return var10000;
    }

    public void setCallback(int var1, Callback var2) {
        switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (MethodInterceptor)var2;
        default:
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (MethodInterceptor)var1[0];
    }

    static {
        CGLIB$STATICHOOK1();
    }
}
```

``` {.java}
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.github.archerda.designpattern.proxy;

import com.github.archerda.designpattern.proxy.TargetClass..EnhancerByCGLIB..92457a4a;
import java.lang.reflect.InvocationTargetException;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.reflect.FastClass;

public class TargetClass$$EnhancerByCGLIB$$92457a4a$$FastClassByCGLIB$$b8e5bdfc extends FastClass {
    public TargetClass$$EnhancerByCGLIB$$92457a4a$$FastClassByCGLIB$$b8e5bdfc(Class var1) {
        super(var1);
    }

    public int getIndex(Signature var1) {
        String var10000 = var1.toString();
        switch(var10000.hashCode()) {
        case -2055565910:
            if (var10000.equals("CGLIB$SET_THREAD_CALLBACKS([Lnet/sf/cglib/proxy/Callback;)V")) {
                return 10;
            }
            break;
        case -2012941911:
            if (var10000.equals("sayHi()V")) {
                return 6;
            }
            break;
        case -1937024608:
            if (var10000.equals("CGLIB$sayHi$0()V")) {
                return 14;
            }
            break;
        case -1725733088:
            if (var10000.equals("getClass()Ljava/lang/Class;")) {
                return 24;
            }
            break;
        case -1457535688:
            if (var10000.equals("CGLIB$STATICHOOK1()V")) {
                return 19;
            }
            break;
        case -1411812934:
            if (var10000.equals("CGLIB$hashCode$4()I")) {
                return 18;
            }
            break;
        case -1026001249:
            if (var10000.equals("wait(JI)V")) {
                return 21;
            }
            break;
        case -894172689:
            if (var10000.equals("newInstance(Lnet/sf/cglib/proxy/Callback;)Ljava/lang/Object;")) {
                return 4;
            }
            break;
        case -623122092:
            if (var10000.equals("CGLIB$findMethodProxy(Lnet/sf/cglib/core/Signature;)Lnet/sf/cglib/proxy/MethodProxy;")) {
                return 13;
            }
            break;
        case -419626537:
            if (var10000.equals("setCallbacks([Lnet/sf/cglib/proxy/Callback;)V")) {
                return 8;
            }
            break;
        case 243996900:
            if (var10000.equals("wait(J)V")) {
                return 22;
            }
            break;
        case 374345669:
            if (var10000.equals("CGLIB$equals$2(Ljava/lang/Object;)Z")) {
                return 16;
            }
            break;
        case 560567118:
            if (var10000.equals("setCallback(ILnet/sf/cglib/proxy/Callback;)V")) {
                return 7;
            }
            break;
        case 811063227:
            if (var10000.equals("newInstance([Ljava/lang/Class;[Ljava/lang/Object;[Lnet/sf/cglib/proxy/Callback;)Ljava/lang/Object;")) {
                return 3;
            }
            break;
        case 946854621:
            if (var10000.equals("notifyAll()V")) {
                return 26;
            }
            break;
        case 973717575:
            if (var10000.equals("getCallbacks()[Lnet/sf/cglib/proxy/Callback;")) {
                return 12;
            }
            break;
        case 1116248544:
            if (var10000.equals("wait()V")) {
                return 23;
            }
            break;
        case 1221173700:
            if (var10000.equals("newInstance([Lnet/sf/cglib/proxy/Callback;)Ljava/lang/Object;")) {
                return 5;
            }
            break;
        case 1230699260:
            if (var10000.equals("getCallback(I)Lnet/sf/cglib/proxy/Callback;")) {
                return 11;
            }
            break;
        case 1365077639:
            if (var10000.equals("CGLIB$finalize$1()V")) {
                return 15;
            }
            break;
        case 1517819849:
            if (var10000.equals("CGLIB$toString$3()Ljava/lang/String;")) {
                return 17;
            }
            break;
        case 1584330438:
            if (var10000.equals("CGLIB$SET_STATIC_CALLBACKS([Lnet/sf/cglib/proxy/Callback;)V")) {
                return 9;
            }
            break;
        case 1826985398:
            if (var10000.equals("equals(Ljava/lang/Object;)Z")) {
                return 0;
            }
            break;
        case 1902039948:
            if (var10000.equals("notify()V")) {
                return 25;
            }
            break;
        case 1913648695:
            if (var10000.equals("toString()Ljava/lang/String;")) {
                return 1;
            }
            break;
        case 1984935277:
            if (var10000.equals("hashCode()I")) {
                return 2;
            }
            break;
        case 2011844968:
            if (var10000.equals("CGLIB$clone$5()Ljava/lang/Object;")) {
                return 20;
            }
        }

        return -1;
    }

    public int getIndex(String var1, Class[] var2) {
        switch(var1.hashCode()) {
        case -2083498450:
            if (var1.equals("CGLIB$finalize$1")) {
                switch(var2.length) {
                case 0:
                    return 15;
                }
            }
            break;
        case -1776922004:
            if (var1.equals("toString")) {
                switch(var2.length) {
                case 0:
                    return 1;
                }
            }
            break;
        case -1334646347:
            if (var1.equals("CGLIB$sayHi$0")) {
                switch(var2.length) {
                case 0:
                    return 14;
                }
            }
            break;
        case -1295482945:
            if (var1.equals("equals")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("java.lang.Object")) {
                        return 0;
                    }
                }
            }
            break;
        case -1053468136:
            if (var1.equals("getCallbacks")) {
                switch(var2.length) {
                case 0:
                    return 12;
                }
            }
            break;
        case -1039689911:
            if (var1.equals("notify")) {
                switch(var2.length) {
                case 0:
                    return 25;
                }
            }
            break;
        case -124978608:
            if (var1.equals("CGLIB$equals$2")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("java.lang.Object")) {
                        return 16;
                    }
                }
            }
            break;
        case -60403779:
            if (var1.equals("CGLIB$SET_STATIC_CALLBACKS")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("[Lnet.sf.cglib.proxy.Callback;")) {
                        return 9;
                    }
                }
            }
            break;
        case -29025554:
            if (var1.equals("CGLIB$hashCode$4")) {
                switch(var2.length) {
                case 0:
                    return 18;
                }
            }
            break;
        case 3641717:
            if (var1.equals("wait")) {
                switch(var2.length) {
                case 0:
                    return 23;
                case 1:
                    if (var2[0].getName().equals("long")) {
                        return 22;
                    }
                    break;
                case 2:
                    if (var2[0].getName().equals("long") && var2[1].getName().equals("int")) {
                        return 21;
                    }
                }
            }
            break;
        case 85179481:
            if (var1.equals("CGLIB$SET_THREAD_CALLBACKS")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("[Lnet.sf.cglib.proxy.Callback;")) {
                        return 10;
                    }
                }
            }
            break;
        case 109213260:
            if (var1.equals("sayHi")) {
                switch(var2.length) {
                case 0:
                    return 6;
                }
            }
            break;
        case 147696667:
            if (var1.equals("hashCode")) {
                switch(var2.length) {
                case 0:
                    return 2;
                }
            }
            break;
        case 161998109:
            if (var1.equals("CGLIB$STATICHOOK1")) {
                switch(var2.length) {
                case 0:
                    return 19;
                }
            }
            break;
        case 495524492:
            if (var1.equals("setCallbacks")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("[Lnet.sf.cglib.proxy.Callback;")) {
                        return 8;
                    }
                }
            }
            break;
        case 1154623345:
            if (var1.equals("CGLIB$findMethodProxy")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("net.sf.cglib.core.Signature")) {
                        return 13;
                    }
                }
            }
            break;
        case 1543336190:
            if (var1.equals("CGLIB$toString$3")) {
                switch(var2.length) {
                case 0:
                    return 17;
                }
            }
            break;
        case 1811874389:
            if (var1.equals("newInstance")) {
                switch(var2.length) {
                case 1:
                    String var10001 = var2[0].getName();
                    switch(var10001.hashCode()) {
                    case -845341380:
                        if (var10001.equals("net.sf.cglib.proxy.Callback")) {
                            return 4;
                        }
                        break;
                    case 1730110032:
                        if (var10001.equals("[Lnet.sf.cglib.proxy.Callback;")) {
                            return 5;
                        }
                    }
                case 2:
                default:
                    break;
                case 3:
                    if (var2[0].getName().equals("[Ljava.lang.Class;") && var2[1].getName().equals("[Ljava.lang.Object;") && var2[2].getName().equals("[Lnet.sf.cglib.proxy.Callback;")) {
                        return 3;
                    }
                }
            }
            break;
        case 1817099975:
            if (var1.equals("setCallback")) {
                switch(var2.length) {
                case 2:
                    if (var2[0].getName().equals("int") && var2[1].getName().equals("net.sf.cglib.proxy.Callback")) {
                        return 7;
                    }
                }
            }
            break;
        case 1902066072:
            if (var1.equals("notifyAll")) {
                switch(var2.length) {
                case 0:
                    return 26;
                }
            }
            break;
        case 1905679803:
            if (var1.equals("getCallback")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("int")) {
                        return 11;
                    }
                }
            }
            break;
        case 1950568386:
            if (var1.equals("getClass")) {
                switch(var2.length) {
                case 0:
                    return 24;
                }
            }
            break;
        case 1951977611:
            if (var1.equals("CGLIB$clone$5")) {
                switch(var2.length) {
                case 0:
                    return 20;
                }
            }
        }

        return -1;
    }

    public int getIndex(Class[] var1) {
        switch(var1.length) {
        case 0:
            return 0;
        default:
            return -1;
        }
    }

    public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
        92457a4a var10000 = (92457a4a)var2;
        int var10001 = var1;

        try {
            switch(var10001) {
            case 0:
                return new Boolean(var10000.equals(var3[0]));
            case 1:
                return var10000.toString();
            case 2:
                return new Integer(var10000.hashCode());
            case 3:
                return var10000.newInstance((Class[])var3[0], (Object[])var3[1], (Callback[])var3[2]);
            case 4:
                return var10000.newInstance((Callback)var3[0]);
            case 5:
                return var10000.newInstance((Callback[])var3[0]);
            case 6:
                var10000.sayHi();
                return null;
            case 7:
                var10000.setCallback(((Number)var3[0]).intValue(), (Callback)var3[1]);
                return null;
            case 8:
                var10000.setCallbacks((Callback[])var3[0]);
                return null;
            case 9:
                92457a4a.CGLIB$SET_STATIC_CALLBACKS((Callback[])var3[0]);
                return null;
            case 10:
                92457a4a.CGLIB$SET_THREAD_CALLBACKS((Callback[])var3[0]);
                return null;
            case 11:
                return var10000.getCallback(((Number)var3[0]).intValue());
            case 12:
                return var10000.getCallbacks();
            case 13:
                return 92457a4a.CGLIB$findMethodProxy((Signature)var3[0]);
            case 14:
                var10000.CGLIB$sayHi$0();
                return null;
            case 15:
                var10000.CGLIB$finalize$1();
                return null;
            case 16:
                return new Boolean(var10000.CGLIB$equals$2(var3[0]));
            case 17:
                return var10000.CGLIB$toString$3();
            case 18:
                return new Integer(var10000.CGLIB$hashCode$4());
            case 19:
                92457a4a.CGLIB$STATICHOOK1();
                return null;
            case 20:
                return var10000.CGLIB$clone$5();
            case 21:
                var10000.wait(((Number)var3[0]).longValue(), ((Number)var3[1]).intValue());
                return null;
            case 22:
                var10000.wait(((Number)var3[0]).longValue());
                return null;
            case 23:
                var10000.wait();
                return null;
            case 24:
                return var10000.getClass();
            case 25:
                var10000.notify();
                return null;
            case 26:
                var10000.notifyAll();
                return null;
            }
        } catch (Throwable var4) {
            throw new InvocationTargetException(var4);
        }

        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }

    public Object newInstance(int var1, Object[] var2) throws InvocationTargetException {
        92457a4a var10000 = new 92457a4a;
        92457a4a var10001 = var10000;
        int var10002 = var1;

        try {
            switch(var10002) {
            case 0:
                var10001.<init>();
                return var10000;
            }
        } catch (Throwable var3) {
            throw new InvocationTargetException(var3);
        }

        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }

    public int getMaxIndex() {
        return 26;
    }
}
```

``` {.java}
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.github.archerda.designpattern.proxy;

import java.lang.reflect.InvocationTargetException;
import net.sf.cglib.core.Signature;
import net.sf.cglib.reflect.FastClass;

public class TargetClass$$FastClassByCGLIB$$4e2d847b extends FastClass {
    public TargetClass$$FastClassByCGLIB$$4e2d847b(Class var1) {
        super(var1);
    }

    public int getIndex(Signature var1) {
        String var10000 = var1.toString();
        switch(var10000.hashCode()) {
        case -2012941911:
            if (var10000.equals("sayHi()V")) {
                return 0;
            }
            break;
        case -1725733088:
            if (var10000.equals("getClass()Ljava/lang/Class;")) {
                return 7;
            }
            break;
        case -1026001249:
            if (var10000.equals("wait(JI)V")) {
                return 1;
            }
            break;
        case 243996900:
            if (var10000.equals("wait(J)V")) {
                return 2;
            }
            break;
        case 946854621:
            if (var10000.equals("notifyAll()V")) {
                return 9;
            }
            break;
        case 1116248544:
            if (var10000.equals("wait()V")) {
                return 3;
            }
            break;
        case 1826985398:
            if (var10000.equals("equals(Ljava/lang/Object;)Z")) {
                return 4;
            }
            break;
        case 1902039948:
            if (var10000.equals("notify()V")) {
                return 8;
            }
            break;
        case 1913648695:
            if (var10000.equals("toString()Ljava/lang/String;")) {
                return 5;
            }
            break;
        case 1984935277:
            if (var10000.equals("hashCode()I")) {
                return 6;
            }
        }

        return -1;
    }

    public int getIndex(String var1, Class[] var2) {
        switch(var1.hashCode()) {
        case -1776922004:
            if (var1.equals("toString")) {
                switch(var2.length) {
                case 0:
                    return 5;
                }
            }
            break;
        case -1295482945:
            if (var1.equals("equals")) {
                switch(var2.length) {
                case 1:
                    if (var2[0].getName().equals("java.lang.Object")) {
                        return 4;
                    }
                }
            }
            break;
        case -1039689911:
            if (var1.equals("notify")) {
                switch(var2.length) {
                case 0:
                    return 8;
                }
            }
            break;
        case 3641717:
            if (var1.equals("wait")) {
                switch(var2.length) {
                case 0:
                    return 3;
                case 1:
                    if (var2[0].getName().equals("long")) {
                        return 2;
                    }
                    break;
                case 2:
                    if (var2[0].getName().equals("long") && var2[1].getName().equals("int")) {
                        return 1;
                    }
                }
            }
            break;
        case 109213260:
            if (var1.equals("sayHi")) {
                switch(var2.length) {
                case 0:
                    return 0;
                }
            }
            break;
        case 147696667:
            if (var1.equals("hashCode")) {
                switch(var2.length) {
                case 0:
                    return 6;
                }
            }
            break;
        case 1902066072:
            if (var1.equals("notifyAll")) {
                switch(var2.length) {
                case 0:
                    return 9;
                }
            }
            break;
        case 1950568386:
            if (var1.equals("getClass")) {
                switch(var2.length) {
                case 0:
                    return 7;
                }
            }
        }

        return -1;
    }

    public int getIndex(Class[] var1) {
        switch(var1.length) {
        case 0:
            return 0;
        default:
            return -1;
        }
    }

    public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
        TargetClass var10000 = (TargetClass)var2;
        int var10001 = var1;

        try {
            switch(var10001) {
            case 0:
                var10000.sayHi();
                return null;
            case 1:
                var10000.wait(((Number)var3[0]).longValue(), ((Number)var3[1]).intValue());
                return null;
            case 2:
                var10000.wait(((Number)var3[0]).longValue());
                return null;
            case 3:
                var10000.wait();
                return null;
            case 4:
                return new Boolean(var10000.equals(var3[0]));
            case 5:
                return var10000.toString();
            case 6:
                return new Integer(var10000.hashCode());
            case 7:
                return var10000.getClass();
            case 8:
                var10000.notify();
                return null;
            case 9:
                var10000.notifyAll();
                return null;
            }
        } catch (Throwable var4) {
            throw new InvocationTargetException(var4);
        }

        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }

    public Object newInstance(int var1, Object[] var2) throws InvocationTargetException {
        TargetClass var10000 = new TargetClass;
        TargetClass var10001 = var10000;
        int var10002 = var1;

        try {
            switch(var10002) {
            case 0:
                var10001.<init>();
                return var10000;
            }
        } catch (Throwable var3) {
            throw new InvocationTargetException(var3);
        }

        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }

    public int getMaxIndex() {
        return 9;
    }
}
```

### <h id="1.4.7"/>Javassist demo

``` {.java}
package com.github.archerda.designpattern.proxy;

import javassist.*;

import java.io.File;
import java.io.FileOutputStream;

/**
 * Javassist动态代理: 采用 字节码 技术创建;
 *
 * @author archerda
 */
public class JavassistByteCodeMain {

    public static void main(String[] args) throws Exception {

        /*
        创建类池
         */
        ClassPool classPool = ClassPool.getDefault();

        String className = TargetClass.class.getName();
        CtClass ctClass = classPool.makeClass(className + "JavassitProxy");

        /*
        添加接口,可选
         */
        ctClass.addInterface(classPool.get(TargetInterface.class.getName()));

        /*
        添加超类
         */
        // ctClass.setSuperclass(classPool.get(TargetClass.class.getName()));

        /*
         添加默认构造函数
         */
        // ctClass.addConstructor(new CtConstructor(new CtClass[]{}, ctClass));

        /*
        添加屬性
         */
        CtField enameField = new CtField(classPool.getCtClass(className), "targetClass", ctClass);
        enameField.setModifiers(Modifier.PRIVATE);
        ctClass.addField(enameField);

        /*
        添加构造函数
         */
        CtConstructor ctConstructor = new CtConstructor(new CtClass[]{}, ctClass);
        /*
        为构造函数设置函数体
         */
        ctConstructor.setBody("{\n" + "targetClass = new " + className + "();" + "\n}");
        /*
        把构造函数添加到新的类中
         */
        ctClass.addConstructor(ctConstructor);

        /*
        添加方法,里面进行动态代理logic
         */
        CtMethod ctMethod = new CtMethod(CtClass.voidType, "sayHi", new CtClass[]{}, ctClass);
        /*
        为自定义方法设置修饰符
         */
        ctMethod.setModifiers(Modifier.PUBLIC);
        /*
        为自定义方法设置函数体
         */
        String buffer2 = "{\n" +
                "targetClass.sayHi();" +
                "\n}";
        ctMethod.setBody(buffer2);
        ctClass.addMethod(ctMethod);

        /*
        把生成的class文件写入文件
         */
        byte[] byteArr = ctClass.toBytecode();
        FileOutputStream fos = new FileOutputStream(new File("/Users/Archerda/Desktop/java-example/com/sun/proxy/JavassitProxy.class"));
        fos.write(byteArr);
        fos.close();

        /*
        生成代理类对象实例
         */
        TargetInterface targetClass = (TargetInterface)ctClass.toClass().newInstance();

        /*
        代理类方法调用
         */
        targetClass.sayHi();
    }
}
```

``` {.java}
package com.github.archerda.designpattern.proxy;

import javassist.*;

import java.io.File;
import java.io.FileOutputStream;

/**
 * Javassist动态代理: 采用 字节码 技术创建;
 *
 * @author archerda
 */
public class JavassistByteCodeMain {

    public static void main(String[] args) throws Exception {

        /*
        创建类池
         */
        ClassPool classPool = ClassPool.getDefault();

        String className = TargetClass.class.getName();
        CtClass ctClass = classPool.makeClass(className + "JavassitProxy");

        /*
        添加接口,可选
         */
        ctClass.addInterface(classPool.get(TargetInterface.class.getName()));

        /*
        添加超类
         */
        // ctClass.setSuperclass(classPool.get(TargetClass.class.getName()));

        /*
         添加默认构造函数
         */
        // ctClass.addConstructor(new CtConstructor(new CtClass[]{}, ctClass));

        /*
        添加屬性
         */
        CtField enameField = new CtField(classPool.getCtClass(className), "targetClass", ctClass);
        enameField.setModifiers(Modifier.PRIVATE);
        ctClass.addField(enameField);

        /*
        添加构造函数
         */
        CtConstructor ctConstructor = new CtConstructor(new CtClass[]{}, ctClass);
        /*
        为构造函数设置函数体
         */
        ctConstructor.setBody("{\n" + "targetClass = new " + className + "();" + "\n}");
        /*
        把构造函数添加到新的类中
         */
        ctClass.addConstructor(ctConstructor);

        /*
        添加方法,里面进行动态代理logic
         */
        CtMethod ctMethod = new CtMethod(CtClass.voidType, "sayHi", new CtClass[]{}, ctClass);
        /*
        为自定义方法设置修饰符
         */
        ctMethod.setModifiers(Modifier.PUBLIC);
        /*
        为自定义方法设置函数体
         */
        String buffer2 = "{\n" +
                "targetClass.sayHi();" +
                "\n}";
        ctMethod.setBody(buffer2);
        ctClass.addMethod(ctMethod);

        /*
        把生成的class文件写入文件
         */
        byte[] byteArr = ctClass.toBytecode();
        FileOutputStream fos = new FileOutputStream(new File("/Users/Archerda/Desktop/java-example/com/sun/proxy/JavassitProxy.class"));
        fos.write(byteArr);
        fos.close();

        /*
        生成代理类对象实例
         */
        TargetInterface targetClass = (TargetInterface)ctClass.toClass().newInstance();

        /*
        代理类方法调用
         */
        targetClass.sayHi();
    }
}
```

### <h id="1.4.8"/>Javassist动态代理字节码源码

``` {.java}
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.github.archerda.designpattern.proxy;

public class TargetClassJavassitProxy implements TargetInterface {
    private TargetClass targetClass = new TargetClass();

    public TargetClassJavassitProxy() {
    }

    public void sayHi() {
        this.targetClass.sayHi();
    }
}
```



### 细节点 

service里面调用service，里面的service也会走aop