# RocketMQ事务消息



## 事务消息

以购物场景为例，张三购买物品，账户扣款 100 元的同时，需要保证在下游的会员服务中给该账户增加 100 积分。由于数据库私有，所以导致在实际的操作过程中会出现很多问题，比如先发送消息，可能会因为扣款失败导致账户积分无故增加，如果先执行扣款，则有可能因服务宕机，导致积分不能增加。

![image-20191106234000098](https://tva1.sinaimg.cn/large/006y8mN6gy1g8oqww7smtj311i0iu461.jpg) 无论是先发消息还是先执行本地事务，都有可能导致出现数据不一致的结果。

事务消息的本质就是为了解决**解决本地事务执行与消息发送的原子性问题。**



**RocketMQ 事务消息设计则主要是为了解决 Producer 端的消息发送与本地事务执行的原子性问题。**
 RocketMQ 的设计中 broker 与 producer 端的双向通信能力，使得 broker 天生可以作为一个事务协调者存在；而 RocketMQ 本身提供的存储机制，则为事务消息提供了持久化能力；RocketMQ 的高可用机制以及可靠消息设计，则为事务消息在系统在发生异常时，依然能够保证事务的最终一致性达成。



## RocketMQ 事务消息设计

**事务消息作为一种异步确保型事务， 将两个事务分支通过 MQ 进行异步解耦，RocketMQ 事务消息的设计流程同样借鉴了两阶段提交理论，**整体交互流程如下图所示：



![image-20191106234158655](https://tva1.sinaimg.cn/large/006y8mN6gy1g8oqyxrs1rj31o60j2gxr.jpg)

- 事务发起方（即生产者）首先发送 prepare 消息到 MQ。
- 事务发起方（即生产者）在发送 prepare 消息成功后执行本地事务。
- 根据本地事务执行结果发送 commit 或者是 rollback 给 MQ。 
  - 如果消息是 rollback，MQ 将删除该 prepare 消息不进行下发。
  - 如果消息是 commit，MQ 将会把这个消息发送给 consumer 端。
- 如果执行本地事务过程中，执行端挂掉，或者超时，导致 MQ 收不到任何的消息（不知道是该 commit 还是该 rollback），RocketMQ 会定期扫描消息集群中的事务消息，这时候发现了某个 prepare 消息还不知道该怎么处理，它会向消息发送者确认，所以消息发送者需要实现一个 check 接口，RocketMQ 会根据消息发送者设置的策略来决定是 rollback 还是继续 commit。这样就保证了消息发送与本地事务同时成功或同时失败。
- Consumer 端的消费成功机制由 MQ 保证。

![image-20191106234353314](https://tva1.sinaimg.cn/large/006y8mN6gy1g8or0xrf0aj30za0jadm8.jpg)



## 具体实现

在具体实现上，如下图所示：

![image-20191106234448356](https://tva1.sinaimg.cn/large/006y8mN6gy1g8or1vz104j30u00uy15o.jpg)

Half Topic 对应队列中存放着 prepare 消息，Operation Topic 对应的队列则存放了 prepare message 对应的 commit/rollback 消息，消息体中则是 prepare message 对应的 offset，服务端通过比对两个队列的差值来找到尚未提交的超时事务，进行回查。

从用户侧来说，**用户需要分别实现本地事务执行以及本地事务回查方法，因此只需关注本地事务的执行状态即可；**而在 service 层，则对事务消息的两阶段提交进行了抽象，同时针对超时事务实现了回查逻辑，通过不断扫描当前事务推进状态，来不断反向请求 Producer 端获取超时事务的执行状态，在避免事务挂起的同时，也避免了 Producer 端的单点故障。
 而在存储层，RocketMQ 通过 Bridge 封装了与底层队列存储的相关操作，用以操作两个对应的内部队列，用户也可以依赖其他存储介质实现自己的 service，RocketMQ 会通过 ServiceProvider 加载进来。

从上述事务消息设计中可以看到，**RocketMQ 事务消息较好的解决了事务的最终一致性问题，事务发起方仅需要关注本地事务执行以及实现回查接口给出事务状态判定等实现，**而且在上游事务峰值高时，可以通过消息队列，避免对下游服务产生过大压力。









## 参考

[RocketMQ事务消息](https://www.jianshu.com/p/5248abc8b8ed)