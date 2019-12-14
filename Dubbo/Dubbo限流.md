# Dubbo限流

Dubbo的限流作用于提供方。可以在高并发的情况，保证系统的稳定性、安全性。避免让系统被流量压垮，导致整体服务不可用。





## 实践

**提供者**添加类似配置

```java
<dubbo:service
  interface="com.huang.yuan.api.service.DemoService"
  ref="demoServiceImpl"
  version="1.0"
  delay="5000"
  filter="tps">
  <dubbo:parameter key="tps" value="1"/>
  <dubbo:parameter key="tps.interval" value="1000"/>  
</dubbo:service>
```

添加filter及dubbo paramter，表示每tps.interval的时间间隔内，能执行tps个请求。



消费方同时发出10个请求

```java
@Test
public void testdada() throws Exception {
  for (int i = 0; i < 10; i++) {
    new Thread(()->{
      demoService.test("huangyuan");
    }).start();
  }

  Thread.sleep(1100000);
}
```



提供方限制流量，多余的请求将抛出异常：

![image-20191214131831430](https://tva1.sinaimg.cn/large/006tNbRwgy1g9w6hytt2ej31sg0mm49q.jpg)









## 源码



### 令牌桶算法

Dubbo默认使用令牌桶算法实现限流。某段时间内，桶里面只能放进n个令牌，然后来一个请求就减少一个令牌，如果桶里面的令牌没有了，则不能继续执行请求。



限流通过com.alibaba.dubbo.rpc.filter.TpsLimitFilter实现。

首先从URL中获取配置的限制，限制由两个参数组成，表示在interval毫秒的时间内允许执行rate个调用。

默认周期是60秒，不限制速率。

```java
public boolean isAllowable() {

  // 获取现在的时间
  long now = System.currentTimeMillis();

  // 当经过了interval时间间隔
  if (now > lastResetTime + interval) {

    // 重新设置token令牌的个数
    token.set(rate);

    // 从现在开始，经过interval的时间
    lastResetTime = now;
  }

  // 获取令牌的值
  int value = token.get();

  boolean flag = false;

  // 使用CAS实现乐观锁
  while (value > 0 && !flag) {

    // 能够执行请求，则令牌减一
    flag = token.compareAndSet(value, value - 1);

    value = token.get();
  }

  return flag;
}
```

