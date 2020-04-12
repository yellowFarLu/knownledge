# DelayQueue



## 概述

DelayQueue（延迟队列）是一个无界的BlockingQueue，用于放置实现了Deplay接口的对象。其中的元素只有在其到期时，才能从队列中取走。

这种队列是有序的，可以自己利用compareTo()接口进行排序。不能将null元素放置到这种队列中。

任务的创建顺序对任务的执行顺序没有任何影响，任务的执行顺序是按照延迟顺序执行的。



底层基于优先级队列实现：

```java
private final PriorityQueue<E> q = new PriorityQueue<E>();
```



延迟队列，它的每个对象都有自己的到期时间，按照你定的比较规则构成一个最小堆，堆顶是优先级最大的对象。

如果你的比较规则是到期时间，那么堆顶就是最快到期的，如果不是那么堆顶就可能不是最快到期的，也就是堆里可能存在已到期的对象。

是一个无界阻塞队列。



## 源码

take()和offer()是同一把锁。



### take()

- 获取锁
- 判断下当前队列有没有元素，如果没有元素则阻塞
- 队列头有元素，获取该元素的剩余时间delay
- 当前线程阻塞deplay的时间之后，被唤醒了，然后就可以获取队列头元素
- 获取队列头元素之后，判断队列中还有其他元素，则唤醒其他线程
- 释放锁



### offer()

- 获取锁
- 向队列中添加任务
- 如果添加的任务是头元素，则唤醒其他线程来获取任务
- 释放锁





### leader节点的作用

**是为了尽量减少锁竞争**。如果leader不为空，表明前面已经有线程尝试去获取元素，并且还没有获取成功，在阻塞着。那么当前线程也就进入阻塞状态，不要去参与锁的竞争。

参考下面代码：

```java
if (leader != null)
  // 如果leader不为空，说明已经有线程在获取队列头部了，那就阻塞当前线程（释放锁）
  available.await();
```







## 应用场景

- 缓存系统设计
  - 可以用DelayQueue保存元素的有效期，然后使用一条线程调用DelayQueue的take()方法，如果元素出队了，证明该元素到期了，那么就可以从缓存中删除该元素。
- 定时任务
  - 使用DelayQueue保存将执行的任务和执行时间，一旦时间到了，就从DelayQueue中获取到元素，就可以开始执行任务。TimerQueue就是使用DelayQueue实现的。





## 代码实战

```java
import java.util.concurrent.*;

/**
 * @author huangy on 2019-05-04
 */

class DelayedTask implements Delayed {

    /**
     * 唯一标明这个任务
     */
    private int id;

    /**
     * 延迟多久就可以执行这个任务（相对时间）
     */
    private long deplay;

    /**
     * 到期时间（绝对时间）
     */
    private long deadLine;

    public DelayedTask(int id, long deplay) {
        this.id = id;
        this.deplay = deplay;

        // 计算过期时间（到这个点就过期了）
        this.deadLine = System.currentTimeMillis() + deplay;
    }

    /**
     * 返回任务剩余的延迟时间
     * @param unit 单位，剩余时间必须转换成传入的单位形式
     */
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(deadLine - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    /**
     * 延迟队列是有序的
     * 出队的顺序就是延迟队列中元素的顺序
     *
     * 本例子中，越早过期的任务在前面
     */
    @Override
    public int compareTo(Delayed other) {
        DelayedTask otherDelayedTask = (DelayedTask)other;
        if (this.deadLine < otherDelayedTask.deadLine) {
            return -1;
        } else if(this.deadLine > otherDelayedTask.deadLine) {
            return 1;
        } else {
            return 0;
        }
    }

    @Override
    public String toString() {
        return "DelayedTask{" +
                "id=" + id +
                ", deplay=" + deplay +
                ", deadLine=" + deadLine +
                '}';
    }
}

/**
 * 延迟队列消费者
 */
class DelayedTaskComsumer implements Runnable {

    private DelayQueue<DelayedTask> queue;

    public DelayedTaskComsumer(DelayQueue<DelayedTask> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {

        try {

            while (!queue.isEmpty()) {

                /**
                 * 按照队列中元素的顺序"取"
                 * 如果队列头部的元素没有到时间，则会阻塞当前线程。（说明后面的元素即时到点了，也无法返回）
                 */
                DelayedTask delayedTask = queue.take();

                System.out.println("DelayedTaskComsumer get delayedTask, delayedTask=" + delayedTask);

                TimeUnit.MILLISECONDS.sleep(100);
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("DelayedTaskComsumer finished");
    }
}



public class DelayedQueueDemo {

    public static void main(String[] args) {

        ExecutorService exec = Executors.newCachedThreadPool();

        DelayQueue<DelayedTask> queue = new DelayQueue<>();

        for (int i = 0; i < 20; i++) {
            queue.put(new DelayedTask(i, i * 100));
        }

        exec.execute(new DelayedTaskComsumer(queue));

        exec.shutdown();
    }

}
```

**Delayed** 用来标记那些应该在给定延迟时间之后执行的对象。此接口的实现必须定义一个compareTo方法，该方法提供与此接口的getDelay方法一致的排序。





## 问题



**DelayQueue是如何实现的？**

DelayQueue是一个延迟队列，底层由优先级队列实现，元素按照过期时间进行排序，最早过期的元素在队列头部。

获取元素的时候，如果元素还没有过期，则休眠一段时间，等到元素过期后，唤醒消费者线程来获取元素。

<br/>



## 参考

https://blog.csdn.net/sinat_34976604/article/details/88071613

https://blog.csdn.net/u014634338/article/details/78385603

