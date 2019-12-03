# 分布式锁



## 背景

分布式应用进行逻辑处理时经常会遇到并发问题。
比如一个操作要修改用户的状态，修改状态需要先读出用户的状态，在内存里进行修改，改完了再存回去。如果这样的操作同时进行了，就会出现并发问题，因为读取和保存状态这两个操作不是原子的。（Wiki 解释：所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch 线程切换。）



![Snip20181118_7](https://ws2.sinaimg.cn/large/006tNbRwgy1fxcbaocwj9j31so0qy796.jpg)

这个时候就要使用到分布式锁来限制程序的并发执行。Redis 分布式锁使用非常广泛，它是面试的重要考点之一，很多同学都知道这个知识，也大致知道分布式锁的原理，但是具体到细节的使用上往往并不完全正确。



## 分布式锁



### 概述

分布式锁本质上要实现的目标就是在 Redis 里面占一个“茅坑”，当别的进程也要来占时，发现已经有人蹲在那里了，就只好放弃或者稍后再试。
占坑一般是使用 setnx(set if not exists) 指令，只允许被一个客户端占坑。先来先占， 用完了，再调用 del 指令释放茅坑。

```
// 这里的冒号:就是一个普通的字符，没特别含义，它可以是任意其它字符，不要误解
> setnx lock:codehole true
OK
... do something critical ...
> del lock:codehole
(integer) 1
```

但是有个问题，如果逻辑执行到中间出现异常了，可能会导致 del 指令没有被调用，这样就会陷入死锁，锁永远得不到释放。

于是我们在拿到锁之后，再给锁加上一个过期时间，比如 5s，这样即使中间出现异常也可以保证 5 秒之后锁会自动释放。

```
> setnx lock:codehole true
OK
> expire lock:codehole 5
... do something critical ...
> del lock:codehole
(integer) 1
```

但是以上逻辑还有问题。如果在 setnx 和 expire 之间服务器进程突然挂掉了，可能是因为机器掉电或者是被人为杀掉的，就会导致 expire 得不到执行，也会造成死锁。
这种问题的根源就在于 setnx 和 expire 是两条指令而不是原子指令。如果这两条指令可以一起执行就不会出现问题。也许你会想到用 Redis 事务来解决。但是这里不行，因为 expire 是依赖于 setnx 的执行结果的，如果 setnx 没抢到锁，expire 是不应该执行的。事务里没有 if-else 分支逻辑，事务的特点是一口气执行，要么全部执行要么一个都不执行。
为了解决这个疑难，Redis 开源社区涌现了一堆分布式锁的 library，专门用来解决这个问题。实现方法极为复杂，小白用户一般要费很大的精力才可以搞懂。如果你需要使用分布式锁，意味着你不能仅仅使用 Jedis 或者 redis-py 就行了，还得引入分布式锁的 library。

![img](https://ws2.sinaimg.cn/large/006tNbRwgy1fxcbdp4haeg30e805kzpw.gif)



为了治理这个乱象，Redis 2.8 版本中作者加入了 set 指令的扩展参数，使得 setnx 和 expire 指令可以一起执行，彻底解决了分布式锁的乱象。从此以后所有的第三方分布式锁 library 可以休息了。

```
> set lock:codehole true ex 5 nx
OK
... do something critical ...
> del lock:codehole
```

上面这个指令就是 setnx 和 expire 组合在一起的原子指令，它就是分布式锁的奥义所在。



### 超时问题

Redis 的分布式锁不能解决超时问题，如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题。因为这时候第一个线程持有的锁过期了，临界区的逻辑还没有执行完，这个时候第二个线程就提前重新持有了这把锁，导致临界区代码不能得到严格的串行执行。
**为了避免这个问题，Redis 分布式锁不要用于较长时间的任务。**如果真的偶尔出现了，数据出现的小波错乱可能需要人工介入解决。

有一个稍微安全一点的方案是为 set 指令的 value 参数设置为一个随机数，释放锁时先匹配随机数是否一致，然后再删除 key，**这是为了确保当前线程占有的锁不会被其它线程释放**，除非这个锁是过期了被服务器自动释放的。 但是匹配 value 和删除 key 不是一个原子操作，Redis 也没有提供类似于delifequals这样的指令，这就需要使用 Lua 脚本来处理了，因为 Lua 脚本可以保证连续多个指令的原子性执行。

```
// 随机数方案
tag = random.nextint()  # 随机数
if redis.set(key, tag, nx=True, ex=5):
    do_something()
    redis.delifequals(key, tag)  # 假想的 delifequals 指令
```

```
// lua脚本
# delifequals
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

但是这也不是一个完美的方案，它只是相对安全一点，因为如果真的超时了，当前线程的逻辑没有执行完，其它线程也会乘虚而入。



### 可重入性

可重入性是指线程在持有锁的情况下再次请求加锁，如果一个锁支持同一个线程的多次加锁，那么这个锁就是可重入的。比如 Java 语言里有个 ReentrantLock 就是可重入锁。Redis 分布式锁如果要支持可重入，需要对客户端的 set 方法进行包装，使用线程的 Threadlocal 变量存储当前持有锁的计数。

```
public class RedisWithReentrantLock {

  private ThreadLocal<Map<String, Integer>> lockers = new ThreadLocal<>();

  private Jedis jedis;

  public RedisWithReentrantLock(Jedis jedis) {
    this.jedis = jedis;
  }

  private boolean _lock(String key) {
    return jedis.set(key, "", "nx", "ex", 5L) != null;
  }

  private void _unlock(String key) {
    jedis.del(key);
  }

  private Map<String, Integer> currentLockers() {
    Map<String, Integer> refs = lockers.get();
    if (refs != null) {
      return refs;
    }
    lockers.set(new HashMap<>());
    return lockers.get();
  }

  public boolean lock(String key) {
    Map<String, Integer> refs = currentLockers();
    Integer refCnt = refs.get(key);
    if (refCnt != null) {
      refs.put(key, refCnt + 1);
      return true;
    }
    boolean ok = this._lock(key);
    if (!ok) {
      return false;
    }
    refs.put(key, 1);
    return true;
  }

  public boolean unlock(String key) {
    Map<String, Integer> refs = currentLockers();
    Integer refCnt = refs
```

以上还不是可重入锁的全部，精确一点还需要考虑内存锁计数的过期时间，代码复杂度将会继续升高。老钱不推荐使用可重入锁，它加重了客户端的复杂性，在编写业务方法时注意在逻辑结构上进行调整完全可以不使用可重入锁。



### 其他指令

除了setnx，还有hsetnx也可以实现分布式锁。hsetnx的参数是key、field、value。如果这个key的field已经存在了，则hsetnx失败。并且指令是原子性的，从而实现分布式锁。









## Watch

Redis 提供了watch 机制，它就是一种乐观锁。有了 watch 我们又多了一种可以用来解决并发修改的方法。

利用watch配合exec指令可以实现乐观锁。



### 原理

- watch 会在**事务开始之前**盯住 1 个或多个关键变量
- 当事务执行时，也就是服务器收到了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被修改了 (包括当前事务所在的客户端)。
- 如果关键变量被其他客户端修改过了，exec 指令就会返回 null 回复告知客户端事务执行失败，这个时候客户端一般会选择重试。



### 使用方式

```java
> watch books
OK
> incr books  # 被修改了
(integer) 1
> multi
OK
> incr books
QUEUED
> exec  # 事务执行失败
(nil)
```

当服务器给 exec 指令返回一个 null 回复时，客户端知道了事务执行是失败的，客户端 (jedis) 通过在 exec 方法里返回一个 null，这样客户端需要检查一下返回结果是否为 null 来确定事务是否执行失败及重试。



### 注意事项

Redis 禁止在 multi 和 exec 之间执行 watch 指令，而必须在 multi 之前盯住关键变量，否则会出错。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9irybheiyj30iw05izkt.jpg" alt="image-20191202230255574" style="zoom:50%;" />

**这个为什么呢？**

假设允许执行multi之后再执行watch，在并发环境下，是可能有问题的，比如下面的场景：

- （1）开始事务前，变量tem的值是1
- （2）这时候执行语句，获取变量tem的值，为1
- （3）另外一个客户端执行变量tem的incr指令，这个tem的值变成2
- （4）当前事务watch住变量tem
- （5）当前事务执行incr变量tem，tem值变为3
- （6）exec执行，并且watch之后，变量tem没有被修改，则事务执行成功
- （7）当前客户端获取事务执行结果，意外发现执行一次incr操作，步长变为2。这就有问题了。





## RedLock



### 背景

说到Redis分布式锁大部分人都会想到：

```java
set(key, value, nx=True, ex=xxx)
```

这种实现方式有2大要点（也是面试概率非常高的地方）：

- 1、set命令要用`set key value px milliseconds nx`
- 2、value要具有唯一性，标识唯一一把锁，释放锁时要验证value值，不能误解锁



不过在集群环境下，这种方式是有缺陷的，它不是绝对安全的。

比如在 Sentinel 集群中，主节点挂掉时，从节点会取而代之，客户端上却并没有明显感知。原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没有来得及同步到从节点，主节点突然挂掉了。然后从节点变成了主节点，这个新的节点内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就批准了。这样就会导致系统中同样一把锁被两个客户端同时持有，不安全性由此产生。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9isl3dsoij30ny0em0x8.jpg" alt="image-20191202232448788" style="zoom:70%;" />

不过这种不安全也仅仅是在主从发生 failover 的情况下才会产生，而且持续时间极短，业务系统多数情况下可以容忍。

为了解决这个问题，Antirez 发明了 Redlock 算法。



### 原理

RedLock一般是三个独立的master节点组成，这个集群用于实现分布式锁。类似于Zookeeper集群。

![image-20191203223819079](https://tva1.sinaimg.cn/large/006tNbRwgy1g9jwv6zz7cj315u0oo778.jpg)



在Redis的分布式环境中，我们假设有N个Redis master。这些节点**完全互相独立，不存在主从复制或者其他集群协调机制**。我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。

现在我们假设有5个Redis master节点，同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:

- 1、获取当前时间（以毫秒为单位）
- 2、依次尝试从5个实例，使用相同的key和**具有唯一性的value**（例如UUID）获取锁。
  - 当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
- 3、客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁消耗了的时间。**当且仅当从大多数**（N/2+1，这里是3个节点）**的Redis节点都取到锁，并且获取锁消耗了的时间 小于 有效时间时，锁才算获取成功**。
- 4、如果取到了锁，key的真正有效时间 等于 有效时间 减去 获取锁消耗了的时间（步骤3计算的结果）。
- 5、如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在**所有的Redis实例上进行解锁**（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）





### RedLock缺点

RedLock严重依赖系统时间，

1. client1 从 ABC 三个节点处申请到锁，DE由于网络原因请求没有到达
2. C节点的时钟往前推了，导致 lock 过期
3. client2 在CDE处获得了锁，AB由于网络原因请求未到达
4. 此时 client1 和 client2 都获得了锁

而这个问题暂时没有解决办法。





## 参考

[redis分布式锁2种实现](https://www.cnblogs.com/chenjianxiang/p/8981134.html)

[watch机制](https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5afc3747f265da0b71567686)

[RedLock原理及源码解析](https://www.jianshu.com/p/7e47a4503b87)

[RedLock真的可行吗](https://blog.csdn.net/chen_kkw/article/details/81433470)

