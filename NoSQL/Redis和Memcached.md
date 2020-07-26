# Redis和Memcached区别



## 持久化

- redis支持快照、AOF两种持久化方式
- Memcached不支持持久化操作，所以的数据都以in-momory的形成存储。



## 数据类型

- **Redis**除了简单的key-value数据类型，同时还提供了list、hash、set、zset数据结构的存储
- **Memcached**只能使用key-value形式存储和访问数据



## 网络IO模型

- **Mongo**是多线程，阻塞IO模型
  - 由主线程进行 accept 连接，然后针对每一个连接创建一个线程进行处理
- **Redis**是单线程的IO多路复用模型



## 内存管理机制

- redis是按需分配内存
- memcached预先向操作系统申请一块内存，并且分成多份小内存块，当客户端请求到来时，获取最适合的一块小内存进行存储数据。
  - 优点是不会造成内存碎片，但是会造成空间浪费





 

















## 参考

[Redis、Mongo、Memcached](https://segmentfault.com/a/1190000012834166)