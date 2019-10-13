#ThreadLocal



##概念

​	维持线程封闭的一种方法，ThreadLocal提供了get和set等访问接口或方法，这些方法为每个使用该变量的线程都存有一份独立的副本，get方法会返回当前线程在调用set方法时设置的最新值。这些线程独有的变量保存在Thread对象中，当线程终止后，这些值会当做垃圾回收。



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

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

虽然`Entry`对`ThreadLocal`的引用是弱引用，但是线程的栈中持有对`ThreadLocal`的强引用，而**当对象只存在弱引用时（当然还包括虚引用）**才会被回收，这就存在**OOM**的风险。

因此需要主动调用`ThreadLocal#remove`方法，清除当前线程的所有键值对，该方法会主动将`Entry`对`ThreadLocal`的弱引用置为**null**，**key**为**null**的`Entry`会被清理。



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



