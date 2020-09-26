# CountDownLatch

CountDownLatch允许一个或多个线程等待其它线程完成操作。

主要的功能就是通过await()方法来阻塞线程，然后等待计数器减少到0了，再唤起阻塞的线程继续执行代码逻辑；即你想要某些线程等待另一些线程执行完再执行，就可以使用CountDownLatch。



state在初始化时被设置，每次被等待任务线程执行完会将state减一，直到state变为0，代表所有被等待任务线程执行完毕，此时被阻塞线程将全部被唤醒，以及之后的线程都将不被阻塞，CountDowbLatch通过此来实现一个线程等待另一组线程执行完再执行的功能。



## await()

当发现state大于0的时候，线程被放入同步队列，挂起，阻塞。



## countDown()

当state==0的时候，表示可以唤起阻塞的线程，所有线程往下执行。但是这里有个问题，就是怎么保证所有线程都唤醒呢？

就是第一次调用doReleaseShared唤醒的线程后，该线程被唤醒，并且会再次执行releaseShared()方法，从而唤起剩余的线程。



## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970948