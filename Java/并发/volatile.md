#volatile



##概念

用来确保变量的通知操作更新到其他线程。读取volatile类型的变量时，总会返回最新写入的值。原因如下：

- 当把变量声明为volatile类型后，编译器和运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起进行重排序
- volatile不会被缓存在寄存器或者处理器看不到的地方



###非阻塞

访问volatile变量时，不会执行加锁操作，因此不会阻塞线程。



###底层实现

对volatile的写操作，其汇编代码会多出一个带Lock前缀的指令，

```java
0x01a3de24: **lock** addl $0x0,(%esp);
```

该指令会导致：

- 将当前处理器缓存写回内存
- 该处理器的缓存写回内存，会导致其他处理器的缓存无效



## 内存语义

**volatile写的内存语义**：把线程对应的本地内存中共享变量刷到主内存

**volatile读的内存语义**：当线程读取volatile变量时，JMM会将该线程的本地内存置为无效，线程只能从主内存中读取该共享变量。

A写一个volatile变量，随后B读到这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息。



### volatile内存语义的实现

#### 重排序规则

JMM针对编译器制定的**volatile重排序规则**

![image-20191011223056448](https://tva1.sinaimg.cn/large/006y8mN6gy1g7umt2fqxkj313q0bogpj.jpg)

- 当**第二个操作是volatile写**时，不管第一个操作是什么，都不能重排序。确保了volatile写之前的操作不会被编译器重排序到其之后。
- 当**第一个操作是volatile读**时，不管第二个操作是什么，都不能重排序。确保了volatile读之后的变量不会被重排序到之前。（没懂）
- 当**第一个操作是volatile写**，**第二个操作是volatile读**时，不能重排序。



#### 内存屏障

通过插入内存屏障来禁止特定类型的处理器重排序

为了应对不同处理器，JMM采取保守策略：在每个volatile写前插入StoreStore，写后插入StoreLoader，在每个volatile读后插入LoardLoard, LoardStore。

![image-20191011223244676](https://tva1.sinaimg.cn/large/006y8mN6gy1g7umuw1jz2j313u0f8n8n.jpg)

之前介绍CAS的文章中说CAS同时具有volatile读和volatile写的内存语义，**Concurrent包就是以volatile与CAS为基石来实现的**。



concurrent包下的源码有一个通用化的实现模式：首先声明共享变量为volatile，
随后CAS更新以实现线程之间的同步。同时配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。





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







##确保可见性

volatile可以确保域的可见性。如果你将1个域声明为volatile，那么只要对这个域进行了修改操作，那么所有的读操作都可以看到这个修改。即便使用了本地缓存，也是如此，因为volatile域会立即被写入主内存，而读操作就发生在主内存中。

如果多个线程同时访问某个域，那么这个域就应该是volatile的。否则这个域就只能使用同步来访问，同步也会导致向主内存中刷新。





