# ThreadLocal



## 概念

ThreadLocal，即线程变量，是一个以ThreadLocal对象为键，任意对象为值的存储结构。这个结构被附带在线程上，也就是说一个线程可以通过ThreadLocal对象查询到绑定在这个线程上的一个值。





## 原理

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





## 父子线程传递实现方案

如果子线程想要拿到父线程的中的ThreadLocal值怎么办呢？



### 错误的示例

比如会有以下的这种代码的实现。由于ThreadLocal的实现机制，在子线程中调用get()方法时，我们拿到的Thread对象是当前子线程对象，那么他的ThreadLocalMap是null的，所以我们得到的value也是null。  

```java
private static void demo1() throws Exception {

  Thread.currentThread().setName("主线程");

  final ThreadLocal<String> threadLocal = new ThreadLocal<>();
  // 调用set方法的时候，会初始化一个ThreadLocalMap
  threadLocal.set("这个父线程设置的变量");

  Thread subThread = new Thread(new Runnable() {
    @Override
    public void run() {
      // 子线程获取父线程的threadLocal，结果为null
      System.out.println("子线程获取的变量为   " +
                         threadLocal.get());
    }
  });
  subThread.setName("子线程");
  subThread.start();
}

public static void main(String[] args) throws Exception {
  demo1();
}
```

那么有没有方法正确的获取父线程中的ThreadLocal呢？



### InheritableThreadLocal

那其实很多时候我们是有子线程获得父线程ThreadLocal的需求的，要如何解决这个问题呢？这就是`InheritableThreadLocal`这个类所做的事情。先来看下InheritableThreadLocal所做的事情。

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    /**
     * 重写ThreadLocal类中的getMap方法，在原Threadlocal中是返回
     * t.theadLocals，而在这么却是返回了inheritableThreadLocals，因为
     * Thread类中也有一个要保存父子传递的变量
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * 同理，在创建ThreadLocalMap的时候不是给t.threadlocal赋值
     * 而是给inheritableThreadLocals变量赋值
     * 
     */
    void createMap(Thread t， T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this， firstValue);
    }
}
```

以上代码大致的意思就是，如果你使用InheritableThreadLocal，那么保存的所有东西都已经不在原来的t.thradLocals里面，而是在一个新的t.inheritableThreadLocals变量中了。下面是Thread类中两个变量的定义

```java
/**
 * 线程所持有的threadLocals
 */
ThreadLocal.ThreadLocalMap threadLocals = null;

/**
 * 线程所持有的inheritableThreadLocals，保持了从父线程继承而来的本地变量信息
 */
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```



**InheritableThreadLocal是如何实现在子线程中能拿到当前父线程中的值的呢？**

一个常见的想法就是把父线程的所有的值都`copy`到子线程中。

```java
// Thread 线程类的初始化方法
private void init(ThreadGroup g， Runnable target， String name，
                     long stackSize， AccessControlContext acc) {
       //省略上面部分代码
       if (parent.inheritableThreadLocals != null)
       //这句话的意思大致不就是，copy父线程parent的map，创建一个新的map赋值给当前线程的inheritableThreadLocals。
           this.inheritableThreadLocals =
               ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
      //ignore
}
```

而且，在copy过程中是`浅拷贝`，key和value都是原来的引用地址

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
  Entry[] parentTable = parentMap.table;
  int len = parentTable.length;
  setThreshold(len);
  table = new Entry[len];

  for (int j = 0; j < len; j++) {
    Entry e = parentTable[j];
    if (e != null) {

      // 获取key
      @SuppressWarnings("unchecked")
      ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
      if (key != null) {
        // 获取value
        Object value = key.childValue(e.value);
        Entry c = new Entry(key， value);

        // 计算存放key的位置
        int h = key.threadLocalHashCode & (len - 1);

        // 线性探测法
        while (table[h] != null)
          h = nextIndex(h， len);
        table[h] = c;

        size++;
      }
    }
  }
```

到了这里，大致的解释了一下`InheritableThreadLocal`为什么能解决父子线程传递Threadlcoal值的问题。

1. 在创建新线程的时候会检查父线程中t.inheritableThreadLocals变量是否为null，如果不为null则拷贝一份ThradLocalMap到子线程的t.inheritableThreadLocals成员变量中去
2. 因为复写了getMap(Thread)和createMap()方法，所以调用get()方法的时候，就可以在getMap(t)的时候就会从t.inheritableThreadLocals中拿到map对象，从而实现了可以拿到父线程ThreadLocal中的值。

```java
private static void demo2() throws Exception {
  Thread.currentThread().setName("主线程");

  final ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
  // 调用set方法的时候，会初始化一个ThreadLocalMap
  threadLocal.set("这个父线程设置的变量");

  Thread subThread = new Thread(new Runnable() {
    @Override
    public void run() {
      // 子线程获取父线程的threadLocal
      // 输出为：    子线程获取的变量为   这个父线程设置的变量
      System.out.println("子线程获取的变量为   " +
                         threadLocal.get());
    }
  });
  subThread.setName("子线程");
  subThread.start();
}
```





### InheritableThreadLocal不足

我们在使用线程的时候往往不会只是简单的new Thread对象，而是使用线程池，当然线程池的好处多多。这里不详解，既然这里提出了问题，那么线程池会给InheritableThreadLocal带来什么问题呢？

我们列举一下线程池的特点：

1. 为了减小创建线程的开销，线程池会缓存**已经使用**过的线程
2. 生命周期统一管理，合理的分配系统资源

如下示例：

```java
private static void demo3() throws Exception {
        final InheritableThreadLocal<String> inheritableThreadLocal =
                new InheritableThreadLocal<>();
        inheritableThreadLocal.set("xiexiexie");

        //输出 xiexiexie
        System.out.println("父线程中获取inheritableThreadLocal， 值为：" +
                inheritableThreadLocal.get());

        Runnable runnable = new Runnable() {
            @Override
            public void run() {

                System.out.println("子线程中获取inheritableThreadLocal， 值为：" +
                        inheritableThreadLocal.get());

                inheritableThreadLocal.set("zhangzhangzhang");

                System.out.println("子线程中获取inheritableThreadLocal， 值为：" +
                        inheritableThreadLocal.get());
            }
        };

        ExecutorService executorService = Executors.newFixedThreadPool(1);
        executorService.submit(runnable);
        TimeUnit.SECONDS.sleep(1);

        /**
         * 第二次执行的时候，使用的是上一条线程，
         * 并且InheritableThreadLocal只有在线程初始化的时候才从父线程继承数据。
         * 因此这次执行任务直接使用线程当前的InheritableThreadLocal
         */
        System.out.println("父线程中获取inheritableThreadLocal， 值为：" +
                inheritableThreadLocal.get());
        executorService.submit(runnable);

        TimeUnit.SECONDS.sleep(1);

        executorService.shutdown();
}
```

可见，在使用线程池的情况，由于复用线程，所以造成InheriableThreadLocal被复用，从而导致无法使用父类的数据。



### 解决方案

如果我们能够在submit新任务的时候在重新从父线程中拷贝所有的变量。然后将这些变量赋值给当前线程的t.inhertableThreadLocal。这样就能够解决在线程池中每一个新的任务都能够获得父线程中ThreadLocal中的值而不受其他任务的影响。Alibaba的一个库解决了这个问题 [github:alibaba/transmittable-thread-local]



### transmittable-thread-local实现原理

这个库最简单的方式是这样使用的，通过简单的修饰，使得提交的runable拥有了上一节所述的功能。具体的API文档详见github，这里不再赘述。

```java
private static void demo4() throws Exception {

        ThreadLocal<Integer> threadLocal = new TransmittableThreadLocal<>();

        // 如果是ThreadLocal，子线程无法读取到父线程的值
//        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

        // 如果是InheritableThreadLocal，子线程无法重新读取到父线程的值
//      ThreadLocal<Integer> threadLocal = new InheritableThreadLocal<>();

        ExecutorService executorService =
                TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(1));

        System.out.println("主线程开启");
        threadLocal.set(1);
        System.out.println("主线程读取本地变量：" + threadLocal.get());

        executorService.submit(() -> {
            System.out.println("子线程读取本地变量：" + threadLocal.get());
        });

        TimeUnit.SECONDS.sleep(1);

        threadLocal.set(2);
        System.out.println("主线程读取本地变量：" + threadLocal.get());

        executorService.submit(() -> {
            //[读到了主线程修改后的新值]
            System.out.println("子线程读取本地变量：" + threadLocal.get());
            threadLocal.set(3);
            System.out.println("子线程读取本地变量：" + threadLocal.get());
        });

        TimeUnit.SECONDS.sleep(1);
        //依旧读取的是 2
        System.out.println("主线程读取本地变量：" + threadLocal.get());
}
```

**原理解析**

使用装饰模式包装线程池。线程池异步执行任务之前，先从父线程的ThreadLocal里面获取到数据，然后传递给子线程。子线程执行任务。





### 更新父线程ThreadLocal值？

如果线程之间出了要能够得到父线程中的值，同时想更新值怎么办呢？在前面我们有提到，当子线程copy父线程的ThreadLocalMap的时候是浅拷贝的，代表子线程Entry里面的value都是指向的同一个引用，我们只要修改这个引用的同时就能够修改父线程当中的值了。







## 问题

**ThreadLocal时要注意什么？比如说内存泄漏?**

需要主动调用remove()方法释放无用的内存，原因查看上面的内存泄漏。









## 参考

[ThreadLocal内存泄漏](https://www.jianshu.com/p/a1cd61fa22da)

[ThreadLocal父子线程传递数据](https://blog.csdn.net/a837199685/article/details/52712547)

[transmittable-thread-local源码解析](https://blog.csdn.net/liubenlong007/article/details/107050929)

