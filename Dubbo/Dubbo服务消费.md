**消费方调用过程中，Dubbo究竟做了什么？**



![4181053155-5be6d41891122_articlex](https://ws1.sinaimg.cn/large/006tNbRwgy1fxz867tz01j30h20b6t9o.jpg)



**消费方**

- 在Directory中找出本次集群中的全部invokers

- 在Router中,将上一步的全部invokers挑选出满足条件的invokers

- 在LoadBalance中,将上一步的能正常的执行invokers中,根据配置的负载均衡策略,挑选出需要执行的invoker



**提供方**

- 找到对应exporter
- expoter中包含了invoker，利用invoker反射调用方法





