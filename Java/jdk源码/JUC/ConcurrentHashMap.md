# ConcurrentHashMap



##  JDK 1.7的实现

ConcurrentHashMap由Segement数组和HashEntry数组组成。Segement是一种可重入锁，在ConcurrentHashMap里面扮演锁的角色，HashEntry用于存储键值对数据。

一个ConcurrentHashMap里面包含一个Segement数组。Segement的结构和HashMap相似，是一种数组和链表结构。一个segment中包含一个HashEntry数组，每个HashEntry是一个链表结构的元素。

每个segment守护着HashEntry数组中的元素，每当对HashEntry数组的元素进行修改时，必须先获取与其对应的segment锁。



Segement + HashEntry （分段锁技术）

- 含有多个segment，根据hashCode计算出来元素在哪个segment，只把要存入元素的segment上锁。
- ConcurrentHashMap使用ReentrantLock保证线程安全。Segment继承自ReentrantLock。

![image-20191121221914794](https://tva1.sinaimg.cn/large/006y8mN6gy1g960vj45tuj31ia0rsapo.jpg)

与HashMap一样，ConcurrentHashMap也是一个基于散列的Map，但它使用了一个完全不同的加锁策略来提供更高的并发性和可扩展性。ConcurrentHashMap并不是每个方法都在同一个锁上同步，而是使用一种粒度更细的加锁机制来来实现更大程度的共享，这种机制叫做**分段锁**。在这种机制中，任意数量的读线程可以并发访问Map，执行读取操作的线程和执行写操作的线程可以并发的访问Map，并且一定数量的写线程可以并发的修改Map。

### 优点

在并发环境下将实现更高的吞吐量，而在单线程环境下只损失非常小的性能。

### 缺点

- 计算桶的位置的时间更长了，因为经历了两次hash计算
- 当某个段很大时，分段锁的性能会下降。







## JDK 1.8实现

Node + CAS + Synchronized

- 取消了segment分段的形式，**每个链表为单位进行加锁**，表头元素作为锁。进一步减少了并发冲突的概率，提高了并发度。
- 当元素个数小于8时，采用链表形式。单个数大于等于8时，采用红黑树结构。（为了避免单个列表太长，导致查询时间复杂度为O(n)的情况）
- CAS实现乐观锁，保证更新值的线程安全
- synchronized更新桶元素的线程安全（如插入、清空等操作）



### 优点

- 相对于jdk.17，jdk1.8根据桶为粒度加锁，并发读更高了。
- 避免计算位置的2次hash运算。
- JAVA7之前ConcurrentHashMap主要采用锁机制，在对某个Segment进行操作时，将该Segment锁定，不允许对其进行非查询操作，而在JAVA8之后采用CAS无锁算法，这种乐观操作在完成前进行判断，如果符合预期结果才给予执行，对并发操作提供良好的优化。（不会阻塞线程）









## 和HashTable的区别

HashTable是一个线程安全的类，它使用synchronized来锁住整张哈希表来实现线程安全，即每次锁住整张表让线程独占，相当于所有线程进行读写时都去竞争一把锁，导致效率非常低下。

ConcurrentHashMap可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，允许多个修改操作并发进行，其关键在于使用了锁分离技术。它使用了多个锁来控制对hash表的不同部分进行的修改。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的Hashtable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行，提高了并发度。





## 和HashMap的区别

ConcurrentHashMap的key、value都不能为null

HashMap的key、value都可以为null，存储在index=0的桶上面





## 源码



### 1.8源码分析



#### 扩容

- 当元素个数到达阈值时，可以进行扩容
- 如果当前线程是第一个进行扩容的线程，则初始化新数组，**容量为原来的2倍**。
- 从旧数组末尾进行rehash，会把链表头当做锁
- 如果期间又有一个新的线程进来了，则分配一些桶给该线程，让该线程并行扩容
- 如果当前线程处理完它的扩容任务了，可以继续领取别的任务进行处理
- 扩容完成，切换新数组





#### ln和hn链表

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdqpyzp6vxj312b0ge799.jpg)

在桶迁移的过程中，如果元素的新位置 等于原来的位置，则放入ln链表；

如果元素的新位置 等于 原来的位置 + 数组旧容量，则把该元素放入hn链表；

如果没有这两条链表的话，每次迁移一个元素都要重新计算迁移后桶的位置。所以通过这样子进行优化，减少了扩容过程中计算桶位置的次数





#### get()

 方法总共会遇到如下3种情形:

- 非扩容情况下：遇到 get 操作，通过计算 h &  (n - 1) 定位到具体的hash桶位置，如果数组上的hash桶就是目标元素则直接返回即可 。如果当前hash桶为普通Node节点链表，则使用普通链表方式去遍历该链表查找目标元素 。如果定位到的hash桶为TreeBin节点，则根据TreeBin内部维护的红黑树锁来确定具体采用哪种方式遍历查找元素，如果如果红黑树锁的状态为 写锁 / 等待写锁，则使用链表方式去遍历查找目标元素，而反之红黑树锁状态为 无锁 / 读锁，则使用红黑树方式去遍历查找目标元素，红黑树锁只存在 读-写 互斥而不存在 写-写 互斥 。

- 集合正在扩容并且当前桶正在迁移中：遇到 get 操作，在扩容过程期间会形成 hn 和 ln链，形成这两条中间链是使用的类似于复制引用的方式，也就是说 ln 和 hn 链是复制出来的，而非原hash桶的链表剪切过去的，所以原来 hash 桶上的链表并没有受到影响，**因此从迁移开始到迁移结束这段时间都是可以正常访问原数组 hash 桶上面的链表**，具体访问方式同上面的 非扩容情况。

- 集合扩容还未结束但是当前hash桶已经迁移完成：遇到 get 操作，每迁移完一个hash桶后当前hash桶的位置都会被替换成 ForwardingNode 节点，**遇到 get 操作时直接将查找操作转发到新的数组上去**，也就是直接到新数组上面查找目标元素，具体的查找方式依旧跟 非扩容情况 相同。





#### put()

**put 方法总共会遇到如下4种情形**

- 集合还未初始化：进行集合的初始化操作，该操作会将设置一个全局的初始化标识 sizeCtl = -1，当其他线程检测到 sizeCtl 的值为 -1 时就会使用 Thread.yield() 方法让出 CPU 资源，让初始化线程能够更快完成初始化操作，同时也保证了只能有一条线程对集合进行初始化 。

- 该位置的值为 null：为了避免线程安全问题，使用CAS方式将元素直接设置到该数组位置上 。

- 该位置的值为 ForwardingNode 节点：说明此时集合在扩容中，并且当前定位到的节点的hash桶已经迁移完毕，此时执行put操作的**线程会优先加入到扩容大军里面去，加快扩容速度，待扩容完成后再继续插入新元素 **

- 该位置已经有其他值：
  - 先锁住位于数组上的头结点 。如果节点类型是普通链表节点，使用**尾插法在末尾拼接上新的节点 **。
  - 如果节点类型是 TreeBin 节点，调用 TreeBin 的 putTreeVal 方法 。putTreeVal 方法具体做法为，如果目标位置为 null，则直接添加进去元素，如果目标位置已经有值，则返回旧值，根据 onlyIfAbsent 属性决定是否覆盖该红黑树上面的旧值 。



#### putIfAbsent

- 如果key对应的映射不存在，则添加到map中，返回null
- 如果key对应的映射已经存在了，则返回已经存在的value











### 1.7源码分析



#### get()

先对key进行hash运算，定位到segment，然后再定位到HashEntry，最后逐个比较HashEntry链表中的元素，如果key相等，则返回该元素。





#### put()

put()方法需要对共享变量进行写入，为了保证线程安全，必须加锁。

put方法首先定位到segment，然后在segment里进行插入操作。插入操作分为两步：

- 判断segment中的HashEntry数组是否需要进行扩容
- 定位插入元素的位置，将元素插入到HashEntry数组中





### 是否需要扩容

判断数组长度是否超过阈值，如果超过，则对数组进行扩容。

值得一提的是，segment的扩容比HashMap更恰当，因为HashMap是插入元素之后，判断是否达到阈值，如果达到了就进行扩容，但是很有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容，占用内存空间。







### 如何扩容

在扩容的时候，首先会创建一个容量是原来两倍的数组，然后将原来数组的元素进行再散列后，插入到新数组中。

为了高效，ConcurrentHashMap不会对整个容器进行扩容，而只对某个Segement进行扩容。



### size()

为了提高统计元素个数的效率，ConcurrentHashMap首先尝试2次不加锁计算元素个数，具体做法是把把每个segment的元素个数加起来，得到整个ConcurrentHashMap的元素个数。如果相加过程中，容器发生了变化，就会重算，计算失败超过2次，那么就会采用加锁的方式计算所有segment的元素总数。

ConcurrentHashMap通过modCount判断自身是否发送变化，因为put()、remove()、clean()方法操作元素前都会将modCount加1，那么在统计元素size的前后，比较modCount是否发生变化，从而得知容器是否发生变化。





## 问题

**ConcurrentHashMap是如何在保证并发安全的同时提高性能？**

- 在jdk1.8中以桶为粒度实现分段锁，提高了并发读，从而提高性能
- 使用CAS实现乐观锁 + synchronize实现线程安全

<br/>



**ConcurrentHashMap是如何让多线程同时参与扩容？**

ConcurrentHashMap从数组尾部开始扩容，给当前线程分配一部分桶，让其进行扩容。

如果此时有新的线程尝试进行更新操作，那么ConcurrentHashMap会让这个线程先去帮忙进行扩容，把一部分桶分配给这个线程。

这些线程将当前的桶扩容完毕后，还可以继续领取新的桶进行扩容。

多线程扩容完毕后，这些线程可以继续原来的操作，比如说插入新元素。

<br/>



**桶迁移中以及迁移后如何处理存取请求？**

桶迁移中：到原数组进行查找

桶迁移后：请求就直接转发到扩容后的新数组去了

<br/>



**ConcurrentHashMap如何保证并发扩容是线程安全的？**

- 每个线程负责一部分桶
- 线程对当前进行迁移的桶加上synchronize锁，防止多线程对同一个桶并发修改
- 把元素放到新位置的时候，利用volatile保证可见性

<br/>



**为什么ConcurrentHashMap在1.8中废弃了分段锁的实现方式**

- 以桶为单位进行加锁，并发度更高
- 分段锁需要进行2次hash运算来计算元素的位置，比较慢

<br/>



**为什么ConcurrentHashMap的get()操作不用加锁？**

在jdk1.7中，ConcurrentHashMap的get操作通过UNSAFE.getObjectVolatile来获取Segment及HashEntry，这保证了内存的可见性。因为，即时不用加锁，也能保证get()方法读取到的是最新的数据。



<br/>



## 参考

[ConcurrentHashMap1.8 - 扩容详解](https://blog.csdn.net/ZOKEKAI/article/details/90051567)

[ConcurrentHash源码解析](https://blog.csdn.net/sinat_34976604/article/details/80971620)

[ConcurrentHashMap源码详解](https://blog.csdn.net/ZOKEKAI/article/details/90741157)

[jdk1.8中ConcurrentHashMap做的改进](https://www.cnblogs.com/everSeeker/p/5601861.html)

[ConcurrentHashMap原理解析](https://www.cnblogs.com/huangjuncong/p/9478505.html)

