# LinkedBlockingQueue



## 概述

Java阻塞队列的一种实现方式，底层基于链表实现。

LinkedBlockingQueue和ArrayBlockingQueue是FIFO队列，比同步List拥有更好的并发性。

生产和消费端使用不同的两把锁，目前已知只有LinkedBlockingQueue这样子做了，可以大大提高并发度。



## 两把锁实现

LinkedBlockingQueue内部分别使用了 takeLock 和 putLock 两个锁对并发进行控制，添加和删除操作并不是互斥操作，可以同时进行，这样也就可以大大提高吞吐量。

```java
public boolean offer(E e) {
  if (e == null) throw new NullPointerException();
  final AtomicInteger count = this.count;
  if (count.get() == capacity) // 数组满直接返回false
    return false;
  int c = -1; // 开始时设为-1，若插入成功c的值应>=0，以此来判定offer操作是否成功
  Node<E> node = new Node<E>(e);
  final ReentrantLock putLock = this.putLock;
  putLock.lock();
  try {
    // 若数组满不做任何处理，c仍未-1，最后会返回false。
    if (count.get() < capacity) {
      enqueue(node);
      c = count.getAndIncrement(); // 这里返回的是增加之前的值
      if (c + 1 < capacity) // 插入后数组未满
        notFull.signal(); // “唤醒”一个插入线程
    }
  } finally {
    putLock.unlock();
  }
  // c等于0 说明本次插入之前数组为空，则可能有不少消费的线程都在阻塞等待，
  // 所以可以在这里唤醒一个，其实并不一定会唤醒线程，很可能是将节点从
  // notEmpty 等待对队列中放回 takeLock 的同步队列。
  // 具体分析见分析 Condition 的文章
  if (c == 0) 
    signalNotEmpty();
  return c >= 0;
}
```

从上面可以看出这种双锁设计的好处，当前插入线程完成后，如果数组没有满，则“唤醒”下一个插入线程，跟取操作互不影响。



## 生产者-消费者模式代码实战

```java
import java.util.Date;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * 生产者-消费者实战
 * @author huangy on 2019-10-15
 */
public class LinkedBlockingQueueDemo {

    private static BlockingQueue<Integer> queue =
            new LinkedBlockingQueue<>(1);

    public static void main(String[] args) throws Exception {

        Thread consumer = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        System.out.println("消费者读取了数据，result=" + queue.take() + "  当前时间=" + new Date());
                        Thread.sleep(2000);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        });

        Thread provider = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        queue.put(1);
                        System.out.println("生产者放入了数据" + "  当前时间=" + new Date());
                        Thread.sleep(2000);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        });

        consumer.start();
        provider.start();

        consumer.join();
        provider.join();
    }

}
```





## 问题



**LinkedBlockingQueue是如何实现的？**

LinkedBlockingQueue是一个先进先出的阻塞队列，底层基于链表实现。

使用Condition来实现阻塞/唤醒操作。

使用ReentrantLock保证线程安全，生产端和消费端使用不同的两把锁。





## 参考

https://blog.csdn.net/sinat_34976604/article/details/87893706