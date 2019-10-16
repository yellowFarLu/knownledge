# PriorityBlockingQueue

`PriorityBlockingQueue` 底层是个最小堆。

当存储元素的数组满时，**插入操作不会阻塞**，而是扩容数组；

数组空时 **take** 方法会阻塞。



## offer

插入元素。插入堆的底部，从进行上浮操作，如果父节点比当前节点小，则当前节点向上移动。直到找到适合的位置插入。



## 扩容

当插入元素的时候，如果判断数组满了，则会进行扩容。扩容的时候，为什么要释放锁，这里其实是一个优化，具体理由看代码

```java
    private void tryGrow(Object[] array, int oldCap) {

        /*
         * 先释放锁，因为后面计算扩容大小，生成新数组。这个线程安全性只需要volatile+cas就可以实现了
         * 并不需要使用lock，如果使用lock的话，那么插入、删除这些操作都将被阻塞，实际上算扩容大小，生成新数组的时候，
         * 这些操作在原数组上，还是可以继续执行的。只不过真正进行数据迁移的时候，就要阻塞插入、删除操作了。
         */
        lock.unlock();

        Object[] newArray = null;

        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {

            try {
                /*
                 * 计算新的容量大小
                 * 如果一开始容量比较小，则扩容快一点。这样子是为了频繁扩容。
                 * 不然容量比较小的时候，一下子容量又不够用了，导致频繁扩容，就会消耗大量CPU及内容。
                 * 后面等到容量比较大了，比如说64，就扩容1.5倍，降低扩容速度
                 */
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) :
                                       (oldCap >> 1));


                if (newCap - MAX_ARRAY_SIZE > 0) {
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }

                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];

            } finally {
                // 保证下一次扩容
                allocationSpinLock = 0;
            }
        }

        if (newArray == null)
            // 证明其他线程正在初始化数组，那么当前线程先让出CPU
            Thread.yield();

        lock.lock();
        // 拷贝数组内容，这部分才是真正要进行加锁的
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```









## 参考

https://blog.csdn.net/sinat_34976604/article/details/88048226

https://www.cnblogs.com/huangjuncong/p/9229832.html