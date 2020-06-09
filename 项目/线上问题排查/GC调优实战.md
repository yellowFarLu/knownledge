# GC调优实战

- 接收到告警：“响应时间突然飚高”

- 猜测是发生了Full GC，并且耗时比较久，导致程序没能及时响应

- jstat -gccause pid查看上一次GC的耗时，居然有2000毫秒

- 之前使用的是Parallel Old垃圾收集器，会停顿用户线程比较久

  - 改成使用-XX:+UseConcMarkSweepGC 改成CMS收集器

- Full GC降至800毫秒，应该还是有优化空间的

- 继续查看GC日志，发现Remark重新标记阶段耗时比较多

- 使用CMSScavengeBeforeRemark，在Remark阶段之前先进行一次YGC，可以大大减少remark时间

  - CMSScavengeBeforeRemark该参数需要衡量利弊，假如年轻代的对象很少，那就触发了一次无用的YGC

- Full GC降至300毫秒



## 扩展

从头到尾模拟故障排除过程
（1）CPU打满： 检查哪个线程cpu高——>jstack日志——>gc频繁——>内存占用高——>原因是记录接口调用链保存在内存中，没有释放，完成内存泄露
（2）JVM调优

（3）内存溢出



## 参考

[GC调优实战](https://www.cnblogs.com/onmyway20xx/p/6626567.html)

