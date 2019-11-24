# CyclicBarrier



## 源码



###**await()**

每一个线程调用await()方法，则将count减一（count表示当前阻塞的线程数）。

当count等于0的时候，唤醒条件队列上的所有线程。

当count不等于0，则要阻塞当前线程（通过Condition实现阻塞，把当前线程放入条件队列）。





### 失败尝试

Generation是CyclicBarrier的一个私有内部类，他只有一个成员变量broken来标识当前的barrier是否已“损坏”：

**如果任何线程在等待时被中断**（等待中的中断分两种，分别对应两种处理REINTERRUPT与THROW_IN，只有后者才是这里所指的），**则线程调用await()方法将抛出 BrokenBarrierException 异常**。

对于失败的同步尝试，CyclicBarrier 使用了一种要么全部   要么全不 (all-or-none) 的破坏模式。



## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970979