# volatile



## 概念

**volatile用来确保变量的更新操作能够被其他线程见到**。

读取volatile类型的变量时，总会返回最新写入的值。原因如下：

- 当把变量声明为volatile类型后，编译器和运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起进行重排序
- volatile不会被缓存在寄存器或者处理器看不到的地方



volatile变量具有如下特性：

- 可见性：对一个volatile变量的读，总是能看到对这个volatile变量最后的写入
- 原子性：对单个volatile变量的读/写 具有原子性。但类似volatile++这种复合操作不具有原子性。



### 非阻塞

访问volatile变量时，不会执行加锁操作，因此不会阻塞线程。



### 底层实现

对volatile的写操作，其汇编代码会多出一个带Lock前缀的指令，

```java
0x01a3de24: **lock** addl $0x0,(%esp);
```

该指令会导致：

- 将当前处理器缓存写回内存
- 该处理器的缓存写回内存，会导致其他处理器的缓存无效



## volatile内存语义

只要变量的volatile变量，对该变量的读写就具有原子性。（即时变量是64位long或者double类型）。

如果是多个volatile操作 或者 volatile++这种复合操作，这些操作整体上不具有原子性。简而言之，volatile具有如下特性：

- 可见性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
- 原子性：对任意单个volatile变量的读写具有原子性，但类似volatile++这种复合操作不具有原子性。



### volatile读写建立的happen-before关系

从内存语义的角度来说：

- volatile读 与 锁的获取 有相同的内存语义
- volatile写 与 锁的释放 有相同的内存语义 

A线程写一个volatile变量，B线程读同一个volatile变量，那么A线程在写volatile变量的之前所有可见的共享变量，在B线程读取volatile变量的时候，这些共享变量将对B线程可见。



### volatile写-读的内存语义

volatile写的内存语义：

**当写一个volatile变量的时候，JMM会把该线程的本地内存的共享变量的值刷到主内存**



volatile读的内存语义：

**当读一个volatile变量的时候，JMM会把该线程的本地内存置为无效，接下来线程从主内存中读取共享变量的值**



发消息：

- 线程A写一个volatile变量，实际上是线程A向接下来读这个volatile变量的线程发出消息。
- 线程B读一个volatile变量，实际上是线程B接收了之前某个线程发出的消息
- 上述2个过程，实际上是线程A通过主内存向线程B发送消息

![image-20191116161143808](https://tva1.sinaimg.cn/large/006y8mN6gy1g8zy5lhunsj30yk0u0wlc.jpg)







### volatile内存语义的实现

重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM会分别限制这两种类型的重排序。JMM为编译器定制了重排序规则表，如下：



#### 重排序规则

JMM针对编译器制定的**volatile重排序规则**，用于禁止特定类型的编译器重排序。

![image-20191011223056448](https://tva1.sinaimg.cn/large/006y8mN6gy1g7umt2fqxkj313q0bogpj.jpg)

- 当**第二个操作是volatile写**时，不管第一个操作是什么，都不能重排序。确保了volatile写之前的操作不会被编译器重排序到volatile写之后。
- 当**第一个操作是volatile读**时，不管第二个操作是什么，都不能重排序。确保了volatile读之后的操作不会被编译器重排序到volatile读之前。（假如2个都是volatile读，重排序应该没有影响才对。。。）
- 当**第一个操作是volatile写**，**第二个操作是volatile读**时，不能重排序。



#### 内存屏障

编译器在生成字节码的时候，会在指令序列中插入内存屏障，来禁止特定类型的处理器重排序。

内存屏障的类型如下：

![image-20191011223244676](https://tva1.sinaimg.cn/large/006y8mN6gy1g7umuw1jz2j313u0f8n8n.jpg)

对于编译器来说，针对这么多不同的虚拟机，要设计一个最优的插入内存屏障方案几乎是不可能的。因此，JVM采用保守策略。

- 在每个volatile写操作的前面插入StoreStore屏障
  - 避免写写重排序，比如说a=1; a=2; 就不能重排序
- 在每个volatile写操作的后面插入StoreLoad屏障
  - 避免写-读重排序，比如说a=1; b = 2;
- 在每个volatile读操作的后面插入一个LoadLoad屏障
  - 这个没能理解，tem = a; tem2 = a; 应该没有影响才对吧
- 在每个volatile读操作的后面插入一个LoadStore屏障
  - 避免读写重排序

上述内存屏障插入策略非常保守，但它可以保证在任何处理器平台、任何程序中都能得到正确的volatile内存语义。

下面是保守策略下，编译器写入内存屏障后，生成的指令序列示意图：

![image-20191116164053943](https://tva1.sinaimg.cn/large/006y8mN6gy1g8zyzwdvcaj318q0t647l.jpg)

图3-19中的StoreStore屏障可以保证volatile写之前，所有普通写操作已经对其他内存可见了。这是因为StoreStore屏障保证保证上面的普通写在volatile写之前已经刷到主内存中。

volatile写后面StoreLoad屏障，此屏障的作用是为了避免volatile写和后面可能有的volatile读重排序。因为编译器无法准确判断volatile写后面是否存在volatile读（比如说volatile写之后方法return，但是调用方是否是使用volatile读，编译器是不知道的），为了保证正确实现volatile内存语义，JMM采用保守策略，在每个volatile写后面插入StoreLoad屏障。

**为什么不是volatile读前面添加StoreLoad屏障呢？**

因为volatile读写内存语义的常见使用方式是：一个线程写volatile变量，多个线程读volatile变量。因此，对一个volatile变量的写操作，一般来说是少于对该volatile变量的读操作。如果读操作远远少于写操作，将带来效率的提升。

下面是保守策略下，volatile读 插入内存屏障后生成的指令序列，如图：

![image-20191116165557282](https://tva1.sinaimg.cn/large/006y8mN6gy1g8zzfkd8ybj319w0u0qc2.jpg)

图3-20中的LoadLoad屏障禁止处理器把上面的volatile读和下面普通读重排序。LoadStore屏障禁止上面的volatile读和下面的普通写重排序。

上述的volatile写、volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile读-写的内存含义，编译器可以根据具体情况省略不必要的内存屏障，下面通过具体代码示例进行说明：

![image-20191116170035356](https://tva1.sinaimg.cn/large/006y8mN6gy1g8zzkdur25j30qw0e6wga.jpg)

针对readAndWrite()方法，编译器在生成字节码时插入内屏屏障，可做如下优化：

![image-20191116170213733](https://tva1.sinaimg.cn/large/006y8mN6gy1g8zzm3a2zyj30wn0u0h3o.jpg)

注意，最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即return，此时编译器无法断定后面是否有vola读或者写。为了安全起见，编译器一般会在最后加入一个StoreLoad屏障。



### JSR133为什么增强volatile内存语义

在旧的内存模型中，**volatile读-写 没有 锁的获取-释放  所具有的内存语义。为了提供一种比锁更加轻量级的线程通讯机制**，专家们决定增强volatile内存语义：

严格限制编译器和处理器**对volatile变量和普通变量的重排序**，确保volatile读-写 和 锁的获取-释放  具有相同的内存语义。

- volatile仅仅保证单个volatile变量的读写操作具有原子性，而锁的互斥执行特征保证整个临界区代码的执行具有原子性。
- 在性能上volatile更具优势

如果读者想使用volatile代替锁，一定要慎重。先确保线程安全再考虑效率。





##正确使用方式

**volatile有它自身的局限性，它只能保证可见性**，而加锁机制既可以保证可见性，又可以保证原子性。当且仅当满足以下所有条件时，才应该使用volatile变量：

- 对变量的写操作不依赖变量的当前值，或者你能确保只有单个线程能更新变量的值
- 该变量不会与其他状态变量一起纳入不变性条件中（如果该变量和其他状态变量一起决定了程序在多线程条件下，能正确执行，那该变量和状态变量应该直接加锁，也就不用使用volatile类型了）
- 在访问变量时不需要加锁



### 示例

```java
// 不使用volatile的情况
public class HYVolatile extends Thread {
    
    private static boolean flag = false;

    public void run() {

        while (!flag) ;

        System.out.println("i receive flag");
    }

    public static void main(String[] args) throws Exception {
        new HYVolatile().start();

        Thread.sleep(2000);

        flag = true;
    }
}

// 结果：HYVolatile线程无法看到flag的最新值，从而一直循环跑下去
```



```java
// 使用volatile的情况
public class HYVolatile extends Thread {

    private static volatile boolean flag = false;

    public void run() {

        while (!flag) ;

        System.out.println("i receive flag");
    }

    public static void main(String[] args) throws Exception {
        new HYVolatile().start();

        Thread.sleep(2000);

        flag = true;
    }
}

// 结果： i receive flag
// HYVolatile线程可以看到flag的最新值，从而退出循环
```

之前介绍CAS的文章中说CAS同时具有volatile读和volatile写的内存语义，**Concurrent包就是以volatile与CAS为基石来实现的**。

concurrent包下的源码有一个通用化的实现模式：首先声明共享变量为volatile，
随后CAS更新以实现线程之间的同步。同时配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。





## 错误使用方式

```java
public class IncrQuestionDemo {

    private volatile static int value;

    public static void main(String[] args) throws Exception {

        Thread  t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    value++;
                }
            }
        });

        Thread  t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    value++;
                }
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(value);
    }

}
```

这个例子中试图使用volatile保证同步，实际上是不可行的，因为线程1和线程2可能同时在本地读取到value是0，然后加1，然后刷回主存，然后把其他行缓存给清空了，2个线程再读取到的value还是1，起不到同步的作用。



## 总线风暴

volatile修饰的变量对应的缓存行使用MESI缓存一致性协议，这种协议需要不断从主内存嗅探”变量是否发生变化“，如果发送了，则置本地内存无效。

无效的嗅探会导致总线带宽达到峰值。从而形成”总线风暴“。因此不要大量使用volatile，只有在适用的场景才去使用。







## 参考

[volatile原理](https://zhuanlan.zhihu.com/p/137193948)

《并发编程艺术》

