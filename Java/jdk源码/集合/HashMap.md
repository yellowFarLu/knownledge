# HashMap



## put插入元素

- 找到桶的位置  
  - 使用(n - 1) & hash，这也说明了为什么数组长度是2的整数次幂
- 插入到链表尾部
- 如果是红黑树，则插入到红黑树中
- 插入完毕后，判断需不需要进行扩容



## 扩容处理

若size大于threshold的时候，就会进行扩容。

进行扩容，会伴随着一次重新hash分配，并且会遍历hash表中所有的元素。

![image-20191015102021139](https://tva1.sinaimg.cn/large/006y8mN6gy1g7yo64l5n5j30kj0ce439.jpg)

rehash是非常耗时的。在编写程序中，要尽量避免resize。









## 参考

https://www.cnblogs.com/zhangyinhua/p/7698642.html#_label1

https://cloud.tencent.com/developer/article/1338265

https://juejin.im/post/5d412a5be51d45620e0b99af