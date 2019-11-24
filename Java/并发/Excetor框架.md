#  Excetor框架



## 概述

Excetor是异步执行框架，它基于“生产者—消费者”模式，提交任务的操作相当于生产者，执行任务的线程相当于消费者。**将任务提交和执行解耦开来**。当执行的任务数量超过线程数量时，会阻塞接下来的任务，直到有任务执行完了，把线程归还到线程池中，才会继续往后执行任务。

任务：是一组逻辑工作单元，而线程则是使任务异步执行的机制。线程池简化了线程的管理工作。只需要提交任务，线程池会自动获取线程来执行该任务。



## Executor框架的两级调度模型

在HotSpot虚拟机的线程模型中，Java线程被一对一映射为本地操作系统线程。

Java线程在启动时，会创建一个本地操作系统线程。当java线程终止时，对应的本地操作系统线程也会被回收。

操作系统会调度所有线程，并将它们分配给可用的CPU。



在上层，Java多线程程序把应用分解为多个任务，然后使用Executor框架将这些任务分配给线程池中的线程。

在底层，操作系统将这些线程映射到硬件处理器上，这两级调度模型如下图所示：

![image-20191124161920054](https://tva1.sinaimg.cn/large/006y8mN6gy1g997bygcb7j30uy0u0qc0.jpg)

从上图可知，应用程序通过Executor控制上层的调度，而下层的调度由操作系统内核控制，下层的调度不受应用程序的控制。





## Executor框架的结构与成员



### Executor框架的结构

Executor框架主要由3部分组成：

- 任务
  - 被执行的任务需要实现Runable或者Callable接口
- 任务的执行
  - 包括任务执行机制的核心接口Executor
  - 继承Executor的ExecutorService接口
- 异步执行结果
  - 包括Future接口，和其实现类FutureTask



**Executor**

Executor是一个接口，它是Executor框架的基础，它将任务的提交和执行分离开来。









### Executor框架的成员



### ThreadPoolExecutor

ThreadPoolExecutor是线程池的核心实现类，用来执行被提交的任务。ThreadPoolExecutor通常由Executors来创建。Executors可以创建3种类型的ThreadPoolExecutor，分别是SingleThreadExecutor、FixedThreadExecutor、CachedThreadExecutor。

- FixedThreadExecutor：
  - 创建固定线程数目的线程池，适用于负载比较重的服务器。
- cacheThreadPool：
  - 创建一个可缓存的线程池。如果线程的当前规模超过了处理需求时，那么将回收空闲的线程；当需求增加时，则可以添加新的线程。
  - 线程池的规模不存在任何限制。将coolPoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，并将超时设置为1分钟，这种方法创建出来的线程池可以无限扩张，并且当需求降低时可以自动收缩。
  - 适用于执行很多短期的任务。或者负载较轻的服务器。（便于资源回收）
- singleThreadExecutor：
  - 是一个单线程的线程池。
  - 适用于需要保证任务串行执行执行
- scheduledThreadPool：创建一个固定线程数量的线程池，以延时 或 定时的方式来执行任务，类似于Timer



### ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor是一个实现类，可以在给定的延时后执行任务，或者定期执行任务。ScheduledThreadPoolExecutor通常由Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor。

- ScheduledThreadPoolExecutor
  - 包含若干个线程的ScheduledThreadPoolExecutor
  - 适用于需要多个线程执行周期任务
  - 需要限制工作线程的数量
- SingleThreadScheduledExecutor
  - 只包含若一个线程的ScheduledThreadPoolExecutor
  - 适用于需要单个后台线程执行周期任务，需要保证顺序执行各个任务的应用场景



###  Callable和Runnable

如果你希望在任务完成时返回一个值，那么可以实现Callable接口。Callable和Runnable都是任务的抽象。Runnable和Callable的区别如下：

- Runnable不能返回值，**Callable能获取返回值**
- **Callable接口的call()方法允许抛出异常**（通过调用Future的get()方法，可以捕获该异常）；而Runnable接口的run()方法的异常只能在内部消化，不能继续上抛；而且如果抛出异常导致子线程挂了，那么任务就不会执行下去了。
- 当将一个Callable的对象传递给ExecutorService的submit方法，则该call方法自动在一个线程上执行，并且会返回执行结果Future对象。同样，将Runnable的对象传递给ExecutorService的submit方法，则该run方法自动在一个线程上执行，并且会返回执行结果Future对象，但是在该Future对象上调用get方法，将返回null。







## ThreadPoolExecutor详解

ThreadPoolExecutor是线程池最核心的类，通过Executors可以创建3种类型的ThreadPoolExecutor，如下：

FixedThreadPool、SingleThreadPool、CacheThreadPool



### FixedThreadPool详解

FixedThreadPool被称为 可重用固定线程数的线程池。

当线程池中的线程数据大于corePoolSize，多余的线程在keepAliveTime之后，将被销毁。FixedThreadPool把KeepAliveTime设置为0，意味着多余的空闲线程将立即被终止。

FixedThreadPool的execute()方法运行如图：

![image-20191124170133823](https://tva1.sinaimg.cn/large/006y8mN6gy1g998jv1c82j31520u047y.jpg)

- 第一步：如果当前运行的线程数目少于corePoolSize，则创建新的线程来执行该任务
- 第二步：在线程池完成预热之后（线程数目等于corePoolSize），将任务加入LinkedBlockingQueue。
- 第三步：线程执行完第一步的任务后，会反复从队列中获取任务来执行

FixedThreadPool使用无界队列LinkedBlockingQueue来作为线程池的工作队列，使用无界队列作为线程池的工作队列带来如下影响：

- 当线程池中线程达到corePoolSize，由于队列无界，多余的任务都被提交到队列中，线程池中的线程数目永远不会超过corePoolSize
- 由于使用无界队列，maximumPoolSize将是一个无效参数
- 由于上述两点，可以总结出keepAliveTime将是一个无效参数
- 由于使用无界队列，因此线程池不会拒绝任务



### SingleThreadExecutor详解

SingleThreadExecutor是一个使用单个工作线程的Executor。

SingleThreadExecutor的corePoolSize和maximumPoolSize设置为1，其他参数与FixedThreadPool相同。

SingleThreadExecutor使用无界队列LinkedBlockingQueue来作为线程池的工作队列。

SingleThreadExecutor的execute()运行示意图如下：

<img src="https://tva1.sinaimg.cn/large/006y8mN6gy1g998uprx1bj31fs0t4wo5.jpg" alt="image-20191124171159805" style="zoom:50%;" />

- 第一步：如果当前线程数少于corePoolSize，则创建一个新的线程来执行任务
- 第二步：在线程池完成预热之后，将任务加入LinkedBlockingQueue
- 第三步：线程执行完第一步的任务后，反复从工作队列中获取任务执行



### CacheThreadPool详解

CacheThreadPool是一个会根据需要创建新线程的线程池。

CacheThreadPool的corePoolSize设置为0，表明线程池可以没有线程。

maximumPoolSize设置为Integer.MAX_VALUE，说明可以无限创建线程。

keepAliveTime设置为60，说明空闲线程等待新任务的最长时间是60秒，超过60秒将被终止。

CacheThreadPool使用SynchronousQueue作为线程池的工作队列。

**如果主线程中提交任务的速度高于线程处理任务的速度，那么将会无限创建新的线程，极端情况下，会因为创建线程过多而耗尽CPU和内存资源。**

CacheThreadPool的execute()执行方法如图：

![image-20191124171859529](https://tva1.sinaimg.cn/large/006y8mN6gy1g999203knmj31a20tkgur.jpg)

- 首先把任务放到SynchronousQueue中
- 如果有空闲线程，则从队列中获取任务执行
- 否则，线程池将创建一个新的线程来执行任务
- 工作线程执行完刚刚的任务后，调用poll(keepAliveTime)方法尝试从队列中获取任务，如果在60秒内没有获取到任务，则被终止



通过SynchronousQueue，把主线程提交的任务传递给工作线程执行。

![image-20191124172254630](https://tva1.sinaimg.cn/large/006y8mN6gy1g99962n1n8j31es0u07j8.jpg)





## ScheduledThreadPoolExecutor详解

ScheduledThreadPoolExecutor用于在给定延迟后运行任务，或者定期执行任务。

ScheduledThreadPoolExecutor与Timer功能类似，但是ScheduledThreadPoolExecutor功能更加强大：

- TImer对应单个后台线程
- ScheduledThreadPoolExecutor可以创建多个后台线程



### ScheduledThreadPoolExecutor的运行机制

ScheduledThreadPoolExecutor的执行示意图如下：

![image-20191124172711533](https://tva1.sinaimg.cn/large/006y8mN6gy1g999aj84ybj314b0u013o.jpg)

DelayQueue是一个无界队列，所以设置maximumPoolSize没有效果。

ScheduledThreadPoolExecutor的执行主要分为2大部分：

- 当调用ScheduledThreadPoolExecutor的schduleAtFixedRate()方法 或者 schduleWithFixedDelay()方法时，会向DelayQueue添加一个ScheduleFutureTask。
- 工作线程反复从DelayQueue中获取ScheduleFutureTask来执行





### ScheduledThreadPoolExecutor的实现

ScheduledThreadPoolExecutor会把待调度的任务ScheduleFutureTask放到一个DelayQueue中。

ScheduleFutureTask主要包含3个成员变量：

- time：表示这个任务将被执行的具体时间
- sequenceNumber：表示这个任务被添加到ScheduledThreadPoolExecutor中的序号，早提交的序号小。
- period：表示任务执行的间隔周期

DelayQueue封装了一个PriorityQueue，这个PriorityQueue会对队列中的ScheduleFutureTask进行排序，排序时，time小的task排在前面，即时间早的任务先执行。

如果两个ScheduleFutureTask的time相同，就比较sequenceNumber，sequenceNumber小的ScheduleFutureTask排在前面，即提交事件相同，先提交的任务先执行。



**ScheduledThreadPoolExecutor线程执行周期任务的过程**

![image-20191124183629143](https://tva1.sinaimg.cn/large/006y8mN6gy1g99bamk3wdj317c0u07in.jpg)

- 工作线程从DelayQueue中获取已到期的ScheduleFutureTask。
  - 到期的任务指ScheduleFutureTask的time大于等于当前时间
  - 只有到时间的任务才能被工作线程获取，如果队列头的任务还没到时间，将阻塞工作线程
- 工作线程执行该ScheduleFutureTask
- 工作线程修改ScheduleFutureTask的time为下次将要执行的时间
- 工作线程把修改time之后的ScheduleFutureTask放回到DelayQueue











##  Executor的生命周期

ExecutorService接口中提供了一些用于管理 Executor生命周期的方法。

ExecutorService生命周期的状态有3种：运行、关闭、已终止：

- ExecutorService在创建的时候处于运行状态。

- shutdown方法将执行平缓的关闭过程：不再接收新任务，同时等待已经提交的任务执行完成（包括那些还未开始任务）
- shutdownNow方法将执行粗暴的关闭过程：它将尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务
- 在ExecutorService关闭后提交的任务，将由“拒绝执行处理器（Reject Execution Handler）”来处理。
- 等所有任务执行完毕后，ExecutorService转入终止状态。







##  延迟任务与周期任务

Timer类负责管理延迟任务（比如在100ms后执行该任务）以及周期任务（每10s执行1次）。Timer类存在一些缺陷，应该使用ScheduleThreadPoolExecutor来代替。

- Timer在执行定时任务时，只会创建一个线程。如果某个任务执行时间过长，将破坏其他定时任务的定时精确性。线程池能弥补这个缺陷，它可以提供多个线程来执行延迟任务和周期任务。
- Timer线程并不捕获异常，因此当TimerTask（定时任务）抛出未检查的异常时，将终止定时线程。这种情况下，Timer不会恢复线程的执行，而是错误的任务整个线程都取消了。因此，已调度但尚未执行完的任务、新的任务  将不会执行（这个问题称为线程泄漏）。



##  Future

Future和其实现类FutureTask用来表示异步执行的结果。

Future是一个任务的生命周期。提供了方法表示任务是否完成及取消，并且可以获取任务的结果。



get()方法的行为取决于任务的状态。如果任务已经完成了，get会立即返回或抛出异常（**也就是可以获取子线程内产生的异常**）；如果任务没有完成，get会阻塞直到任务完成。



Future可以调用cancel()方法。可以用来中断某个特定任务。如果将true传递给cancel()，那么它就拥有在该线程上调用interrupt()以停止这个线程的权限。因此，cancel()是一种中断由Executor启动的单个线程的方式。

```java
class SleepBlocked implements Runnable {
    @Override
    public void run() {
        try {
            TimeUnit.SECONDS.sleep(100);
        } catch (InterruptedException e) {
            System.out.println("SleepBlocked catch  " + e);
        }
        System.out.println("exists SleepBlocked");
    }
}

class IOBlock implements Runnable {

    private InputStream in;

    public IOBlock(InputStream in) {
        this.in = in;
    }

    @Override
    public void run() {
        try {
            in.read();
        } catch (IOException e) {
            if (Thread.currentThread().isInterrupted()) {
                System.out.println("IOBlock isInterrupted");
            } else {
                System.out.println(e);
            }
        }

        System.out.println("exists IOBlock");
    }
}

class SynchonizeBlock implements Runnable {
    public synchronized void f() {
        while (true) {
            Thread.yield();
        }
    }

    public SynchonizeBlock() {
        // 构造函数初始化线程，并且占用锁
        new Thread() {
            @Override
            public void run() {
                f();
            }
        }.start();
    }

    @Override
    public void run() {
        // 另外一条线程执行run（）方法，尝试获取锁
        System.out.println("try to call f()");
        f();
        System.out.println("exists SynchonizeBlock");
    }
}

public class InterruptExecutorDemo {

    private static ExecutorService exec = Executors.newCachedThreadPool();

    static void test(Runnable r) throws InterruptedException {
        Future<?> f = exec.submit(r);
        TimeUnit.MILLISECONDS.sleep(100);
        System.out.println("begin to interrupt " + r.getClass().getName());
        f.cancel(true);

    }

    public static void main(String[] args) throws Exception {
        // sleep阻塞 能够中断
//        test(new SleepBlocked());

        // IO阻塞不可以中断
//        test(new IOBlock(System.in));

        // 同步等待锁的阻塞 不可以中断
        test(new SynchonizeBlock());

        exec.shutdown();
    }

}
```





只有当大量相互独立且同构（相同）的任务可以并发处理时，才能体现出程序的工作负载分配到多个线程中带来的性能提升。



###Future示例

```java
class TaskWithResult implements Callable<String> {

    private int id;

    public TaskWithResult(int id) {
        this.id = id;
    }

    @Override
    public String call() throws Exception {
        return "TaskWithResult, id=" + id;
    }
}

public class CallableDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newCachedThreadPool();
        ArrayList<Future<String>> futures = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            // submit方法提交Callable任务，并且返回Future
            futures.add(executorService.submit(new TaskWithResult(i)));
        }

        for (Future<String> future : futures) {
            System.out.println(future.get());
        }

        executorService.shutdown();
    }

}

```





### FutureTask

FutureTask是Future的实现类，可以作为Future得到Callable的返回值。

FutureTask可以处于以下3种状态：

- 未启动
  - 指新建一个FutureTask，但是还没有调用run()方法
- 已启动
  - run()方法被执行的过程中
- 已完成
  - FutureTask的run()方法正常执行完、或者任务被取消、获取任务因抛出异常而结束

![image-20191124185224127](https://tva1.sinaimg.cn/large/006y8mN6gy1g99br75nhej31hk0q4drj.jpg)

FutureTask处于不同的状态，调用其方法会有不同的影响：

- get()方法
  - 当FutureTask处理未启动 或者 已启动状态，get()方法将导致调用线程阻塞。
  - 当FutureTask处于已完成状态，get()方法将立即返回结果或抛出异常。
- cancel()方法
  - 当FutureTask处于未启动状态时，调用cancel()方法将导致此任务不会被执行。
  - 当FutureTask处于已启动状态，cancel(true)将尝试中断线程，来停止任务
  - 当FutureTask处于已启动状态，cancel(false)将不会对正在执行任务的线程产生影响，会让任务执行完成
  - 当FutureTask处于已完成状态，cancel方法返回false

![image-20191124190101683](https://tva1.sinaimg.cn/large/006y8mN6gy1g99c066hgfj31650u015t.jpg)



#### 示例

```java
public class FutureTaskDemo {

    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newCachedThreadPool();

        MyTask myTask = new MyTask();
        FutureTask<Integer> futureTask = new FutureTask<>(myTask);

        executor.submit(futureTask);
        executor.shutdown();

        System.out.println("主线程在执行任务, time=" + System.currentTimeMillis());

        System.out.println("task运行结果"+futureTask.get());
    }
}

class MyTask implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算, time=" + System.currentTimeMillis());

        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        System.out.println("子线程计算完了, time=" + System.currentTimeMillis());
        return sum;
    }
}
```





#### 实现原理

FutureTask的实现基于AQS。设计意图如下：

![image-20191124192348506](https://tva1.sinaimg.cn/large/006y8mN6gy1g99cnvcswuj30zd0u04ao.jpg)

Sync是FutureTask的内部私有类，它继承AQS。创建FutureTask时会创建内部成员Sync，FutureTask的共有方法会委托给Sync。



##### get()方法

- FutureTask的get()方法会判断任务是否已执行完或者已取消，如果是的话，则get()方法立即返回；

- 否则，到线程等待队列中，等待其他线程执行release操作。

- 当其他线程执行release操作（比如说执行了FutureTask的run()方法 或者 cancel方法），唤醒当前线程后，当前线程再次尝试tryAcquireShare，当前线程将离开线程等待队列，并且唤醒后面等待的线程。（这里会产生级联唤醒的效果）。

- 最后返回计算的结果，或者抛出异常



##### run()方法

- 执行任务
- 以原子方法更新state，表示计算完毕
- 唤醒等待队列中的第一个线程



##### 流程综述

当执行FutureTask的get()方法，如果FutureTask不是出于已完成或者已取消状态，当前线程将会到AQS的同步队列中等待（见下图线程A、B、C、D）。

当某个线程执行FutureTask的run()方法 或者 FutureTask的cancel()方法时，会唤醒同步队列的第一个线程。

![image-20191124193601374](https://tva1.sinaimg.cn/large/006y8mN6gy1g99d0kn5sqj312p0u0n8j.jpg)

假设开始时FutureTask出于未启动状态 或者 已启动状态，同步队列中已经有3个线程（A、B、C）在等待。此时，线程D执行get()方法将导致线程D也到同步队列中等待。

当线程E执行run()方法时，会唤醒队列中的第一个线程A。线程A被唤醒后，首先把自己从队列中删除，然后唤醒它的后继线程B，最后线程A从get()方法返回。

线程B、C、D重复线程A的处理流程。最终，在队列中所有线程被级联唤醒，并从get()方法返回。

参考:[1.7 FutureTask实现](https://www.jianshu.com/p/385df6faf585)



##### 注意

新版本的FutureTask自己实现了队列。每一个FutureTask都有一条等待队列，当调用get()，任务还没有执行完毕，则阻塞当前线程，往队列中加入一个节点。当执行完线程，则唤醒队列头部节点。

参考:[1.8 FutureTask实现](https://www.jianshu.com/p/d0be341748de)







##  ExecutorCompletionService

想要获取**一组**计算的结果时，可以使用ExecutorCompletionService。ExecutorCompletionService将Executor和BlockingQueue的功能融合在一起。你可以将Callable任务交给它执行，然后使用队列操作中take和poll等方法来获得已经完成的结果。

ExecutorCompletionService的实现非常简单。在构造函数中创建一个BlockingQueue来保存计算完成的结果。当计算完成时，把结果放到BlockingQueue中。










## ScheduledExecutorService

ScheduledExecutorService的主要作用就是可以将定时任务与线程池功能结合使用



### scheduleAtFixedRate

使用scheduleAtFixedRate()方法实现周期性执行

```java
public class ScheduledExecutorServiceTest {

    public static void main(String[] args) {

        ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();

        executorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("run "+ System.currentTimeMillis());
            }

            // 1秒以后开始执行，而且每隔100毫秒执行1次
        }, 1000, 100, TimeUnit.MILLISECONDS);
    }
}
```



### schedule

ScheduledExecutorService使用Callable延迟运行

```java
public class CallableRun {

    static class MyCallableA implements Callable<String> {

        @Override
        public String call() throws Exception {
            try {
                System.out.println(
                        "callA begin " + Thread.currentThread().getName() + ", "
                                + System.currentTimeMillis());
                TimeUnit.SECONDS.sleep(3); // 休眠3秒
                System.out.println("callA end " + Thread.currentThread().getName() + ", "
                        + System.currentTimeMillis());
            } catch (Exception e) {
                e.printStackTrace();
            }
            return "returnA";
        }
    }

    static class MyCallableB implements Callable<String>  {

        @Override
        public String call() throws Exception{
            System.out.println("callB begin " + Thread.currentThread().getName() + ", "
                    + System.currentTimeMillis());
            System.out.println("callB end " + Thread.currentThread().getName() + ", "
                    + System.currentTimeMillis());
            return "returnB";
        }
    }

    public static void main(String[] args) {

        try {

            MyCallableA myCallableA = new MyCallableA();
            MyCallableB myCallableB = new MyCallableB();

            ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();

            // 延迟3秒执行A
            ScheduledFuture futureA =
                    executorService.schedule(myCallableA, 3L, TimeUnit.SECONDS);

            // 延迟1秒执行B
            ScheduledFuture futureB =
                    executorService.schedule(myCallableB, 1L, TimeUnit.SECONDS);

            System.out.println("返回值A：" + futureA.get());
            System.out.println("返回值B：" + futureB.get());

            executorService.shutdown();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```



### scheduleWithFixedDelay

使用scheduleWithFixedDelay()方法实现周期性执行

```java
public class RunMain {

    static class MyRunable implements Runnable {
        @Override
        public void run() {
            System.out.println("MyRunable run, time=" + System.currentTimeMillis());
        }
    }

    public static void main(String[] args) {
        ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();

        // 在1秒之后，间隔2秒执行1次
        executorService.scheduleWithFixedDelay(new MyRunable(), 1, 2, TimeUnit.SECONDS);
    }
}
```





## 扩展



### 线程饥饿死锁

线程池中的任务 等待 池中其他任务的执行结果，将很有可能发生死锁。这种 称为“线程饥饿死锁”。