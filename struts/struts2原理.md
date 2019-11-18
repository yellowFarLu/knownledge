# struts2



## 系统架构

![image-20191118231947318](https://tva1.sinaimg.cn/large/006y8mN6gy1g92lrlregqj30u00xqqrl.jpg)

- FilterDispatcher是整个Struts2的调度中心，也就是MVC中的C（控制中心），根据ActionMapper的结果来决定是否处理请求，如果ActionMapper指出该URL应该被Struts2处理，那么它将会执行Action处理，并停止过滤器链上还没有执行的过滤器。
- ActionMapper 会判断这个请求是否应该被Struts2处理，如果需要Struts2处理，ActionMapper会返回一个对象来描述请求对应的ActionInvocation的信息。
- ActionProxy，它会创建一个ActionInvocation实例，位于Action和xwork之间，使得我们在将来有机会引入更多的实现方式，比如通过WebService来实现等。
- ConfigurationManager是xwork配置的管理中心，可以把它看做struts.xml这个配置文件在内存中的对应
- struts.xml，是开发人员必须光顾的地方。是Stuts2的应用配置文件，负责诸如URL与Action之间映射关系的配置、以及执行后页面跳转的Result配置等。
- ActionInvocation：真正调用并执行Action，它拥有一个Action实例和这个Action所依赖的拦截器实例。ActionInvocation会按照指定的顺序去执行这些拦截器、Action以及相应的Result。
- Interceptor(拦截器)：是Struts2的基石，类似于JavaWeb的Filter，拦截器是一些无状态的类，拦截器可以自动拦截Action，它们给开发者提供了在Action运行之前或Result运行之后来执行一些功能代码的机会
- Action：用来处理请求，封装数据。





## 原理

- FilterDispatcher拦截请求
- FilterDispatcher会将请求转发给ActionMapper。ActionMapper负责识别当前的请求是否需要Struts2做出处理
- 如果需要Struts2处理，ActionMapper会通知FilterDispatcher，需要处理这个请求，FilterDispatcher会停止过滤器链以后的部分，（这也就是为什么，FilterDispatcher应该出现在过滤器链的最后的原因）。然后建立一个ActionProxy实例，这个对象会代理Action的运行过程。
- ActionProxy从ConfigurationManager中知道该运行那个action，然后创建ActionInvocation对象
  - Action完整的调用过程都是由ActionInvocation对象负责
- 执行拦截器
- 执行Action的execute方法
- 通过结果生成页面
- ActionInvocation对象倒序执行拦截器
- 向客户端执行结果



















## 参考

[struts2原理](https://blog.csdn.net/wjw0130/article/details/46371847)