## IdentityHashMap

IdentityHashMap利用Hash表来实现Map接口，比较键（和值）时使用引用相等性代替对象相等性。总结如下：

- 比如对于要保存的key，k1和k2，当且仅当k1== k2的时候，IdentityHashMap才会相等，而对于HashMap来说，相等的条件则是：k1.equals(k2)。

- 同HashMap，IdentityHashMap也是无序的，并且该类不是线程安全的，如果要使之线程安全，可以调用Collections.synchronizedMap(new IdentityHashMap(...))方法来实现。



###结构

![616953-20160308093443804-1958725444](https://ws3.sinaimg.cn/large/006tNc79gy1g2972u9zkoj30i10emq3a.jpg)

说明：IdentityHashMap的数据很简单，底层实际就是一个Object数组（key1、value1、key2、value2、key3、value3的顺序存放），在逻辑上需要看成是一个环形的数组，解决冲突的办法是：根据计算得到散列位置，如果发现该位置上已经有元素，则往后查找（index增加2），直到找到空位置，进行存放，如果没有，直接进行存放。当元素个数达到一定阈值时，Object数组会自动进行扩容处理。



### 源码解析

####类的继承关系

```java
public class IdentityHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, java.io.Serializable, Cloneable
```

说明：继承了AbstractMap抽象类，实现了Map接口，可序列化接口，可克隆接口。



####类的属性

```java
public class IdentityHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, java.io.Serializable, Cloneable
{
    // 缺省容量大小
    private static final int DEFAULT_CAPACITY = 32;
    // 最小容量
    private static final int MINIMUM_CAPACITY = 4;
    // 最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 29;
    // 用于存储实际元素的表
    transient Object[] table;
    // 大小
    int size;
    // 对Map进行结构性修改的次数
    transient int modCount;
    // null key所对应的值
    static final Object NULL_KEY = new Object();
}
```

说明：可以看到类的底层就是使用了一个Object数组来存放元素。



####capacity函数

```java
// 此函数返回的值是最小大于expectedMaxSize的2次幂(真没看懂。。。)
private static int capacity(int expectedMaxSize) {
    return
      (expectedMaxSize > MAXIMUM_CAPACITY / 3) ? MAXIMUM_CAPACITY :
    (expectedMaxSize <= 2 * MINIMUM_CAPACITY / 3) ? MINIMUM_CAPACITY :
    Integer.highestOneBit(expectedMaxSize + (expectedMaxSize << 1));
}
```



#### hash函数

```java
// hash函数，由于length总是为2的n次幂，所以 & (length - 1)相当于对length取模
private static int hash(Object x, int length) {
    int h = System.identityHashCode(x);
    // Multiply by -127, and left-shift to use least bit as part of hash
    return ((h << 1) - (h << 8)) & (length - 1);
}
```

说明：hash函数用于散列，并且保证key元素的散列值会在数组偶次索引。



#### get函数

```java
public V get(Object key) {
    // 保证null的key会转化为Object()
    Object k = maskNull(key);
    // 保存table
    Object[] tab = table;
    int len = tab.length;
    // 得到key的散列位置
    int i = hash(k, len);
    // 遍历table，解决散列冲突的办法是若冲突，则往后寻找空闲区域
    while (true) {
      Object item = tab[i];
      // 判断是否相等(地址是否相等)
      if (item == k)
        // 地址相等，即完全相等的两个对象
        return (V) tab[i + 1];
      // 对应散列位置的元素为空，则返回空
      if (item == null)
        return null;
      // 取下一个Key索引
      i = nextKeyIndex(i, len);
    }
}
```

说明：该函数比较key值是否完全相同（对象类型则是否为同一个引用）



#### nextKeyIndex函数

```java
// 下一个Key索引
private static int nextKeyIndex(int i, int len) {
    // 往后移两个单位
    return (i + 2 < len ? i + 2 : 0);
}
```

说明：此函数用于发生冲突时，取下一个位置进行判断。



#### put函数

```java
public V put(K key, V value) {
    // 保证null的key会转化为Object(NULL_KEY)
    final Object k = maskNull(key);

    retryAfterResize: for (;;) {
      final Object[] tab = table;
      final int len = tab.length;
      int i = hash(k, len);

      for (Object item; (item = tab[i]) != null;
           i = nextKeyIndex(i, len)) {
        if (item == k) { // 经过hash计算的项与key相等
          // 取得值
          V oldValue = (V) tab[i + 1];
          // 将value存入
          tab[i + 1] = value;
          // 返回旧值
          return oldValue;
        }
      }

      // 大小加1
      final int s = size + 1;
      // Use optimized form of 3 * s.
      // Next capacity is len, 2 * current capacity.
      // 如果3 * size大于length，则会进行扩容操作
      if (s + (s << 1) > len && resize(len))
        // 扩容后重新计算元素的值，寻找合适的位置进行存放
        continue retryAfterResize;
      // 结构性修改加1
      modCount++;
      // 存放key与value
      tab[i] = k;
      tab[i + 1] = value;
      // 更新size
      size = s;
      return null;
    }
}
```

说明：若传入的key在表中已经存在了（强调：是同一个引用），则会用新值代替旧值并返回旧值；如果元素个数达到阈值，则扩容，然后再寻找合适的位置存放key和value。



#### resize函数

``` java
private boolean resize(int newCapacity) {
    // assert (newCapacity & -newCapacity) == newCapacity; // power of 2
    int newLength = newCapacity * 2;
    // 保存原来的table
    Object[] oldTable = table;
    int oldLength = oldTable.length;
    // 旧表是否为最大容量的2倍
    if (oldLength == 2 * MAXIMUM_CAPACITY) { // can't expand any further
      // 之前元素个数为最大容量，抛出异常
      if (size == MAXIMUM_CAPACITY - 1)
        throw new IllegalStateException("Capacity exhausted.");
      return false;
    }
    // 旧表长度大于新表长度，返回false
    if (oldLength >= newLength)
      return false;
    // 生成新表
    Object[] newTable = new Object[newLength];
    // 将旧表中的所有元素重新hash到新表中
    for (int j = 0; j < oldLength; j += 2) {
      Object key = oldTable[j];
      if (key != null) {
        Object value = oldTable[j+1];
        oldTable[j] = null;
        oldTable[j+1] = null;
        int i = hash(key, newLength);
        while (newTable[i] != null)
          i = nextKeyIndex(i, newLength);
        newTable[i] = key;
        newTable[i + 1] = value;
      }
    }
    // 新表赋值给table
    table = newTable;
    return true;
}
```

说明：当表中元素达到阈值时，会进行扩容处理，扩容后会旧表中的元素重新hash到新表中。



#### remove函数

```java
public V remove(Object key) {
    // 保证null的key会转化为Object(NULL_KEY)
    Object k = maskNull(key);
    Object[] tab = table;
    int len = tab.length;
    // 计算hash值
    int i = hash(k, len);

    while (true) {
      Object item = tab[i];
      // 找到key相等的项
      if (item == k) {
        modCount++;
        size--;
  
        V oldValue = (V) tab[i + 1];
        tab[i + 1] = null;
        tab[i] = null;
  
        // 删除后需要进行后续处理，把之前由于冲突往后挪的元素移到前面来
        closeDeletion(i);
        return oldValue;
      }
      
      // 该项为空
      if (item == null)
        return null;
      
      // 查找下一个位置
      i = nextKeyIndex(i, len);
    }
}
```



####closeDeletion函数

```java
private void closeDeletion(int d) {
     
    Object[] tab = table;
    int len = tab.length;

    Object item;

    // 把该元素后面符合移动规定的元素往前面移动
    // d是被删除元素的位置，i是被删除元素的下一个位置   r是i位置上元素本来应该在的位置。
    for (int i = nextKeyIndex(d, len); (item = tab[i]) != null; i = nextKeyIndex(i, len) ) {
      
      int r = hash(item, len);
      
      if ((i < r && (r <= d || d <= i)) || (r <= d && d <= i)) {
          tab[d] = item;
          tab[d + 1] = tab[i + 1];
          tab[i] = null;
          tab[i + 1] = null;
          d = i;
      }
      
    }
}
```

说明：在删除一个元素后会进行一次closeDeletion处理，重新分配元素的位置。

下图表示在closeDeletion前和closeDeletion后的示意图

![Snip20190420_2](https://ws3.sinaimg.cn/large/006tNc79gy1g298f0dpmtj30fe10qjtu.jpg)

假设：其中，（"aa" -> "aa"）经过hash后在第0项，("bb" -> "bb")经过hash后也应该在0项，发生冲突，往后移到第2项，("cc" -> "cc")经过hash后在第2项，发生冲突，往后面移动到第4项，("gg" -> "gg")经过hash在第2项，发生冲突，往后移动到第6项，("dd" -> "dd")在第8项，("ee" -> "ee")在第12项。

当删除("bb" -> "bb")后，进行处理后的元素布局如右图所示。

![Snip20190420_2](https://ws1.sinaimg.cn/large/006tNc79gy1g298ekgyw8j30da0rudgw.jpg)