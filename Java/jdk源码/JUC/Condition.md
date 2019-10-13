# Condition

ReentrantLock 的newCondition():最终调用的是AQS中的内部类**ConditionObject**，它实现了Condition接口。AQS有一个同步队列；ConditionObject维护了一个等待队列，二者节点的类型都是AQS.Node。



## AQS和Condition队列不同

- Condition队列是一个单向链表，而Sync队列是一个双向链表
- Sync队列的头节点一直是个空节点，而Condition队列初始化时，头结点就开始持有等待线程了。



## await()

首先执行await的线程此时已经获取到了**独占锁**（成功改变同步状态），创建该线程的节点放到ConditionObject的条件队列尾部，之后release唤醒同步队列的下一个线程，之后就LockSupport.park挂起当前线程。



##signal()

signal主要逻辑就是将条件队列的第一个节点加入到同步队列尾部



## awaitNanos()

基本逻辑与await相同，加入了时间限制条件。返回值代表剩余时间，大于0说明中断发生。



## 参考

https://blog.csdn.net/sinat_34976604/article/details/80970980