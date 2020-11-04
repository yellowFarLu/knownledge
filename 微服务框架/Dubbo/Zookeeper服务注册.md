#Zookeeper服务注册

在Dubbo注册中心的整体架构设计中，Zookeeper上的节点设计如图：

![image-20191019192154089](https://tva1.sinaimg.cn/large/006y8mN6gy1g83qasdn7hj317g0de0yu.jpg)



**/dubbo**

这是Dubbo在Zookeeper上创建的根节点。



**/dubbo/com.foo.BarService**

这是服务节点，代表dubbo的一个服务



**/dubbo/com.foo.BarService/providers**

这是服务提供者的根节点，其子节点代表了每一个服务的真正提供者



**/dubbo/com.foo.BarService/consumers**

这是服务消费者的根节点，其子节点代表了每一个服务的真正消费者





## 注册流程



### 服务提供者

服务提供者在启动的时候，会在Zookeeper上面写入/dubbo/com.foo.BarService/providers（只是举例子）下面新建一个临时节点，并且向该节点写入自己的URL地址，这就代表com.foo.BarService这个服务的其中一个提供者。



### 服务消费者

服务消费者在启动的时候，读取并订阅Zookeeper上的/dubbo/com.foo.BarService/providers下的所有子节点。并**提取出所有提供者的URL**地址来作为该服务的地址列表，并且选择其中一个提供者进行调用。

同时服务消费者还会在/dubbo/com.foo.BarService/consumers下面创建一个临时节点，并且写入自己的URL。这就代表com.foo.BarService这个服务的其中一个消费者。





## 监控中心

监控中心是Dubbo服务治理的重要部分。其需要知道服务的所有提供者、消费者。因此监控中心在启动的时候，会通过/dubbo/com.foo.BarService节点获取所有消费者、提供者的URL，并且注册watcher监听其子节点的变化。