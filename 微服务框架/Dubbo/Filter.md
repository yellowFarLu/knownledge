### 概要

Dubbo的Filter机制，是专门为服务提供方和服务消费方调用过程进行拦截设计的，每次远程方法执行，该拦截都会被执行。这样就为开发者提供了非常方便的扩展性，比如实现调用前进行处理的功能。



### 使用步骤

自定义FIlter实现方法很简单，分为3个步骤：
 1、自定义Filter，必须继承com.alibaba.dubbo.rpc.Filter接口
 2、在resources目录下添加纯文本文件META-INF/dubbo/com.alibaba.dubbo.rpc.Filter，内容写成 xxx=xxx.xxx.xxxFilter
 3、在dubbo配置xml中配置filter，这里有2种配置方式：

（1）<dubbo:provider filter="xxxFilter" />  提供方 被调用时，执行的filter

（2） <dubbo:consumer filter="xxxFilter" />  消费方调用方法时，会触发filter



### 执行顺序

- 默认filter链，先执行原生的filter，再依次执行自定义的filter，继而回溯到原点。
- 如果配置了 filter="filter1,filter2"，则会默认执行filter1—》filter2—》原生filter。注意配置多个filter的时候，不能有空格
- 如果配置了 filter="filter1,-filter2"，也就是在filter前面加-号，则不会执行该filter
- 同时配置了loadBalance、filter。执行顺序为 loadBalance ——》filter



### 疑惑

- 自定义的filter，设置了@Activate(group = Constants.PROVIDER)是无效的，因为源码没有处理。
- 自定义的filter，@Activate(order = 1)是无效的，源码也没有处理



