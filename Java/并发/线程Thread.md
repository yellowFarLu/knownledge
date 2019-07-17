#Thread



## 线程状态

线程可以处于以下4个状态之一：

###新建

当线程被初始化时，它只会短暂的处于这个状态。此时它已经分配了必需的系统资源，并执行了初始化。



### 就绪

在这种状态下，只要调度器把时间片分给线程，线程就可以运行。



### 阻塞

线程能够运行，但是某个条件阻止它的运行。当线程处于阻塞状态时，调度器将忽略线程，不会分配给线程任何CPU时间，直到线程重新进入就绪状态。



#### 阻塞状态的原因

- 调用sleep使线程阻塞，线程在指定时间内不会运行
- 调用wait()方法阻塞
- 任务等待输入/输出
- 同步等待(等待锁)





### 死亡

处于死亡状态的线程，将不再是可调度的，并且再也不会得到CPU时间。任务死亡的方式通常是从run()方法返回，但是任务的线程还可以被中断。









##yield()

Java线程中的Thread.yield( )方法，译为线程让步。顾名思义，就是说当一个线程使用了这个方法之后，它就会把自己CPU执行的时间让掉，

让自己或者其它的线程运行，注意是让自己或者其他线程运行，并不是单纯的让给其他线程。

 

yield()的作用是让步。它能让当前线程由“运行状态”进入到“就绪状态”，从而让**其它具有相同优先级的**等待线程获取执行权；但是，并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权；也有可能是当前线程又进入到“运行状态”继续运行！



调用yield()的时候，不会释放锁





##sleep()

Thread.sleep(long millis)方法，**使线程转到阻塞状态**。millis参数设定睡眠的时间，以毫秒为单位。**当睡眠结束后，就转为就绪（Runnable）状态**。sleep()平台移植性好。sleep() 可能会抛出 InterruptedException。因为异常不能跨线程传播回 main() 中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。



###sleep和wait的区别

- sleep属于Thread类，wait()方法属于Object类

- sleep让线程休眠指定的时间，让出CPU给其他线程，但是它的监听状态依旧保持着，当指定的时间到了又会恢复运行状态；调用对象的wait()方法，会将该对象的线程挂起，直到有别的线程调用这个对象的notify()方法。(或者notifyAll()方法)

- 在调用sleep方法中，线程不会释放锁。当调用wait方法时，线程会释放锁。

- sleep能在任何时候使用；wait()、notify()、notifyAll()只能在同步控制块synchronized内使用

- sleep需要捕获异常，wait()、notify()、notifyAll()不需要捕获异常。

- 本质区别：sleep是线程运行状态控制，wait、notify()是线程之间的通讯方式。



##join()

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







##异常

异常不能跨线程传播，必须在本地线程处理所有异常。Executor可以解决这个问题







##优先级

给线程设置优先级，调度器倾向于让优先级高的线程先执行。优先级低的线程也会得到执行，因此，优先权不会导致死锁，优先级较低的线程，仅仅是执行的频率较低。



## 后台线程

又称为"守护线程"。是指程序运行时在后台提供的一种通用服务的线程，并且这种线程并不属于程序中不可或缺的一部分，因此，当所有非后台线程结束时，程序就终止了，同时会杀死进程中的所有后台线程。

必须在程序启动之前调用setDaemon()方法，才能把它设置为后台线程。

如果一个是线程是后台线程，那么它创建的任何线程都是后台线程。



## 线程组

可以把线程归属到某一个线程组中，线程组中可以有线程对象，也可以有线程组，组中还可以有线程，这样的组织结构有点类似于树的形式，如图所示：

![img](https://images2015.cnblogs.com/blog/801753/201510/801753-20151005180622909-789401754.png)

https://www.cnblogs.com/xrq730/p/4856072.html  (貌似不推荐使用线程组了，目前使用Executor代替线程组)