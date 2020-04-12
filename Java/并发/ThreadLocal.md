#ThreadLocal



##概念

ThreadLocal，即线程变量，是一个以ThreadLocal对象为键，任意对象为值的存储结构。这个机构被附带在线程上，也就是说一个线程可以通过ThreadLocal对象查询到绑定在这个线程上的一个值。





##原理

关于ThreadLocal的原理，理清四个角色关系：Thread，ThreadLocal，ThreadLocalMap，Entry

![image-20191013150324072](https://tva1.sinaimg.cn/large/006y8mN6gy1g7wlaezl1yj313q0iy0vr.jpg)

在ThreadLocal中有个变量指向ThreadLocalMap

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

ThreadLocalMap是ThreadLocal的静态内部类，当线程第一次执行set时，ThreadLocal会创建一个ThreadLocalMap对象，设置给Thread的threadLocals变量。

ThreadLocalMap中存放的是Entry，Entry是ThreadLocal和value的映射。

每一个线程都拥有一个ThreadLocalMap。



![image-20191013151928154](https://tva1.sinaimg.cn/large/006y8mN6gy1g7wlkox7kxj313s0pqdwl.jpg)



### 关于内存泄漏

ThreadLocal在ThreadLocalMap中是以一个弱引用身份被Entry中的Key引用的，因此如果ThreadLocal没有外部强引用来引用它，那么ThreadLocal会在下次JVM垃圾收集时被回收。

这个时候就会出现Entry中Key已经被回收，出现一个null Key的情况，外部读取ThreadLocalMap中的元素是无法通过null Key来找到Value的。因此如果当前线程的生命周期很长，一直存在，那么其内部的ThreadLocalMap对象也一直生存下来，这些null key就存在一条强引用链的关系：Thread --> ThreadLocalMap-->Entry-->Value，这条强引用链会导致Entry不会回收，Value也不会回收，但Entry中的Key却已经被回收的情况，造成内存泄漏。

但是JVM团队已经考虑到这样的情况，并做了一些措施来保证ThreadLocal尽量不会内存泄漏：

- 在ThreadLocal的get()、set()、remove()方法调用的时候会清除掉线程ThreadLocalMap中**所有Entry中Key为null的Value，并将整个Entry设置为null，利于下次内存回收Entry、value。**





### ThreadLocalMap处理Hash冲突

采用线性探测法来处理冲突，从当前位置往后找寻空位，空位指的是table[ i ] = null 或是 table[ i ] .key = null，将Entry插入该位置。也就是说一个Entry要么在它的hash位置上，要么就在该位置往后的某一位置上。

由于线性探测发 **table** 数组中的情况一定是一段一段连续的片段，我们将一个连续的片段称为 **run**。





### 关于线程安全性

每个线程都有自己的ThreadLocalMap，以及Entry[] 数组，只有自己操作，所以是线程安全的。那么ThreadLocal呢？它并没有可更改的状态，所以也是线程安全的，来看看它的三个成员变量

```java
// 每个ThreadLocal对象初始化后都会得到自己的hash值，之后不会再变
private final int threadLocalHashCode = nextHashCode();

// 静态对象AtomicInteger，与ThreadLocal对象无关，
// 在第一次ThreadLocal类加载时初始化
private static AtomicInteger nextHashCode = new AtomicInteger();

// 不可变
private static final int HASH_INCREMENT = 0x61c88647;
```

所以说 ThreadLocal 也是线程安全的。





## 使用场景

常用于同一次请求的参数传递。比如说把身份信息埋到ThreadLocal中，然后该请求的所有接口都可以获取到这个身份信息。





## 问题

**ThreadLocal时要注意什么？比如说内存泄漏?**

需要主动调用remove()方法释放无用的内存，原因查看上面的内存泄漏。









## 参考

[ThreadLocal内存泄漏](https://www.jianshu.com/p/a1cd61fa22da)

