# AQS



## 概述

在java.util.concurrent包（下称j.u.c包）中，大部分的同步器（例如锁，屏障等等）都是基于AbstractQueuedSynchronizer（下称AQS类）这个类来构建的；它是个抽象的FIFO队列式的同步器，AQS定义了一套多线程访问共享资源的同步器框架。



J.U.C包下的同步器的实现都主要依赖于以下几个功能：

- 内部同步状态的**管理**，比如说更新和检查操作
- 调用线程在获取同步状态时阻塞，以及在其他线程改变这个同步状态时唤醒阻塞的线程

而AQS就实现了以上功能，供其他同步器使用。



### acquire和release

所有同步器都有两个基本方法，acquire，release。

acquire操作阻塞调用的线程，直到release改变同步状态，才允许其继续执行。、

release操作则是通过某种方式改变同步状态，使得一或多个被acquire阻塞的线程继续执行。（不用同步器命名不同Lock.lock，Semaphore.acquire，CountDownLatch.await...)



### 同步状态

AQS用单个32位int值 state 来表示同步状态



### 阻塞

利用LockSupport来阻塞/唤醒线程





### 同步队列

无法获取执行资格的线程会构建一个节点Node加入队列。

AQS中使用的是CLH队列：CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理。

当前线程如果获取同步状态失败时，AQS则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒，使其再次尝试获取同步状态。

![image-20191013165205523](https://tva1.sinaimg.cn/large/006y8mN6gy1g7wo92hbssj30sw0akq5e.jpg)

关于队列的节点Node：5条属性，分别是waitStatus 、prev、next、thread、nextWater。

- 其中 prev，next用于构建双向链表，thread 指向节点对应的线程。

- waitStatus表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。
  - CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，其结点的waitStatus为CANCELLED，进入该状态后的结点将会被删除。
  - SIGNAL：值为-1，表明该节点之后有节点在阻塞，当该节点被唤醒或删除后会必须唤醒其后继节点
  - CONDITION：值为-2，与Condition相关，该标识的结点处于条件队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从条件队列转移到同步队列中，等待获取同步锁。
  - PROPAGATE：值为-3，与共享模式相关。这个标记只在一处用到，即共享锁的doReleaseShared方法，该方法在同步队列头节点状态waitStatus=0 时将其设为PROPAGATE ，本身doReleaseShared是为了释放后继节点的，但是当头节点状态为0，我们不知道有没有后继节点，所以就采用这种方式，将头节点标记为PROPAGATE，意味着将共享锁的释放传递下去，它与setHeadAndPropagate方法有关，可以去看我的CountDownLatch文章里对该方法的分析。





## 源码流程



### 获取排他锁

#### acquire

```java
public final void acquire(int arg) {
    // 尝试获取锁；
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

当前线程会尝试去获取锁，如果获取不到锁，那么当前线程就被加入到同步队列，尝试自旋获取同步状态，再次失败则被挂起。这时如果又有线程获取失败了，则加入到同步队列末尾。



#### release

之前获取锁的线程执行完了，释放同步状态，唤醒同步队列中的队列头结点的后继线程，后继线程再次尝试获取同步状态。





**举例说明整个acquire流程：**

现在有一个线程N正在执行，同步队列为空。

A线程tryAcquire失败，addWaiter创建了一个空的Node作为Head，它的next指向A线程的Node，随后进入acquireQueued，再次尝试tryAcquire因为N线程可能已经执行完了。

若失败tryAcquire则调用shouldParkAfterFailedAcquire，此时将头Head的同步状态由0变为SIGNAL返回false，回到acquireQueued里再次循环，还是尝试再次tryAcquire，失败调用shouldParkAfterFailedAcquire，由于Head的waitStatus为SIGNAL返回true，进入parkAndCheckInterrupt，将A线程阻塞；若又一线程B也失败，它将会将A 的waitStatus变为SIGNAL，排在A的后面阻塞着；

执行的线程完成了，release释放同步状态，唤醒同步队列里的线程；接着上面的逻辑，首先N线程tryRelease成功，取出head节点执行unparkSuccessor，将head节点waitStatus重置为0，取出head.next也就是A，LockSupport.unpark唤醒A线程，逻辑回到了A阻塞的地方也就是acquireQueued的for循环里，再次尝试tryAcquire（在这里可能被插队），成功，将A设为head，将原先为空的head的next指针清除以便GC回收；





### 获取共享锁





### state操作的线程安全性

**同步状态state在AQS中是volatile修饰的，确保了可见性，循环CAS确保了符合操作的原子性**，二者结合保证了state操作的线程安全性。

一个线程通过getState得到了state的最新值，计算出了remaining大于0，于是CAS赋值，那么在这些个操作过程中，state很可能已经被改变，CAS会失败，如何处理？循环再来一次呗，再次获取state最新值进行计算不就可以了。Doug Lea大神在设计JUC框架时采用了这种非锁式的不排他的方法来确保线程安全。



## 问题



**AQS是如何唤醒下一个线程的？**

当之前的线程释放锁了，那么从同步队列头获取一个节点，调用LockSupport.unpark的系统调用去唤醒该线程




## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970943





