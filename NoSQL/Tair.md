# Tair



## 概念

Tair是由淘宝网自主开发的Key/Value结构数据存储系统，在淘宝网有着大规模的应用。在登录淘宝、查看商品详情页面的时候，都在直接或间接地和Tair交互。

Tair有四种引擎：mdb, rdb, kdb和ldb。分别基于四种开源的key/value数据库：memcached, Redis, Kyoto Cabinet和leveldb。**Tair可以让你更方便地使用这些KV数据库**。比如Redis没有提供sharding操作，如果有多个Redis Server，你需要自己写代码实现sharding，Tair帮你封装了这些。



## 功能特性



### Version支持

Tair中的每个数据都包含版本号，版本号在每次更新后都会递增。这个特性有助于防止由于数据的并发更新导致的问题。

比如，系统有一个value为“a,b,c”，A和B同时get到这个value。A执行操作，在后面添加一个d，value为 “a,b,c,d”。B执行操作添加一个e，value为”a,b,c,e”。如果不加控制，无论A和B谁先更新成功，它的更新都会被后到的更新覆盖。

Tair无法解决这个问题，但是引入了version机制避免这样的问题。还是拿刚才的例子，A和B取到数据，假设版本号为10，A先更新，更新成功 后，value为”a,b,c,d”，与此同时，版本号会变为11。当B更新时，由于其基于的版本号是10，服务器会拒绝更新，从而避免A的更新被覆盖。 B可以选择get新版本的value，然后在其基础上修改，也可以选择强行更新。





### 原子计数器

Tair从服务器端支持原子的计数器操作，这使得Tair成为一个简单易用的分布式计数器。

Tair有以下优点：

- 统一的API。无论底层使用何种引擎，上层的API是一样的。
- Tair将集群操作封装起来，解放了开发者。淘宝内部在使用Tair时，一般都是双机房双集群容错，利用invalid server保证两个集群间的一致性，这些对于开发者都是透明的。



### Item支持

Tair还支持将value视为一个item数组，对value中的部分item进行操作。比如有个key的value为[1,2,3,4,5]，我们可以只获取前两个item，返回[1,2]，也可以删除第一个item，还支持将数据删除，并返回被删除的数据，通过这个接口可以实现一个原子的分布式 FIFO的队列。





## 使用场景



### 非持久化

- 数据可以以key/value的形式存储
- 数据可以接受丢失
- 访问速度要求很高
- 单个数据大小不是很大，一般在KB级别
- 数据量很大，并且有较大的增长可能性
- 数据更新不频繁



### 持久化

- 数据可以以key/value的形式存储
- 数据需要持久化
- 单个数据大小不是很大，一般在KB级别
- 数据量很大，并且有较大的增长可能性
- 数据的读写比例较高





## 实现原理



### 架构

Tair是Master/Slave结构。Config Server管理Data Server节点、维护Data Server的状态信息；Data Server负责数据存储，按照Config Server的指示完成数据复制和迁移工作，并定时给Config Server发送心跳信息。Config Server是单点，采用一主一备的方式保证可靠性。



**tair网络拓扑**：

![image-20201030182838405](https://tva1.sinaimg.cn/large/0081Kckwgy1gk7jdnmu8xj30x00gok2q.jpg)



**tair架构：**

![image-20201030182925239](https://tva1.sinaimg.cn/large/0081Kckwgy1gk7jeddcduj30f908b774.jpg)







### 数据的分布问题

分布式系统需要解决的一个重要问题便是决定数据在集群中的分布策略，好的分布策略应该能将数据均衡地分布到所有节点上，并且还应该能适应集群节点的变化。Tair采用的对照表方式较好地满足了这两点。

对照表的行数是一个固定值，这个固定值应该远大于一个集群的物理机器数，由于对照表是需要和每个使用Tair的客户端同步的，所以不能太大，不然同步将带来较大的开销。我们在生产环境中的行数一般为1023 。



### 对照表简介

下面我们看对照表是怎么完成数据的分布功能的，为了方便，我们这里假设对照表的行数为6。最简单的对照表包含两列，第一列为hash值，第二列为负责该 hash值对应数据的dataserver节点信息。比如我们有两个节点192.168.10.1和192.168.10.2，那么对照表类似：

```properties
0 192.168.10.1
1 192.168.10.2
2 192.168.10.1
3 192.168.10.2
4 192.168.10.1
5 192.168.10.2
```

（hash值指的是key的hash值？ 假如函数固定1023，那么仅支持1023个key？）



**对照表如何适应节点数量的变化**

我们假设新增了一个节点——192.168.10.3，当configserver发现新增的节点后，会重新构建对照表。构建依据以下两个原则：

1. 数据在新表中均衡地分布到所有节点上。
2. 尽可能地保持现有的对照关系。















## 参考

[深入理解Tair](https://www.jianshu.com/p/ccb17daed766)