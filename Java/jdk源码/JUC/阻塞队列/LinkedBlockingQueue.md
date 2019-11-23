# LinkedBlockingQueue

Java阻塞队列的一种实现方式。



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
  // c等于0 说明本次插入之前数组为空，则可鞥有不少获取操作的线程都在阻塞等待，
  // 所以可以在这里唤醒一个，其实并不一定会唤醒线程，很可能是将节点从
  // notEmpty 等待对队列中放回 takeLock 的同步队列。
  // 具体分析见我分析 Condition 的文章
  if (c == 0) 
    signalNotEmpty();
  return c >= 0;
}
```

从上面可以看出这种双锁设计的好处，当前插入线程完成后，如果数组没有满，则“唤醒”下一个插入线程，跟取操作互不影响。





## 参考

https://blog.csdn.net/sinat_34976604/article/details/87893706