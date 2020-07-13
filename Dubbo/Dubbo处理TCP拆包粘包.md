# Dubbo处理TCP拆包粘包

dubbo在处理tcp的粘包和拆包时是借助InternalDecoder的buffer缓存对象来缓存不完整的dubbo协议栈数据，等待下次inbound事件，合并进去。所以说在dubbo中解决TCP拆包和粘包的时候是通过buffer变量来解决的。

- 拆包：
  - 情况：将一个dubbo协议的数据拆成多个tcp包
  - 解决方法：将不完整的数据保存在缓存中，等待下一次输入，然后读取到完整的数据，提交给用户线程
- 粘包：
  - 情况：将一个或者多个dubbo协议的数据放到tcp包中
  - 解决方法：将数据读取到缓存对象中，然后进行逐个解析每一个消息





## 参考

[Dubbo处理TCP拆包粘包](https://blog.csdn.net/zh_ka/article/details/84735879)

[Dubbo拆包粘包（更好）](https://blog.csdn.net/u013076044/article/details/89279699?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

