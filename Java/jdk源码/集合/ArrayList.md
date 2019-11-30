# ArrayList



## 概览

- ArrayList可以存放null。
- ArrayList本质上就是一个elementData数组。
- ArrayList区别于数组的地方在于**能够自动扩展大小**，其中关键的方法就是gorw()方法。
- ArrayList中removeAll(collection c)和clear()的区别就是removeAll可以删除批量指定的元素，而clear是全是删除集合中的元素。
- ArrayList由于本质是数组，所以它在数据的查询方面会很快，而在插入删除这些方面，性能下降很多，有移动很多数据才能达到应有的效果
- ArrayList实现了RandomAccess，所以在遍历它的时候推荐使用for循环。



## 源码



### gorw()

ArrayList默认扩容1.5倍。

添加元素时使用 ensureCapacityInternal() 方法来保证容量足够，如果不够时，需要使用 grow() 方法进行扩容，新容量的大小为 `oldCapacity + (oldCapacity >> 1)`，也就是旧容量的 1.5 倍。

扩容操作需要调用 `Arrays.copyOf()` 把原数组整个复制到新数组中，这个操作代价很高，因此最好在创建 ArrayList 对象时就指定大概的容量大小，减少扩容操作的次数。



### 删除元素

需要调用 System.arraycopy() 将 index+1 后面的元素都复制到 index 位置上，该操作的时间复杂度为 O(N)，可以看出 ArrayList 删除元素的代价是非常高的。



### RandomAccess

RandomAccess 是一个标志接口，表明实现这个这个接口的 List 集合是支持快速随机访问的。也就是说，实现了这个接口的集合是支持 **快速随机访问** 策略的。

同时，官网还特意说明了，如果是实现了这个接口的 **List**，那么使用for循环的方式获取数据会优于用迭代器获取数据。

```java
for (int i=0, n=list.size(); i < n; i++)
 list.get(i);

for (Iterator i=list.iterator(); i.hasNext(); )
 i.next();
```



### Fail-Fast

modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。

在进行序列化或者迭代等操作时，需要比较操作前后 modCount 是否改变，如果改变了需要抛出 ConcurrentModificationException。



### 序列化

ArrayList 基于数组实现，并且具有动态扩容特性，因此保存元素的数组不一定都会被使用，那么就没必要全部进行序列化（只会序列化有元素的位置）。

保存元素的数组 elementData 使用 transient 修饰，该关键字声明数组默认不会被序列化。

```java
transient Object[] elementData; // non-private to simplify nested class access
```

ArrayList 实现了 writeObject() 和 readObject() 来控制实现序列化，只序列化数组中有元素填充那部分内容。

序列化时需要使用 ObjectOutputStream 的 writeObject() 将对象转换为字节流并输出。而 writeObject() 方法在传入的对象存在 writeObject() 的时候会去反射调用该对象的 writeObject() 来实现序列化。反序列化使用的是 ObjectInputStream 的 readObject() 方法，原理类似。











## 参考

https://www.cnblogs.com/zhangyinhua/p/7687377.html