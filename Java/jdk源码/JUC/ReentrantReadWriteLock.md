# ReentrantReadWriteLock

简单了解一下ReentrantReadWriteLock

- 读锁是个共享锁，写锁是个独占锁。读锁同时被多个线程获取，写锁只能被一个线程获取。读锁与写锁不能同时存在。
- 一个线程可以多次重复获取读锁和写锁
- 锁升级：不支持升级。获取读锁的线程去获取写锁的话会造成死锁。
- 重入数：state表示重入数，前16位表示读锁的总数量，后16位表示写锁的总数量
- 支持公平与非公平两种模式

![image-20191120144237017](https://tva1.sinaimg.cn/large/006y8mN6gy1g94i22dam2j30rw04xdis.jpg)



AQS维护了一个int值，表示同步状态，那么如何用它来即表示读锁又表示写锁呢？读锁我们是允许多个线程同步运行的，我们还允许重入，那么拿什么来记录每个线程读锁的重入数？

针对上面两个问题，**对于同步状态status，高16位表示所有线程持有的读锁总数，低16位为一个线程的写锁总数，包括重入**。

![image-20191120141141117](https://tva1.sinaimg.cn/large/006y8mN6gy1g94h5x6m8gj30r80eajvv.jpg)







## 源码



### 写锁

#### 写饥饿

表示由于读锁一直占用资源，导致线程无法正常获得写锁。

解决方案：如果当前有线程获取写锁，那么资源已经被前面获取读锁的线程占有了，那么只要等这些已有读锁的线程，就好了。新的获取读锁线程，就阻塞一下。从而避免写饥饿



#### 获取写锁

1，持有写锁的线程可以再次获取写锁。

2，持有写锁的线程可以获取读锁 

3、同一个线程，获取了读锁，再获取写锁，会导致死锁



### 锁降级

```java
/**
 * 锁降级实例
 * @author huangy on 2019-11-20
 */
public class LockDowngrada {

    private ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    private ReentrantReadWriteLock.ReadLock readLock;

    private ReentrantReadWriteLock.WriteLock writeLock;

    private volatile boolean update;

    {
        readLock =  readWriteLock.readLock();
        writeLock = readWriteLock.writeLock();
    }

    public void processData() {

        readLock.lock();

        if (!update) {

            // 必须先释放读锁，才能获取写锁
            readLock.unlock();

            // 锁降级从写锁获取到开始
            writeLock.lock();

            try {
                if (!update) {
                    // 假装在准备数据

                    // 准备好了，设置标志位
                    update = true;
                }

                /*
                 * 因为下面todo的地方需要进行数据处理，所以这里需要先获取读锁，为什么这里要先获取读锁呢？
                 * 由于这里获取了读锁，由于不支持锁升级，那么任意线程都不能再获取写锁，因此在todo处理数据的时候，是线程安全的。
                 * 如果这里不获取读锁，那么在writeLock.unlock();释放掉写锁后，其他线程可以获取写锁，并且修改数据，那么当前线程是看不到最新的数据的。
                 */
                readLock.lock();

            } finally {
                writeLock.unlock();
            }

            // 锁降级完成，写锁降级为读锁
        }

        try {
            // todo 假装处理数据
        } finally {
            readLock.unlock();
        }
    }

}
```







## 问题

**锁降级中，释放写锁之前，为什么要获取读锁？**

![image-20191120165520273](https://tva1.sinaimg.cn/large/006y8mN6gy1g94lw5fmg8j30s406kmzd.jpg)



**为什么不支持锁升级**

![image-20191120165619754](https://tva1.sinaimg.cn/large/006y8mN6gy1g94lx6zk7hj30rz047jso.jpg)

只要当前有线程获取了读锁，任意线程都无法获取写锁。



## 参考

https://blog.csdn.net/sinat_34976604/article/details/80971628