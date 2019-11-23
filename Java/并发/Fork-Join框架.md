# Fork/Join框架



## 概述

Fork/Join框架是Java7提供的用于并行执行任务的框架，是一个把大任务分割成小任务进行执行，然后把小任务的结果进行汇总，得到大任务结果的框架。

Fork/Join框架的运行流程如下图所示：

<img src="https://tva1.sinaimg.cn/large/006y8mN6gy1g9835clgp4j30sa0wkna9.jpg" alt="image-20191123170903661" style="zoom:50%;" />





## 工作窃取算法

工作窃取算法 指某个线程从其他线程的队列中窃取任务来执行。![image-20191123171218260](https://tva1.sinaimg.cn/large/006y8mN6gy1g9838q994tj30ws0sm4cs.jpg)

**为什么要使用工作窃取算法？**

假如我们需要做一个比较大的任务，可以把这个大任务分割成互不依赖的子任务，把这些任务放到不同的队列中，为每一个队列创建一个线程，用于执行队列中的任务。有些线程提前干完自己的活了，与其等着，不如从其他队列中窃取任务来执行。

队列一般使用双端队列，就是为了减少窃取任务线程和被窃取任务线程的竞争。被窃取任务线程从队列头部拿任务执行，窃取线程从队列尾部拿任务执行。



**优点**

充分利用多线程进行并行计算，减少线程间的竞争



**缺点**

某些情况下还是有竞争的，比如双端队列里只有一个任务时。（一个线程先拿到了，后面那个线程就只能阻塞了）

并且该算法消耗了更多系统资源，比如说创建多个双端队列 和 多个线程。





## Fork/Join框架的设计



**第一步：分割任务**

首先我们要有一个Fork类把大任务分割成小任务，有些小任务可能还是很大，因此要继续分割，直到分割出的小任务足够小。



**第二步：执行任务，并合并结果**

分割的小任务分别放到双端队列中，然后启动几个线程分别从双端队列中拿任务进行执行。

小任务执行完的把结果统一放到一个队列中，然后启动一个线程从队列里面拿数据，并且合并这些数据。





## 示例

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;

/**
 * 使用Fork-Join框架计算1+2+3+4的结果，每个任务最多执行2个数的相加
 * @author huangy on 2019-11-23
 */
public class CountTask extends RecursiveTask<Integer> {

    /**
     * 阈值，用于控制每个任务最多执行相加的数
     */
    private static final int THRESHOLD = 2;

    private int start;

    private int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;

        // 如果足够小，就可以进行计算
        boolean canCompute = (end - start) <= THRESHOLD;

        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            // 如果任务大于阈值，就分成2个子任务进行计算
            int middle = (end - start) / 2 + start;

            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);

            // 调用fork()方法执行任务
            leftTask.fork();
            rightTask.fork();

            // 等待子任务执行，并且得到结果
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();

            // 合并子任务的结果
            sum = leftResult + rightResult;
        }

        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(1, 4);
        Future<Integer> result = forkJoinPool.submit(task);
        try {
            System.out.println(result.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

输出
10
```





## 实现原理



## fork()

如果当前线程是工作队列，则把任务放到当前线程的工作队列中，工作线程会从队列中拿到任务并执行。

不是当前线程不是工作线程，则把任务放到其中一个工作线程的队列中。



### join()

join()方法用于阻塞当前线程，并且等待结果。

如果任务已经完成，直接返回结果。否则，任务没有执行完，则从任务数组中取出任务并且执行。

如果任务被取消，则抛出异常。如果任务产生了异常，则抛出该异常。