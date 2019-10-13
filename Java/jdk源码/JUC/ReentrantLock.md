# ReentrantLock



##源码

### 构造器

```java
public ReentrantLock() {
  sync = new NonfairSync();
}

// 可以通过构造器实现公平、非公平
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```



### 公平锁和非公平锁

####非公平锁

啥叫非公平？就是**允许插队**。未获取锁（就是更改同步状态失败）的线程会在同步队列中等待，直到它前面的线程都被一一唤醒后，轮到了它时，它会被唤醒（代码逻辑回到acquireQueued的for循环中，再次尝试tryAcquire更改状态），那么此时如果一个还未入队列的新的线程tryAcquire，这里就会存在竞争，也就是存在被插队的情况。
如果新插入成功了，则刚刚被唤醒的线程只能尝试去获取锁，多次获取失败后，就只能被挂起，等待下一次唤醒。



#### 公平锁

之前说非公平是因为存在插队，而公平锁不允许插队。

```java
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    // hasQueuedPredecessors返回true说明前面有比它等更久的线程，你不能插队乖乖排队等待
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  // 重入逻辑代码
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```



## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970978

