# CPU缓存一致性协议



缓存行：指缓存的最小操作单位。



## 修改原理

当对缓存进行修改的时候，怎么保证线程安全？

假设双核读取，CPU A和CPU B的缓存行中都有共享变量x，修改流程为：

- CPU A 计算完成后发指令需要修改x
- CPU A 对x进行赋值
- CPU A 将x设置为M状态（修改）并通知缓存了x的CPU B, CPU B将本地cache b中的x设置为I状态(无效)
- CPU B重新从内存中读取x

![image-20191110211301250](https://tva1.sinaimg.cn/large/006y8mN6gy1g8t956zozjj30um0u016f.jpg)



















## 参考

[CPU缓存一致性协议](https://www.cnblogs.com/yanlong300/p/8986041.html)