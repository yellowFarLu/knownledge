# ReentrantLock



## 概述

ReentrantLock是一个可重入的互斥锁，是实现了AQS的同步器。

ReentrantLock中state表示**持有锁的线程已重复获取该锁的次数**。初始时state=0，表示未锁定状态，当线程A调用lock()方法，使得资源对象被锁定。其他线程再对该资源进行lock()，就会失败。当时线程A可以自己继续lock()，lock()一次，state字段值增加1。调用lock方法多少次，就要调用多少次unlock方法，资源才会释放。



此类的构造方法提供一个可选的公平参数，默认是非公平的。公平与非公平有何区别**，**所谓

- **公平**就是严格按照FIFO的顺序获取锁
- **非公平**可以插队，这也是默认的

ReentrantLock属于独占锁，只有持有锁的线程能访问数据

等待可中断：当持有锁的线程长期不释放锁时，等待锁的线程可以选择放弃等待，改为处理其他事情

锁绑定多个条件：一个ReentrantLock对象可以同时绑定多个Condition对象。

Condition对象：用于管理那些已经获得了锁，但是却不满足往下执行条件的线程

ReentrantLock是唯一实现了Lock接口的类，并且ReentrantLock提供了更多的方法。



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
    // hasQueuedPredecessors返回true说明前面有比它等更久的线程，你不能插队，需要排队等待
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



#### 区别

非公平锁可能导致导致饥饿。但是非公平锁的效率更高，能够减少线程上下文切换。



### 几个lock()方法的区别

```
* lock、tryLock、lockInterruptibly、tryLock(long time, TimeUnit timeUnit)
*
* - lock 拿不到锁会一直等待。tryLock是去尝试，拿不到就返回false，拿到返回true

* - tryLock 拿到锁返回true，否则false

* - tryLock(long time, TimeUnit timeUnit) 拿不到锁，就等待一段时间，获取到锁返回true，超时返回false

* - lockInterruptibly 调用后如果没有获取到锁会一直阻塞，阻塞过程中会接受中断信号
```

```java
public static void main(String[] args) throws Exception {

        ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

        ReentrantReadWriteLock.ReadLock readLock = readWriteLock.readLock();

        ReentrantReadWriteLock.WriteLock writeLock = readWriteLock.writeLock();

        Thread t1 = new Thread(() -> {
            try {
                readLock.lock();
                System.out.println("线程1 获取了读锁");

                Thread.sleep(3000);

                readLock.unlock();

                System.out.println("线程1 释放了读锁");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t1.start();


        Thread t2 = new Thread(() -> {
            try {
                // 保证读锁先被线程1获得
                Thread.sleep(100);

                System.out.println("线程2 尝试获取写锁");
                writeLock.lockInterruptibly();
                System.out.println("线程2 获取了写锁");
                writeLock.unlock();

            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t2.start();

        Thread t3 = new Thread(() -> {
            try {
                Thread.sleep(200);
                // 等线程2尝试获取锁之后，中断线程2
                t2.interrupt();

            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
```





## 实战

```java
class Outputter1 {  

    private Lock lock = new ReentrantLock();// 锁对象  

 

    public void output(String name) {         

        lock.lock();      // 得到锁  

        try {  

            for(int i = 0; i < name.length(); i++) {  

                System.out.print(name.charAt(i));  

            }  

        } finally {  

            lock.unlock();// 释放锁  

        }  

    }  

}  
```

需要注意的是，用sychronized修饰的方法或者语句块在代码执行完之后锁自动释放，而是用Lock需要我们手动释放锁，所以为了保证锁最终被释放(发生异常情况)，**要把同步代码块放在try内**，释放锁放在finally内！

**得到锁的操作要放在try前面**，因为如果放到try代码块里，若等待过程中响应中断，导致finally的释放锁语句执行，会尝试释放还没有得到的锁，抛出IllegalMonitorStateException异常







## 问题



**ReentrantLock如何实现公平和非公平锁？**

在尝试获取的锁的时候，可能有新线程来尝试获取锁，而这个新线程原本不在同步队列中，如果是非公平锁，则允许新线程尝试获取锁，如果是公平锁，则新线程只能插入到条件队列后面，按照FIFO的顺序去获取锁。







## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970978

