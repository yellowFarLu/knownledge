# HashTable

 Hashtable同样是基于哈希表实现的，同样每个元素是一个key-value对，其内部也是通过单链表解决冲突问题，容量不足（超过了阀值）时，同样会自动增长。



## 与HashMap区别

- Hashtable中，key和value都不允许出现null值。HashMap中，null可以作为键，这样的键只有一个。可以有一个或多个键所对应的值为null。
- Hashtable 中的方法是Synchronize的，而HashMap中的方法在缺省情况下是非Synchronize的
- HashTable直接使用对象的hashCode。而HashMap重新计算hash值。
- HashTable在不指定容量的情况下的默认容量为11，而HashMap为16，Hashtable不要求底层数组的容量一定要为2的整数次幂，而HashMap则要求一定为2的整数次幂。
- Hashtable扩容时，将容量变为原来的2倍加1，而HashMap扩容时，将容量变为原来的2倍。







## 参考

[HashTable和HashMap](https://www.cnblogs.com/williamjie/p/9099141.html)

