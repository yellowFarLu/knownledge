# RocketMQ命令



## 启动RocketMQ

参考：

 [安装、启动、关闭RocketMQ](https://xuxiangyang.blog.csdn.net/article/details/102257585?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.pc_relevant_is_cache&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.pc_relevant_is_cache)

[启动、安装RocketMQ(这个也是必要的)](https://zhuanlan.zhihu.com/p/45724441)

注意：

- 启动NameServer的命令 `nohup sh mqnamesrv &` 需要在 ROCKETMQ_HOME/bin 这个路径下面执行。

  - ![image-20201104110011925](https://tva1.sinaimg.cn/large/0081Kckwgy1gkcyii6ysmj30tw08676i.jpg)

- 启动broker的命令 `nohup sh mqbroker &` 需要在 ROCKETMQ_HOME/bin 这个路径下面执行。

  - ![image-20201104110110437](https://tva1.sinaimg.cn/large/0081Kckwgy1gkcyjibbeoj30wc05k0u7.jpg)

- ROCKETMQ_HOME是一个环境变量，如下图：

  - ![image-20201104105807257](https://tva1.sinaimg.cn/large/0081Kckwgy1gkcygdtbzuj30pc06jq45.jpg)

- 通过 \$可以访问环境变量，如 `cd $ROCKETMQ_HOME/bin`





## 创建Topic

执行命令：  `sh mqadmin updateTopic -n 127.0.0.1:9876 -b 127.0.0.1:10911 -t TestTopic`

-n 表示 NameServer的IP及端口

-b 表示 Broker的IP及端口

-t 表示topic的名称

![image-20201104111117330](https://tva1.sinaimg.cn/large/0081Kckwgy1gkcyu1l5grj30rv01xaal.jpg)

**注意，如果是新建了topic，要重启RocktMQ才生效，我这边是重启了电脑。。。**

参考：https://blog.csdn.net/jiahao1186/article/details/86691014







## 查看Topic

sh mqadmin topicList -n 127.0.0.1:9876





## 查看消费情况

格式：`sh mqadmin consumerProgress -g consumeGroupName`

示例：`sh mqadmin consumerProgress -g test`

![image-20201104115904164](https://tva1.sinaimg.cn/large/0081Kckwgy1gkd07rcrfoj30sa0d3wge.jpg)

参考：https://blog.csdn.net/howie_zhw/article/details/52669759



