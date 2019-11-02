# CopyOnWriteArrayList

利用写时复制来实现的一个线程安全的ArrayList类，任何对内部数组的更改操作都被锁保护，更改操作都是在拷贝的新数组上进行。

适合读请求远大于写请求的场景。



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





## 问题

**为什么没有ConcurrentArrayList？**

如果有ConcurrentArrayList，当ConcurrentArrayList插入或者删除元素的时候，如果为了保证线程安全，将阻塞其他线程，那么将极大影响到性能。因为ArrayList的插入、删除涉及到元素移动，会很慢，而元素移动这段时间将一直造成阻塞其他线程的请求。





## 参考

https://blog.csdn.net/sinat_34976604/article/details/86684926