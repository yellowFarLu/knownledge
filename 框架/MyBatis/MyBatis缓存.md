# MyBatis缓存



## 背景

一次数据库会话中，可能会执行的重复的查询语句，极短时间内重复查询，结果往往是一样的，但是造成了数据库资源的浪费。

为了解决这个问题，MyBatis使用了一级缓存。MyBatis在sqlSession对象（表示会话的对象）中建立了一个简单的缓存，将每次查询的结果缓存起来。如果下次有完全一样的查询，直接返回结果。





## 一级缓存

### 特点

- 一级缓存默认是开启的

- 作用域是session级别，每个sqlSession各自拥有一级缓存，各自隔离。

- 第一次查询结果换以key value的形式存起来，如果有相同的key进来，直接返回value，这样有助于减轻数据库的压力。

- 缓存的key格式如下：

  - ```java
    cache key: id + sql + limit + offset 
    ```

- 当commit或者rollback的时候会清除缓存，并且当执行insert、update、delete的时候也会清除缓存。







## 二级缓存


### 特点

- 一个namespace（mapper.xml）就会有一个缓存
- 不同的sqlSession之间的二级缓存是共享的
- 实现二级缓存的时候，MyBatis要求返回的POJO必须是可序列化的，也就是要求实现Serializable接口
  - 因为二级缓存的数据不一定都是存储到内存中，它的存储介质多种多样
  - 如果存储在内存中的话，实测不序列化也可以的。
- 一般为了避免出现脏数据，所以我们可以在每一次的insert | update | delete操作后都进行缓存刷新











## 参考

[MyBatis一级缓存和二级缓存](https://www.jianshu.com/p/5515640d14fe)

[MyBatis-系统缓存及简单配置介绍](https://www.cnblogs.com/jian0110/p/9387941.html)

[MyBatis-SqlSession运行原理](https://www.cnblogs.com/jian0110/p/9452592.html)

