# Dubbo序列化协议

dubbo 支持 hession、protocol buffer等序列化协议。

hessian 是其默认的序列化协议。





## Hessian 的数据结构

Hessian 的对象序列化机制有 8 种原始类型：

- 原始二进制数据
- boolean
- 64-bit date（64 位毫秒值的日期）
- 64-bit double
- 32-bit int
- 64-bit long
- null
- UTF-8 编码的 string

另外还包括 3 种递归类型：

- list for lists and arrays
- map for maps and dictionaries
- object for objects

还有一种特殊的类型：

- ref：用来表示对共享对象的引用。





## 问题

**为什么 PB 的效率是最高的？**

内容根据数据类型动态变化，所以序列化后的数据量体积小。因为体积小，传输起来带宽和速度上会有优化。











## 参考

[dubbo支持的通讯协议、序列化协议](https://www.jianshu.com/p/0cbac7c13311)