# SynchronousQueue



## 原理

SynchronousQueue 不是一个真正的队列，其主要功能不是存储元素，而且维护一个排队的线程清单，这些线程等待把元素加入或者移除队列。每一个线程的入队（出队）操作必须等待另一个线程的出队（入队）操作。

当许多线程进行入队时，并且没有线程来取数据，那么这些入队线程都会阻塞，SynchronousQueue 也会形成一个队列，队列中的每个节点有存储的数据，同时也有阻塞的线程（方便后续唤醒），因此SynchronousQueue 中即存储的数据，也存储了线程。把SynchronousQueue 当做普通的队列来用肯定是不行的，其SynchronousQueue 的主要意义在于适合传递性的工作，充当一个传球手的角色，只有两边同时准备好了的情况下才了交接“数据”。



SynchronousQueue有两种模式：

- 公平模式
  所谓公平就是遵循先来先服务的原则，因此其内部使用了一个FIFO队列 来实现其功能。
- 非公平模式
  最开始我以为非公平性和前面的重入锁可能会差不多，结果后来才知道，差别大了去了。
  SynchronousQueue 中的非公平模式是默认的模式，其内部使用栈来实现其功能，也就是 后来的先服务,这个似乎也太不公平了。



















## 参考

https://blog.csdn.net/u014634338/article/details/78419445

https://blog.csdn.net/sinat_34976604/article/details/88317045