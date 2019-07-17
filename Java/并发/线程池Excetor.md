### Excetor框架

Excetor是异步执行框架，它基于“生产者—消费者”模式，提交任务的操作相当于生产者，执行任务的线程相当于消费者。将任务提交和执行解耦开来。当执行的任务数量超过线程数量时，会阻塞接下来的任务，直到有任务执行完了，把线程归还到线程池中，才会继续往后执行任务。

任务：是一组逻辑工作单元，而线程则是使任务异步执行的机制。线程池简化了线程的管理工作。只需要提交任务，线程池会自动获取线程来执行该任务。



### 线程池

管理一组工作线程的资源池。线程池与工作队列(work queue)密切相关，其中在工作队列中保存了所有等待执行的任务。工作线程从队列中获取任务，执行，然后工作线程返回线程池中，等待下一个任务。

优点：

- 通过重用线程，避免多个线程创建和销毁的巨大开销
- 提高响应性：当请求到达时，工作线程通常已经存在了，不同等到创建完线程再执行任务
- 创建适量的线程数量，使处理器保持忙碌状态，同时防止过多线程相互竞争资源，导致应用程序耗尽内存



#### 线程池的类型

- newFixedThreadPool：创建一个固定长度的线程池，每当提交一个任务就创建一个线程，直到达到线程池的最大数量，这时线程池的数量将不再变化（如果某个线程由于发生了Exception而结束，那么线程池会补充一个新的线程）。newFixedThreadPool将coolPoolSize和maximumPoolSize设置为指定的值，并且创建的线程不会超时。
- newCacheThreadPool：newCacheThreadPool将创建一个可缓存的线程池。如果线程的当前规模超过了处理需求时，那么将回收空闲的线程；当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制。将coolPoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，并将超时设置为1分钟，这种方法创建出来的线程池可以无限扩张，并且当需求降低时可以自动收缩。
- newSingleThreadExecutor：是一个单线程的Executor。它创建单个工作者线程来执行任务，如果这个线程异常结束，会创建另外一个线程代替。能确保任务按照指定的顺序串行执行（FIFO、LIFO、优先级）。可以确保线程安全性。
- newScheduledThreadPool：创建一个固定线程数量的线程池，以延时 或 定时的方式来执行任务，类似于Timer



#### Executor的生命周期

ExecutorService接口中提供了一些用于管理 Executor生命周期的方法。

ExecutorService生命周期的状态有3种：运行、关闭、已终止：

- ExecutorService在创建的时候处于运行状态。

- shutdown方法将执行平缓的关闭过程：不再接收新任务，同时等待已经提交的任务执行完成（包括那些还未开始任务）
- shutdownNow方法将执行粗暴的关闭过程：它将尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务
- 在ExecutorService关闭后提交的任务，将由“拒绝执行处理器（Reject Execution Handler）”来处理。
- 等所有任务执行完毕后，ExecutorService转入终止状态。



#### 无限创建线程的不足

- 线程生命周期开销非常高：线程的创建与销毁是有代价的，根据平台的不同，实际的开销也有不同
- 资源消耗：线程会消耗系统资源，如内存。如果可运行的线程数量多于处理器的数量，那么空闲的线程还是会占用大量的内存
- 稳定性：创建线程的数量超过一定的限制，很可能抛出OutOfMemoryError



#### submit()和execute()的区别

- submit有返回值，而execute没有
- submit方便Exception处理。如果你在你的task里会抛出checked（受检查的）或者unchecked(未受检查的) exception，而你又希望外面的调用者能够感知这些exception并做出及时的处理，那么就需要用到submit，通过捕获Future.get抛出的异常。

​       比如说，我有很多更新各种数据的task，我希望如果其中一个task失败，其它的task就不需要执行了。那我就需要catch Future.get抛出的异常，然后终止其它task的执行，代码如下：

```java
public class ExecutorServiceTest {  
    public static void main(String[] args) {  
        ExecutorService executorService = Executors.newCachedThreadPool();  
        List<Future<String>> resultList = new ArrayList<Future<String>>();  
  
        // 创建10个任务并执行  
        for (int i = 0; i < 10; i++) {  
            // 使用ExecutorService执行Callable类型的任务，并将结果保存在future变量中  
            Future<String> future = executorService.submit(new TaskWithResult(i));  
            // 将任务执行结果存储到List中  
            resultList.add(future);  
        }  
        executorService.shutdown();  
  
        // 遍历任务的结果  
        for (Future<String> fs : resultList) {  
            try {  
                System.out.println(fs.get()); // 打印各个线程（任务）执行的结果  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            } catch (ExecutionException e) {  
                executorService.shutdownNow();  
                e.printStackTrace();  
                return;  
            }  
        }  
    }  
}  
  
class TaskWithResult implements Callable<String> {  
    private int id;  
  
    public TaskWithResult(int id) {  
        this.id = id;  
    }  
  
    /** 
     * 任务的具体过程，一旦任务传给ExecutorService的submit方法，则该方法自动在一个线程上执行。 
     *  
     * @return 
     * @throws Exception 
     */  
    public String call() throws Exception {  
        System.out.println("call()方法被自动调用,干活！！！             " + Thread.currentThread().getName());  
        if (new Random().nextBoolean())  
            throw new TaskException("Meet error in task." + Thread.currentThread().getName());  
        // 一个模拟耗时的操作  
        for (int i = 999999999; i > 0; i--)  
            ;  
        return "call()方法被自动调用，任务的结果是：" + id + "    " + Thread.currentThread().getName();  
    }  
}  
  
class TaskException extends Exception {  
    public TaskException(String message) {  
        super(message);  
    }  
}  


输出
all()方法被自动调用,干活！！！             pool-1-thread-1  
call()方法被自动调用,干活！！！             pool-1-thread-2  
call()方法被自动调用,干活！！！             pool-1-thread-3  
call()方法被自动调用,干活！！！             pool-1-thread-5  
call()方法被自动调用,干活！！！             pool-1-thread-7  
call()方法被自动调用,干活！！！             pool-1-thread-4  
call()方法被自动调用,干活！！！             pool-1-thread-6  
call()方法被自动调用,干活！！！             pool-1-thread-7  
call()方法被自动调用,干活！！！             pool-1-thread-5  
call()方法被自动调用,干活！！！             pool-1-thread-8  
call()方法被自动调用，任务的结果是：0    pool-1-thread-1  
call()方法被自动调用，任务的结果是：1    pool-1-thread-2  
java.util.concurrent.ExecutionException: com.cicc.pts.TaskException: Meet error in task.pool-1-thread-3  
    at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:222)  
    at java.util.concurrent.FutureTask.get(FutureTask.java:83)  
    at com.cicc.pts.ExecutorServiceTest.main(ExecutorServiceTest.java:29)  
Caused by: com.cicc.pts.TaskException: Meet error in task.pool-1-thread-3  
    at com.cicc.pts.TaskWithResult.call(ExecutorServiceTest.java:57)  
    at com.cicc.pts.TaskWithResult.call(ExecutorServiceTest.java:1)  
    at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)  
    at java.util.concurrent.FutureTask.run(FutureTask.java:138)  
    at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)  
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)  
    at java.lang.Thread.run(Thread.java:619)  
  
  
  
 **可以看见一旦某个task出错，其它的task就停止执行。**
```





### 延迟任务与周期任务

Timer类负责管理延迟任务（比如在100ms后执行该任务）以及周期任务（每10s执行1次）。Timer类存在一些缺陷，应该使用ScheduleThreadPoolExecutor来代替。

- Timer在执行定时任务时，只会创建一个线程。如果某个任务执行时间过长，将破坏其他定时任务的定时精确性。线程池能弥补这个缺陷，它可以提供多个线程来执行延迟任务和周期任务。
- Timer线程并不捕获异常，因此当TimerTask（定时任务）抛出未检查的异常时，将终止定时线程。这种情况下，Timer不会恢复线程的执行，而是错误的任务整个线程都取消了。因此，已调度但尚未执行完的任务、新的任务  将不会执行（这个问题称为线程泄漏）。





### Future和Callable

如果你希望在任务完成时返回一个值，那么可以实现Callable接口。Callable是任务的抽象。Runnable和Callable的区别如下：

- Runnable不能返回值，Callable能获取返回值
- 当将一个Callable的对象传递给ExecutorService的submit方法，则该call方法自动在一个线程上执行，并且会返回执行结果Future对象。同样，将Runnable的对象传递给ExecutorService的submit方法，则该run方法自动在一个线程上执行，并且会返回执行结果Future对象，但是在该Future对象上调用get方法，将返回null。
- Callable接口的call()方法允许抛出异常；而Runnable接口的run()方法的异常只能在内部消化，不能继续上抛；



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



####Future示例

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





#### FutureTask

FutureTask实现了RunnableFuture接口，RunnableFuture继承了Runnable接口和Future接口，所以FutureTask既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。

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









### ExecutorCompletionService

想要获取**一组**计算的结果时，可以使用ExecutorCompletionService。ExecutorCompletionService将Executor和BlockingQueue的功能融合在一起。你可以将Callable任务交给它执行，然后使用队列操作中take和poll等方法来获得已经完成的结果。

ExecutorCompletionService的实现非常简单。在构造函数中创建一个BlockingQueue来保存计算完成的结果。当计算完成时，把结果放到BlockingQueue中。



### 线程饥饿死锁

线程池中的任务 等待 池中其他任务的执行结果，将很有可能发生死锁。这种 称为“线程饥饿死锁”。



### 设置线程池的大小

- 对于计算密集型的任务（计算量大的任务）：在拥有N个处理器的系统上，当线程池的大小为N+1时，通常能实现最优的利用率。因为当某个线程因为特殊故障而暂停计算时，这个“额外”的线程能确保CPU的时钟周期没有被浪费
- 对于IO密集型的任务：由于任务不会一直执行，也就是不会一直占用CPU，因此线程池的规模应更大，确保CPU的时钟周期没有被浪费。





### 配置ThreadPoolExecutor

可以利用ThreadPoolExecutor的构造函数来修改执行策略

- coolPoolSize：线程池的最小数量，只有在工作队列满了的情况下，才会创建超出这个数量的线程。
- maximumPoolSize：线程池最大数量，表示可同时活动的线程数量上限。如果某个线程的空闲时间超过了存活时间keepAliveTime，那么将被标记为可回收的，并且当线程池的大小超过coolPoolSize，这个线程将被终止



#### 队列

来不及处理的请求，将累计起来，这些请求在Executor管理的Runnable队列中等待。ThreadPoolExecutor使用一个BlockingQueue来保存等待执行的任务。队列有3形式：有界队列、无界队列、同步移交（SynchronousQueue）

- newFixedThreadPool和newSingleThreadExecutor默认情况下使用无界的LinkedBlockingQueue，这将有可能导致队列无限增加，耗尽资源。更稳妥的方法是使用有界队列。
- SynchronousQueue：当使用这个队列时，如果没有线程在等待，并且线程池的当前大小小于最大值，那么ThreadPoolExecutor将创建一个新的线程，否则根据饱和策略，这个任务将被拒绝。只有线程池无界或者可以拒绝任务时，才能使用这种队列。



#### 饱和策略

当队列满了，将使用饱和策略来拒绝任务。有以下几种饱和策略：

- 中止（Abort）：是默认的饱和策略，该策略将抛出的RejectedExeception异常。调用者可以捕获这个异常，做相应的处理

- 抛弃（Discard）：当新提交的任务无法保存到队列中时，该策略会“悄悄”地抛弃掉该任务，所谓“悄悄”是不抛出异常，也不做其他处理。

- 抛弃最旧的（Discard-Oldest）：会抛弃下一个将被执行的任务，然后尝试重新提交新的任务。（如果工作队列是一个优先级队列，那么将导致抛弃优先级最高的任务，因此不要将该策略和优先级队列放在一起使用）

- 调用者运行（Caller-Runs）：不会抛弃任务，也不会抛出异常，而是将任务回退给调用者，从而降低新任务的流量。当队列满了之后，执行该策略，会在调用了java.util.concurrent.ThreadPoolExecutor#execute()方法的线程中执行该任务，通常情况下就是主线程。这样子就阻塞了主线程，因此主线程在一段时间内无法提交任何任务，从而使得工作者有时间处理正在执行的任务。在这期间，主线程不会调用accept，因此到达的请求被保存在TCP层的队列中。如果持续过载，这种情况会逐渐蔓延开来——从线程池 到 工作队列 到 应用程序 到 TCP层，最终到达客户端，导致服务器在高负载的情况下实现一种平缓的性能降低。

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      if (!e.isShutdown()) {
          // 该策略的实现，其实就是在当前线程执行任务
          r.run();
      }
  }
  ```






###ScheduledExecutorService

ScheduledExecutorService的主要作用就是可以将定时任务与线程池功能结合使用



#### scheduleAtFixedRate

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



#### schedule

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



#### scheduleWithFixedDelay

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

