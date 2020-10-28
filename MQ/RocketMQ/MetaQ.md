# MetaQ



## 概述

MetaQ 是一款分布式、队列模型的消息中间件。分为 Topic 与 Queue 两种模式，Push 和 Pull 两种方式消费，支持严格的消息顺序，亿级别的堆积能力，支持消息回溯和多个维度的消息查询。

MetaQ 主要使用了拉模型，解决了顺序消息和海量堆积问题。



## MetaQ 和 RocketMQ 区别

**RocketMQ是基于MetaQ开源的中间件。**

两者等价，在阿里内部称为 MetaQ 3.0，对外称为 RocketMQ 3.0。





## MetaQ物理机群部署架构

![image-20201028152702982](https://tva1.sinaimg.cn/large/0081Kckwgy1gk52w19k9ij30p40e7q6s.jpg)

- NameServer 集群：MetaQ 1.x 和 MetaQ 2.x 是依赖 ZooKeeper 的，由于 ZooKeeper 功能过重，RocketMQ（即 MetaQ 3.x）去掉了对 ZooKeeper 依赖，采用自己的 NameServer。
- Broker：消息中转角色，负责存储消息，转发消息。
- Consumer：Push Consumer / Pull Consumer。前者向 Consumer 对象注册一个 Listener 接口，收到消息后回调 Listener 接口方法，采用 long-polling 长轮询实现 push；后者主动由 Consumer 主动拉取信息，同 kafka。
- Producer：消息生产者。





## MetaQ 事务消息

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gk52y4ia7lj30o207lmyd.jpg" alt="image-20201028152905560" style="zoom:150%;" />

- 1、发送方向 MQ 服务端发送消息。
- 2、MQ Server 将消息持久化成功之后，向发送方 ACK 确认消息已经发送成功，此时消息为半消息。
- 3、发送方开始执行本地事务逻辑。
- 4、发送方根据本地事务执行结果向 MQ Server 提交二次确认（Commit 或是 Rollback），MQ Server 收到 Commit 状态则将半消息标记为可投递，订阅方最终将收到该消息；MQ Server 收到 Rollback 状态则删除半消息，订阅方将不会接受该消息。
- 5、在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达 MQ Server，经过固定时间后 MQ Server 将对该消息发起消息回查。
- 6、发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
- 7、发送方根据检查得到的本地事务的最终状态再次提交二次确认，MQ Server 仍按照步骤4对半消息进行操作。

事务消息发送对应步骤1、2、3、4，事务消息回查对应步骤5、6、7。













## 参考

[认识MetaQ](http://www.linkedkeeper.com/1609.html)