# Java线程

现在操作系统在运行一个程序时，会创建一个进程。现代操作系统调度的最小单位是线程，也叫轻量级线程。在一个进程里可以创建多个线程，这些线程有各自的计数器、堆栈、局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。



一个Java程序从Main()方法开始执行，看似没有其他线程的参与，实际上任何一个java程序都是多线程执行的，如下，打印出最简单的main()方法的每条线程信息：

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;

/**
 * @author huangy on 2019-11-17
 */
public class MultiThread {

    public static void main(String[] args) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);

        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println(threadInfo.getThreadId() + "      " + threadInfo.getThreadName());
        }
    }

}

输出
7      JDWP Command Reader
6      JDWP Event Helper Thread
5      JDWP Transport Listener: dt_socket
4      Signal Dispatcher
3      Finalizer
2      Reference Handler
1      main  
```

可以看到，一个Java程序的运行不仅仅是main()的运行，而是main()线程和其他多个线程一起运行。





## 线程状态

线程可以处于以下6个状态之一，在给定的某一时刻，线程只能处于其中一个状态：

- new：初始状态。线程被构建，但是还没有调用start()方法
- runnable：运行状态。
  - Java线程将操作系统中的“就绪“和”运行“两种状态统称为”运行状态“
- blocked：阻塞状态。表示线程阻塞于锁
- waiting：等待状态。表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作，比如通知或中断
- time_waiting：超时等待状态。该状态不同于waiting，它可以在指定的时间自行返回
- terminated：终止状态。表示当前线程已经执行完毕



线程在自身的生命周期中，并不是固定的处于某个状态，而是随着代码执行在不同状态之间切换。java线程状态变迁如下图：

![image-20191117224055277](https://tva1.sinaimg.cn/large/006y8mN6gy1g91f0t0084j31610u0wvy.jpg)

由上图可以看到：

- 线程创建之后，调用start()方法开始运行
- 当线程执行wait()方法之后，线程进入阻塞状态。
  - 进入阻塞状态的线程需要依赖其他线程的通知才能重新进入运行状态
- 超时等待状态相当于等待状态的基础上增加了超时机制，到达了超时时间自动返回运行状态
- 当线程调用同步方法没有获取到锁，线程将会进入到阻塞状态
- 线程执行完Runnable的run()方法之后，将会进入终止状态
- 阻塞状态是进入synchronize修饰的代码块时线程阻塞的状态。但是阻塞在concurrent包中Lock接口的线程却是等待状态，因为Lock接口的实现均使用了LockSupport的方法，这些方法将线程挂起/唤醒。



## 守护线程

守护线程（Daemon线程）是一种支持型线程，因为它主要用于后台调度以及支持性工作。

当JVM中不存在非Daemon线程，JVM将会退出。

可以通过Thread.setDaemon(true)设置线程为守护线程。注意，只能在线程启动前设置，不能在线程启动之后设置。

在JVM退出时，Daemon线程中finally块不一定会执行。因此，在Daemon线程中不能依赖finally块来释放资源。



## 启动或终止线程

### 构造线程

在运行线程之前，首先要构造一个线程对象。线程对象在构造时需要提供线程所需要的属性，如线程所属的线程组、线程优先级、是否守护线程等信息。

一个新构造的线程对象是由其父线程来进行空间分配的，而子线程继承了父线程是否守护线程、优先级等属性。

至此一个可运行的线程就对象就初始化好了，在堆内存中等待着运行。



### 启动线程

线程对象在初始化完成后，调用start()方法可以启动这个线程。

start()方法的含义是：当前线程（即父线程）同步告诉JVM，只要线程规划器空闲，就应立即启动 调用了start()方法 的线程。

这个过程其实就是让系统安排一个时间来调用 Thread 中的 run() 方法，也就是使线程得到运行。

多线程是异步的，线程在代码中启动的顺序不是线程被调用的顺序。



### 理解中断

中断可以理解为一个线程的标志位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。

中断好比其他线程向该线程打了个招呼，其他线程通过调用该线程的Interrupt()方法对其进行中断操作。

线程通过检查自身是否被中断来进行响应，线程可以通过isInterrupted()来判断是否被中断，也可以调用Thread.interrupted()对当前线程的中断标示位进行复位。如果该线程已经是终止状态，即时对该线程进行中断，该线程的isInterrupted()方法依旧会返回false。

当方法抛出InterruptedException之前，JVM会先将该线程的中断标志位复位（也就是改为false），然后抛出InterruptedException，此时调用isInterrupted()会返回false。

IO阻塞不可以中断； 



### 过期的函数

suspend()用于挂起暂停线程，resume()用于唤醒线程，stop()用于停止线程。

这些API是过期的，不建议使用的，原因如下：

- suspend()方法调用后，线程不会释放占有的资源（比如锁），它会占有着资源进入睡眠状态，容器造成死锁。
- stop()方法会立即停止线程，stop()方法在终结一个线程时，不保证线程的资源会正常释放，通常没有给线程完成释放资源的机会，就把线程给终止了。也就是无法确定资源是否释放。



### 安全的终止线程

可以使用中断操作来安全的终止线程。因为使用中断操作，被中断的线程检测到中断标志位改变了，可以自己决定是否结束任务，然后释放资源，而不是武断的将线程终止。这种方式显然更加优雅。



## 线程间通讯



### volatile

通过volatile的内存语义，可以实现线程间通讯。



### synchronzied

synchronzied的锁获取和锁释放 符合 happens-before的**监视器锁规则**，从而保证可见性，从而实现线程间通讯。（A释放锁，B获取同一把锁，那么A释放锁之间对同步代码块内的修改，都能被线程B看到）



### 等待/通知机制

等待/通知相关方法是每一个java对象都有的，如下：

![image-20191119233630848](https://tva1.sinaimg.cn/large/006y8mN6gy1g93rv9tqyxj31k00dkao1.jpg)

等待/通知机制指的是线程A调用对象O的wait()方法挂起，然后线程B调用对象O的notify()或者notifyAll()方法，线程A则从wait()方法返回，继续执行后续操作。利用这种方法实现线程间通讯。

使用wait()、notify()、notifyAll()需要注意以下细节：

- 使用wait()、notify()、notifyAll()时需要先对**调用对象**加锁，并且需要是同一个对象
- 调用wait()方法后，线程从RUNNING状态变为waiting状态，并将这个线程放置到这个对象的等待队列
- notify()、notifyAll()方法调用之后，等待线程依旧不会从wait()方法返回，需要调用notify()、notifyAll()的线程释放锁后，等待线程才有机会重新获取锁，然后返回
- notify()方法将等待队列中一个线程移动到了同步队列中，而notifyAll()将等待队列中所有线程移动到了同步队列中，被移动的线程状态由waiting变成blocked
- 从wait()方法返回的前提是获得被调用对象的锁

从上述细节中可以知道等待/通知机制依赖于同步机制，这样子的目的是为了让等待线程从wait()方法中返回时，能感知到通知线程对变量做出的修改。（通知线程释放了锁，等待线程获取了锁，那么就存在happens-before关系，从而保证内存可见性）

![image-20191119234734277](https://tva1.sinaimg.cn/large/006y8mN6gy1g93s6rmrgjj31910u015u.jpg)

在上图中，waitThread等待线程首先获取了对象的锁，然后调用对象的wait()方法，从而放弃了锁进入到等待队列，变成waiting状态。

由于等待线程释放了锁，随后notifyThread通知线程获取对象的锁，并且调用对象的notify()方法，将等待线程从等待队列移动到同步队列，此时等待线程的状态从waiting变成blocked。

通知线程释放锁之后，等待线程再次获取锁，并从wait()方法中返回，继续执行。



#### 等待/通知的经典范式

等待线程遵循如下原则：

- 获取对象的锁
- 判断条件是否满足，不满足则调用wait()方法等待
- 等待线程被通知后仍然要检查条件
- 条件满足则执行对应的逻辑

伪代码如下：

<img src="https://tva1.sinaimg.cn/large/006y8mN6gy1g93sciiu0bj30h208qgme.jpg" alt="image-20191119235305872" style="zoom:50%;" />





通知线程遵循如下原则：

- 获取对象的锁
- 改变条件
- 通知等待线程

伪代码如下：

![image-20191119235404190](https://tva1.sinaimg.cn/large/006y8mN6gy1g93sdj1jc0j30fy06gdga.jpg)





### 管道输入输出流

利用管道也可以实现线程间通讯

对于PIPE类型的流，必须先调用connect方法把输入和输出流绑定起来，不然会报错。





### Thread.join()

利用Thread.join()也可以实现线程间通讯。

join()方法内部是使用wait()来实现的。如果线程A中调用线程B的join()方法，那么线程A就会被挂起，进入waiting状态，等到线程B终止了，就会调用自身的notifyAll()方法，唤醒线程A。





## Thread类的方法

### init()

在该方法中，调用者线程将作为父线程，新创建的线程继承父线程的部分属性，设置当前线程执行的任务，当前线程的线程ID等属性。



### start()

start()方法会让当前线程从新建状态进入就绪状态，通过调用本地方法实现。





### yield()

Java线程中的Thread.yield( )方法，译为线程让步。顾名思义，就是说当一个线程使用了这个方法之后，它就会把自己CPU执行的时间让掉，

让自己或者其它的线程运行，注意是让自己或者其他线程运行，并不是单纯的让给其他线程。

 

yield()的作用是让步。它能让当前线程由“运行状态”进入到“就绪状态”，从而让**其它具有相同优先级的**等待线程获取执行权；但是，并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权；也有可能是当前线程又进入到“运行状态”继续运行！



调用yield()的时候，不会释放锁





### sleep()

Thread.sleep(long millis)方法，让线程休眠一段时间，**使线程转到阻塞状态**。millis参数设定睡眠的时间，以毫秒为单位。**当睡眠结束后，就转为就绪（Runnable）状态**。sleep()平台移植性好。sleep() 可能会抛出 InterruptedException。因为异常不能跨线程传播回 main() 中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。

使用本地方法实现。



#### sleep和wait的区别

- sleep属于Thread类，wait()方法属于Object类

- sleep让线程休眠指定的时间，让出CPU给其他线程，但是它的监听状态依旧保持着，当指定的时间到了又会恢复运行状态；调用对象的wait()方法，会将该对象的线程挂起，直到有别的线程调用这个对象的notify()方法。(或者notifyAll()方法)

- 在调用sleep方法中，线程不会释放锁。当调用wait方法时，线程会释放锁。

- sleep能在任何时候使用；wait()、notify()、notifyAll()只能在同步控制块synchronized内使用

- sleep需要捕获异常，wait()、notify()、notifyAll()不需要捕获异常。

- 本质区别：sleep是线程运行状态控制，wait、notify()是线程之间的通讯方式。



### join()

概念：join()方法的作用，是等待这个线程结束。t.join()方法阻塞调用此方法的线程(calling thread)，此线程将被挂起，直到线程t完成，此线程再继续



#### 实例

```java
class Sleeper extends Thread {
    private int duration;

    public Sleeper(String name, int duration) {
        super(name);
        this.duration = duration;
        start();
    }

    @Override
    public void run() {
        try {
            sleep(duration);
        } catch (InterruptedException e) {
            System.out.println(getName() + " was Interrupted. "
            + " isInterrupted=" + isInterrupted());
        }
        System.out.println(getName() + " Sleeper awake");
    }
}

class Joiner extends Thread {

    private Sleeper sleeper;

    public Joiner(String name, Sleeper sleeper) {
        super(name);
        this.sleeper = sleeper;
        start();
    }

    @Override
    public void run() {
        try {
            sleeper.join();
        } catch (InterruptedException e) {
            System.out.println(getName() + " InterruptedException");
        }
        System.out.println(getName() + " Joiner awake");
    }
}

public class JoinDemo {

    public static void main(String[] args) {
        Sleeper sleeperOne = new Sleeper("sleeperOne", 1500);

        Sleeper sleeperTwo = new Sleeper("sleeperTwo", 1500);

        Joiner joinerOne = new Joiner("joinerOne", sleeperOne);

        Joiner joinerTwo = new Joiner("joinerTwo", sleeperTwo);
    }

}

输出
sleeperOne Sleeper awake
sleeperTwo Sleeper awake
joinerOne Joiner awake
joinerTwo Joiner awake
```





### interrupt()

interrupt()方法是中断当前的线程（设置中断标志位）。

线程当检测到中断标志位被设置后，**可能会抛出InteruptedExeption,，同时会清除线程的中断状态**。

通常中断可以作为取消任务的一种比较安全的方式（前提是能响应中断）。

**调用interrupt并不意味着必然停止目标线程正在进行的工作，它仅仅传递了请求中断的消息（设置中断标志位）**。
所以说我们**对中断本身最好的理解应该是：它并不会真正中断一个正在运行的线程；它仅仅是发出中断请求，线程自己会在下一个方便的时刻中断自己**。





##异常

异常不能跨线程传播，必须在本地线程处理所有异常。Executor可以解决这个问题







##优先级

给线程设置优先级，调度器倾向于让优先级高的线程先执行。（优先级高的线程得到更多的CPU时间片）

优先级低的线程也会得到执行，因此，优先权不会导致死锁，优先级较低的线程，仅仅是执行的频率较低。

线程优先级不能作为程序正确性的依赖，因为操作系统可以完全不理会Java线程对于优先级的设定。





## 后台线程

又称为"守护线程"。是指程序运行时在后台提供的一种通用服务的线程，并且这种线程并不属于程序中不可或缺的一部分，因此，当所有非后台线程结束时，程序就终止了，同时会杀死进程中的所有后台线程。

必须在程序启动之前调用setDaemon()方法，才能把它设置为后台线程。

如果一个是线程是后台线程，那么它创建的任何线程都是后台线程。



## 线程组

可以把线程归属到某一个线程组中，线程组中可以有线程对象，也可以有线程组，组中还可以有线程，这样的组织结构有点类似于树的形式，如图所示：

![img](https://images2015.cnblogs.com/blog/801753/201510/801753-20151005180622909-789401754.png)

https://www.cnblogs.com/xrq730/p/4856072.html  (貌似不推荐使用线程组了，目前使用Executor代替线程组)









## 参考

https://blog.csdn.net/u014634338/article/details/78995645

https://blog.csdn.net/u014634338/article/details/77844521

[多线程编程核心技术](https://www.infoq.cn/article/Jtv2XL3a0HvRE2xwrNFs)

