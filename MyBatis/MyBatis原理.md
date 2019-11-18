# MyBatis原理



## 原理

- MyBatis启动时，解析mybatis的配置文件，并且从指定路径下解析mapper.xml配置文件
  - 把每条sql语句映射成MappedStatement
- 然后把MappedStatement存放到Configuration的一个mappedStatements属性中（mappedStatements是一个HashMap），key为namespace + id，value为MappedStatement
- 当要执行sql语句的时候，从mappedStatements这个map中通过id找到MappedStatement
- 获取MappedStatement对应sql语句、查询参数
- 查看一级缓存中有没有数据，有则直接返回
- 缓存没有数据，则查询数据库
  - 通过调用原生的jdbc方法，执行sql语句，获取到结果，删除旧缓存
- 把结果放到一级缓存，返回结果





## 接口方式

思考一个问题，通常的Mapper接口，我们可以不实现方法，却可以使用。这是为什么呢？答案很简单 `动态代理`



开始之前介绍一下MyBatis初始化时对接口的处理：

- 当判断解析到接口时，会创建此接口对应的MapperProxyFactory对象，存入HashMap中，key = 接口的class对象，value = 此接口对应的MapperProxyFactory对象。
- 当执行sql的时候，通过MapperProxyFactory生成动态代理实例（JDK动态代理）
- 通过代理实例，将请求转发给invoker方法
- 在invoke()方法中，最终SqlSession中的方法执行sql语句























## 参考

[MyBatis原理分析](https://blog.csdn.net/weixin_43184769/article/details/91126687)