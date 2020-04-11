# CPU打满了



## 常用步骤



### 步骤

- 使用top指令，看看那个进程占用CPU比较高
  - ![image-20191229160807943](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnp2h2vzj31z40pk45w.jpg)
- 使用top -Hp PID查看该进程内的线程执行情况（左边第一列是线程ID）
  - H表示设置线程模式，也就是**展示线程的信息**
  - p表示展示某个进程的线程信息
  - ![image-20191229160955623](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnqxvz8hj31z40mojyt.jpg)
- 将这个线程ID转换成二进制
  - 通过语句  `printf "%x\n" TID`
  - ![image-20191229161127817](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnsjaj7tj30ji01wwel.jpg)
- 使用jstack找该进程的对应线程的执行情况
  - 通过语句  `jstack 20956 | grep -C30 4e6a --color`
  -  ![image-20191229161312495](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnuck5b9j31z40rwn7j.jpg)
  - 可以看到，由于等待锁（估计是synchronize的轻量级锁空转），导致线程占用CPU变高





### 可能情况

- 代码里面产生了死循环
- 频繁发生Full GC
  - 一次请求生成了很多小对象，并且这些小对象被数组对象强引用着，随着请求的增多，内存逐渐上升，引发MiniGC，后面内存依旧不足，引发了FullGC。







## 实战一

有一天，有个支付相关的服务由于CPU超过了阈值（k8s可以设置CPU的阈值），导致重启，因此我去查看原因。

- 首先，使用top命令观察CPU使用情况，该服务的CPU使用率达到了0.6
- 从监控系统观察内存使用情况，发现这段时间JVM内存的老年代大小变小了很多，于是推测是频繁Full GC导致CPU使用率很高。
- 使用jstat -gc查看GC次数，Full GC的确频繁，估计存在内存泄漏
- 使用jmap -dump把快照dump下来
- 使用MAT的dominator tree进行分析，发现某个类型的对象占了35%，这个对象用于统计请求的调用链，由于一次请求的调用链可能有很多个接口，每个接口都有各自的调用信息（耗时、IP地址、入参、出参等等），因此占用字节非常多。
- 把代码里面这个统计关掉，问题了得到解决了

总结来说：CPU升高——》发现Full GC频繁——》存在内存泄漏——》MAT分析——》定位代码







## 参考

[如何排查CPU占用过高以及常见的几种情况](https://blog.csdn.net/coderpopo/article/details/80332496)

[线上故障排查积累](https://m.toutiaocdn.com/group/6760056297392964103/?app=news_article&timestamp=1586569157&req_id=20200411093917010130037138022AE6B9&group_id=6760056297392964103&tt_from=mobile_qq&utm_source=mobile_qq&utm_medium=toutiao_android&utm_campaign=client_share)

