# 缓存和DB中数据一致性



## 背景

**缓存与数据库不一致**

![image-20191108231125166](https://tva1.sinaimg.cn/large/006y8mN6gy1g8r1brjo9sj30ie094tbk.jpg)



如上图，发生的场景也是，写后立刻读：

（1+2）先一个写请求，淘汰缓存，写数据库

（3+4+5）接着立刻一个读请求，读缓存，cache miss，读从库，写缓存放入数据，以便后续的读能够cache hit（主从同步没有完成，缓存中放入了旧数据）

（6）最后，主从同步完成

导致的**结果**是：旧数据放入缓存，即使主从同步完成，后续仍然会从缓存**一直**读取到旧数据。

可以看到，加入缓存后，导致的不一致影响时间会很长，并且最终也不会达到一致。



## 方案

![image-20191108231456000](https://tva1.sinaimg.cn/large/006y8mN6gy1g8r1ff1x5yj30i40dwwis.jpg)

如上图所述，在并发读写导致缓存中读入了脏数据之后：

（6）主从同步

（7）通过工具订阅从库的binlog，这里能够最准确的知道，从库数据同步完成的时间

*画外音：本图画的订阅工具是DTS，可以是cannal，也可以自己订阅和分析binlog*

（8）从库执行完写操作，**向缓存再次发起删除**，淘汰这段时间内可能写入缓存的旧数据



（这样子还是在短时间内可能存在缓存和DB不一致，但是能达到最终一致性）





## 参考

[缓存一致性](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961356&idx=1&sn=8fa6a57d128a3255a049bee868a7a917&chksm=bd2d0dd08a5a84c62c1ac1d90b9f4c11915c9e6780759d167da5343c43445759bce0f16de395&mpshare=1&scene=23&srcid=1108Wczoo8Z6tSJZLtW0IxKy&sharer_sharetime=1573224956114&sharer_shareid=1c062d5c810b024acf7d4936fe834135#rd)

