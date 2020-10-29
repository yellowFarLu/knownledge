# TDDL



## 概述

淘宝根据自身业务需求研发了 TDDL（Taobao Distributed Data Layer）框架，主要用于解决分库分表场景下的访问路由（持久层与数据访问层的配合）以及异构数据库之间的数据同步，它是一个基于集中式配置的 JDBC DataSource 实现，具有分库分表、Master/Salve、动态数据源配置等功能。

![image-20201029112034140](https://tva1.sinaimg.cn/large/0081Kckwgy1gk61dulaquj30mv0cnwg4.jpg)

TDDL 部署在 IBatis、MyBatis 或其它 ORM 框架之下，JDBC Driver 之上，整个中间件实现了 JDBC 规范。TDDL 是 JDBC 或者持久框架与底层 JDBC 驱动交互的桥梁。





## TDDL 架构

TDDL 其实主要可以划分为3层架构，分别是 Matrix 层、Group 层和 Atom 层。

![image-20201029112138951](https://tva1.sinaimg.cn/large/0081Kckwgy1gk61ez3g95j30o30cgq7s.jpg)

- Matrix 层在于分库分表路由，SQL 语句的解释、优化和执行，事务的管理规则的管理，各个子表查询出来结果集的Merge 等。
- Group 层实现了数据库的 Master/Salve 模式的写分离逻辑。
- Atom 层（TAtomDataSource）实现真正和物理数据库交互，提供数据库配置动态修改能力。











## 参考

[TDDL github](https://github.com/alibaba/tb_tddl/wiki/TDDL-dynamic-datasource-%E5%85%A5%E9%97%A8%E4%B8%8E%E4%BD%BF%E7%94%A8)

[TDDL原理](http://www.linkedkeeper.com/1608.html)

