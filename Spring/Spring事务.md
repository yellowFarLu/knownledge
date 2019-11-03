# Spring事务

事务管理是应用系统开发中必不可少的一部分。Spring 为事务管理提供了丰富的功能支持。Spring 事务管理分为编码式和声明式的两种方式。编程式事务指的是通过编码方式实现事务；声明式事务基于 AOP,将具体业务逻辑与事务处理解耦。声明式事务管理使业务代码逻辑不受污染, 因此在实际使用中声明式事务用的比较多。声明式事务有两种方式，一种是在配置文件（xml）中做相关的事务规则声明，另一种是基于@Transactional 注解的方式。注释配置是目前流行的使用方式，因此本文将着重介绍基于@Transactional 注解的事务管理。



## Spring事务隔离级别

其实就是事务的隔离级别

- DEFAULT
  - 使用数据库默认的事务隔离级别.
- 读未提交（read uncommited） 
  - 脏读，不可重复读，虚读都有可能发生
- 已提交读 （read commited）
  - 避免脏读。但是不可重复读和虚读有可能发生
- 可重复读 （repeatable read） 
  - 避免脏读和不可重复读.但是虚读有可能发生.
- 串行化的 （serializable）
  - 所有事务请求串行执行



## 实现原理

利用Spring Aop实现的。

当一个方法使用了@Transactional注解，在运行时，JVM为该Bean创建一个代理对象，并且在调用该方法的时候进行使用TransactionInterceptor拦截，在方法执行之前会开启一个事务，然后执行方法的逻辑。

方法执行成功，则提交事务。如果执行方法中出现异常，则回滚事务。





## Spring事务传播

事务传播行为指**当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行。**

例如：methodA事务方法调用methodB事务方法时，methodB是继续在调用者methodA的事务中运行呢，还是为自己开启一个新事务运行，这就是由methodB的事务传播行为决定的。

Spring定义了七种传播行为：



### PROPAGATION_REQUIRED

如果上下文中存在一个事务，则加入到当前事务。如果没有事务则开启一个新的事务。
可以把事务想像成一个胶囊，在这个场景下方法B用的是方法A产生的胶囊（事务）。

![image-20191103205605011](https://tva1.sinaimg.cn/large/006y8mN6gy1g8l5bf2b9sj30jy0heacu.jpg)

举例有两个方法

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
     methodB();
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
}
```

调用methodA方法时，因为当前上下文不存在事务，所以会开启一个新的事务。当执行到methodB时，methodB发现当前上下文有事务，因此就加入到当前事务中来。



### PROPAGATION_SUPPORTS

如果上下文中存在一个事务，则加入当前事务。如果没有事务，则非事务的执行。

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
 methodB();
}

// 事务属性为SUPPORTS
@Transactional(propagation = Propagation.SUPPORTS)
public void methodB() { 
}
```

单纯的调用methodB时，methodB方法是非事务的执行的。

当调用methdA时，methodB则加入了methodA的事务中，事务地执行。



### PROPAGATION_MANDATORY

如果上下文中已经存在一个事务，则加入当前事务。如果没有一个活动的事务，则抛出异常。

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
 methodB();
}

// 事务属性为MANDATORY
@Transactional(propagation = Propagation.MANDATORY)
public void methodB() {
}
```

当调用methodA时，methodB则加入到methodA的事务中，事务地执行。

当单独调用methodB时，因为当前没有一个活动的事务，则会抛出异常。



### PROPAGATION_REQUIRES_NEW

会开启一个新的事务。如果已经存在一个事务，则先将这个存在的事务挂起。

使用PROPAGATION_REQUIRES_NEW,需要使用 JtaTransactionManager作为事务管理器。

![image-20191103211914127](https://tva1.sinaimg.cn/large/006y8mN6gy1g8l5zhvfnuj30og0h241m.jpg)

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
  doSomeThingA();
  methodB();
  doSomeThingB();
}


// 事务属性为REQUIRES_NEW
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
}
```

当调用

```java
public static void main {  
	methodA();
} 
```

相当于调用

```java
public static void main { 
    TransactionManager tm = null;
    try{
        //获得一个JTA事务管理器
        tm = getTransactionManager();
        tm.begin();//开启一个新的事务
        Transaction ts1 = tm.getTransaction();
        doSomeThing();
        tm.suspend();//挂起当前事务
        try{
            tm.begin();//重新开启第二个事务
            Transaction ts2 = tm.getTransaction();
            methodB();
            ts2.commit();//提交第二个事务
        } Catch(RunTimeException ex) {
            ts2.rollback();//回滚第二个事务
        } finally {
            //释放资源
        }
        //methodB执行完后，恢复第一个事务
        tm.resume(ts1);
        doSomeThingB();
        ts1.commit();//提交第一个事务
    } catch(RunTimeException ex) {
        ts1.rollback();//回滚第一个事务
    } finally {
        //释放资源
    }
}
```

在这里，我把ts1称为外层事务，ts2称为内层事务。从上面的代码可以看出，ts2与ts1是两个独立的事务，互不相干。Ts2是否成功并不依赖于 ts1。如果methodA方法在调用methodB方法后的doSomeThingB方法失败了，而methodB方法所做的结果依然被提交。而除了 methodB之外的其它代码导致的结果却被回滚了



### PROPAGATION_NOT_SUPPORTED

总是非事务地执行，并挂起任何存在的事务。

使用PROPAGATION_NOT_SUPPORTED,也需要使用JtaTransactionManager作为事务管理器。

![image-20191103212250075](https://tva1.sinaimg.cn/large/006y8mN6gy1g8l638jpoqj30nu0hgad0.jpg)



### PROPAGATION_NEVER

总是非事务地执行，如果存在一个活动事务，则抛出异常。





### PROPAGATION_NESTED

![image-20191103212418620](https://tva1.sinaimg.cn/large/006y8mN6gy1g8l64rq3csj30p80jq0x4.jpg)

嵌套执行事务。

```java
@Transactional(propagation = Propagation.REQUIRED)
void methodA(){
  doSomeThingA();
  methodB();
  doSomeThingB();
}

@Transactional(propagation = Propagation.NEWSTED)
void methodB(){
  ……
}
```

如果单独调用methodB方法，则按REQUIRED属性执行。如果调用methodA方法，相当于下面的效果：

```java
public static void main(){
    Connection con = null;
    Savepoint savepoint = null;
    try{
        con = getConnection();
        con.setAutoCommit(false);
        doSomeThingA();
        savepoint = con2.setSavepoint();
        try{
            methodB();
        } catch(RuntimeException ex) {
            con.rollback(savepoint);
        } finally {
            //释放资源
        }
        doSomeThingB();
        con.commit();
    } catch(RuntimeException ex) {
        con.rollback();
    } finally {
        //释放资源
    }
}
```

当methodB方法调用之前，调用setSavepoint方法，保存当前的状态到savepoint（保存节点）。如果methodB方法调用失败，则恢复到之前保存的状态。但是需要注意的是，这时的事务并没有进行提交，如果后续的代码(doSomeThingB()方法)调用失败，则回滚包括methodB方法的所有操作。

嵌套事务一个非常重要的概念就是内层事务依赖于外层事务。外层事务失败时，会回滚内层事务所做的动作。而内层事务操作失败并不会引起外层事务的回滚。

只有外部事务提交了，内部事务才会提交。





## 参考

[事务传播](https://blog.csdn.net/soonfly/article/details/70305683)