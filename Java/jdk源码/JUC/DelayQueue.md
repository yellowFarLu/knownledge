# DelayQueue

延时队列

```
// 延时队列，底层基于优先级队列实现
private final PriorityQueue<E> q = new PriorityQueue<E>();
```

延迟队列，它的每个对象都有自己的到期时间，按照你定的比较规则构成一个最小堆，堆顶是优先级最大的对象。

如果你的比较规则是到期时间，那么堆顶就是最快到期的，如果不是那么堆顶就可能不是最快到期的，也就是堆里可能存在已到期的对象。



## take()

- 判断下当前队列有没有元素，如果没有元素则阻塞
- 队列头有元素，获取该元素的剩余时间delay
- 判断下是否有leader线程了，如果没有，则当前线程成为leader线程
- leader线程阻塞deplay的时间，到delay之后，就唤醒了，然后就可以获取队列头元素
- leader置空，并且唤醒等待队列的下一个线程获取队列头部





## leader节点的作用

是为了尽量减少锁竞争。

参考下面代码：

```java
if (leader != null)
  // 如果leader不为空，说明已经有线程在获取队列头部了，那就阻塞当前线程（释放锁）
  available.await();
```

如果leader不为空，表明前面已经有线程去尝试获取元素，并且还没有获取成功，在阻塞着。那么当前线程也就进入阻塞状态，不要去参与锁的竞争。







## 参考

https://blog.csdn.net/sinat_34976604/article/details/88071613

https://blog.csdn.net/u014634338/article/details/78385603

