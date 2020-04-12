# ArrayBlockingQueue

ArrayBlockingQueue的内部是通过一个可重入锁ReentrantLock和两个Condition条件对象来实现阻塞。

**添加元素和删除元素使用通一把锁。**





## 添加元素

- add()：添加元素到队列，满了，则抛出异常
- offer()：尝试添加队列到队列，无论成功、失败都直接返回
- put()：添加队列到队列，队列满则会一直阻塞
- 所有插入元素操作，都会先获取锁





## 删除元素

- remove()：队列头出队，队列空了，则抛出异常
- poll()：队列头出队，队列空了，则返回null
- take()：队列头出队，队列空了，则阻塞
- element()：获取队列头，队列空了，则抛出异常
- peek()：获取队列头，空则返回null