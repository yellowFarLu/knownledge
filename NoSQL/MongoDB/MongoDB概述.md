# MongoDB概述



## 概述

MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。

在高负载的情况下，添加更多的节点，可以保证服务器性能。

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

![image-20200726130515926](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh4ajkbf6xj30vu092di9.jpg)





## 主要特点

- MongoDB 是一个文档型数据库
- 支持分布式存储，提高读写性能
- Mongodb中的Map/reduce主要是用来对数据进行批量处理和聚合操作。





## MongoDB和Redis的区别

- 数据存储方面：
  - Redis将数据存储在内存中，而MongoDB将数据存储在文件中
- 操作方面：
  - Redis的操作比较简单，而MongDB的增删查改操作支持添加很多条件
- 性能方面：
  - Redis是纯内存操作，性能比MongoDB更高
- 可靠性：
  - Redis采用AOF + 快照的方式进行持久化
  - MongoDB采用binLog方式持久化
- 适用场景：
  - Redis适合小数据量，高性能访问的场景
  - MongoDB适合海量数据存储，提高访问效率的场景
- 一致性
  - Redis支持事务，但是仅支持操作在事务中顺序执行（我认为这其实都不能算是事务）
  - MongoDB不支持事务
- 数据分析
  - Redis不支持数据分析
  - MongoDB支持Map-Reduce





## 细节

- Mongo对应的库、表不存在，则会自动创建对应的库、表





## 参考

[MonogoDB和Redis的区别](https://www.cnblogs.com/java-spring/p/9488227.html)



