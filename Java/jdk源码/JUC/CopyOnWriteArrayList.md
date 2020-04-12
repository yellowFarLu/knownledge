# CopyOnWriteArrayList



## 概述

"写入时复制(Copy-on-write)" 容器。每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。

（1）多个线程可以并发的读该容器，即时是有线程正在修改这个容器，别的线程也能读（读旧的）。

（2）原理是，当要修改这个容器时，会拷贝一份新的容器，然后修改新的容器，在修改过程中，别的线程不能同时修改，但是可以继续读取旧的容器。当修改完成后，会将原容器的引用指向新的容器。

（3）读的时候不加锁，写的时候加锁。



## 适用场景

适合读请求远大于写请求的场景。

但是 CopyOnWriteArrayList 有其缺陷：

- 内存占用：在写操作时需要复制一个新的数组，使得内存占用为原来的两倍左右；
- 数据不一致：读操作不能读取实时性的数据，因为部分写操作的数据还未同步到读数组中。但是能达到最终一致性。

所以 CopyOnWriteArrayList 不适合内存敏感以及对实时性要求很高的场景。



## 读写分离

写操作在一个复制的数组上进行，读操作还是在原始数组中进行，读写分离，互不影响。（读写并行）

写操作需要加锁，防止并发写入时导致写入数据丢失。（写写串行）

写操作结束之后需要（把原始数组指向新的数组）。





## 源码



### get()

`CopyOnWriteArrayList`的`get`操作是弱一致性的，因为获取的`array`对象可能过时，例如在获取到`array`后由于其它线程`array`指向了新的数组对象则当前线程获取的是旧数组的元素。





### add()

先获取锁，获取`array`数组，拷贝一个新数组，**大小为原数组大小+1**，元素插入到新数组尾部，赋值`array`，最后释放锁。



### addIfAbsent()

获取锁后，再次调用`getArray`得到最新的array数组，检查其是否含有`e`元素，若无则插入其尾端。



### 弱一致性的迭代器

1，之所以说是若一致性，从`iterator()`可以看出，迭代器遍历的是调用该方法时的那个`array`数组快照。只能支持弱一致性，保证最终一致性。
2，`COWIterator`是`ListIterator`。
3，不支持 **增/删/改** 操作。





## 写时复制机制

多个调用者请求相同的资源时，系统会让这些调用者使用同一份资源，即读取同一份资源。

只有当某个调用者想要修复数据的时候，才会给他复制一份资源副本。

这样子做的优点是可以节约资源。





## 实战

```java
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * @author huangy on 2019-10-14
 */
public class CopyOnWriteArrayListDemo {

    public static void main(String[] args) throws Exception {

        CopyOnWriteArrayList<Integer> arrayList = new CopyOnWriteArrayList<>();

        Thread t1 = new Thread(() -> {
            try {
                System.out.println("线程1尝试修改");
                arrayList.add(1);
                System.out.println("线程1修改好了");
                Thread.sleep(3000);
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t1.start();

        Thread t2 = new Thread(() -> {
            try {
                // 睡一下，保证线程1获取到锁
                Thread.sleep(200);

                System.out.println("线程2尝试修改");
                arrayList.add(2);
                System.out.println("线程2修改好了");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t2.start();

        t1.join();
        t2.join();

        System.out.println(arrayList);
    }

}
```









## 问题

**为什么没有ConcurrentArrayList？**

很难去开发一个通用并且没有并发瓶颈的线程安全的List。像ConcurrentHashMap可以在保证线程安全的情况不存在性能瓶颈，因为其使用了分段锁，而List很难做到，List保证线程安全只能锁住整个List，但是这样子就造成性能瓶颈。

<br/>



**CopyOnWriteArrayList是如何保证线程安全的？**

- 当多个写请求同时发生的时候，使用ReentrantLock进行同步，保证一次只有一个线程能进行修改操作

<br/>



## 参考

https://blog.csdn.net/sinat_34976604/article/details/86684926

[写时复制机制](https://blog.csdn.net/ljb825802164/article/details/88528726)

[为什么没有ConcurrentArrayList](https://www.cnblogs.com/heyi-77/p/9835184.html)

