# PriorityBlockingQueue

`PriorityBlockingQueue` 底层是个最小堆。

当存储元素的数组满时，**插入操作不会阻塞**，而是扩容数组；

数组空时 **take** 方法会阻塞。







## 参考

https://blog.csdn.net/sinat_34976604/article/details/88048226