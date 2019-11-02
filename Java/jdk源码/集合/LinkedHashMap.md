# LinkedHashMap

LinkedHashMap 继承自 HashMap，在 HashMap 基础上，通过维护一条双向链表，解决了 HashMap 不能保持遍历顺序和插入顺序一致的问题。除此之外，LinkedHashMap 对访问顺序也提供了相关支持。



## 结构

首先看下HashMap的结构：

![image-20191102212032574](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k0ek06ukj30tm0m8n7e.jpg)

由数组 + 链表+ 红黑树实现，那么LinkedHashMap仅仅是在此基础上加入了双向链表而已，如图：

![image-20191102212128712](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k0fix0fwj30z60noqgk.jpg)



## 源码



### 插入元素

基本沿用HashMap的插入过程。只是增加了新元素加入队列的步骤。

- 链表的建立过程是在插入键值对节点时开始的，初始情况下，让 LinkedHashMap 的 head 和 tail 引用同时指向新节点，链表就算建立起来了。
- 随后不断有新节点插入，通过将新节点接在 tail 引用指向节点的后面。

双向链表建立之后，我们就可以按照插入顺序去遍历 LinkedHashMap。



### 删除元素

与插入操作一样，LinkedHashMap 删除操作相关的代码也是直接用父类的实现。只是增加了删除链表中节点的步骤。

- 找到数组中元素的桶
- 从单链表中删除该元素
- 从双链表中删除该元素



举个例子说明一下，假如我们要删除下图键值为 3 的节点。

![image-20191102212800905](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k0mbixd9j30w80lkwsn.jpg)



根据 hash 定位到该节点属于3号桶，然后在对3号桶保存的单链表进行遍历。找到要删除的节点后，先从单链表中移除该节点。如下

![image-20191102212815657](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k0mkpqldj30wm0b2gsd.jpg)

然后再双向链表中移除该节点：

![image-20191102212830574](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k0mu7o5bj30ve0lutkw.jpg)



### 访问顺序的维护

LinkedHashMap默认按照插入顺序来遍历元素，但是也可以设置为按照访问顺序遍历元素。

```java
public class LinkedHashMapDemo {

    public static void main(String[] args) {

        LinkedHashMap<Integer, Integer> map =
                new LinkedHashMap<>(20, 0.75f, true);

        for (int i = 0; i < 10; i++) {
            map.put(i, i);
        }

        System.out.println(map);

        map.get(4);

        System.out.println(map);
    }

}

输出
{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9}
{0=0, 1=1, 2=2, 3=3, 5=5, 6=6, 7=7, 8=8, 9=9, 4=4}
```

源码其实很简单，把访问的节点放到双向链表的尾部，如图：

假设我们访问下图键值为3的节点，访问前结构为：

![image-20191102214441480](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k13o5mm6j30y80ngqgc.jpg)

访问后，键值为3的节点将会被移动到双向链表的最后位置，其前驱和后继也会跟着更新。访问后的结构如下：

![image-20191102214519232](https://tva1.sinaimg.cn/large/006y8mN6gy1g8k14c1xahj30y00lyqfx.jpg)



## 实现LRU

使用LinkedHashMap可以实现LRU算法，因为LinkedHashMap可以设置accessOrder=true，让被访问的元素移动到双向链表尾部，那么双向链表头部就是最近最少被访问的。如果发现数量超过限制了，就删除双向链表头部的元素。

下面演示一下如何使用LinkedHashMap实现LRU算法。

```java
public class SimpleCache<K, V> extends LinkedHashMap<K, V> {

    private static final int MAX_NODE_NUM = 100;

    private int limit;

    public SimpleCache() {
        this(MAX_NODE_NUM);
    }

    public SimpleCache(int limit) {
        super(limit, 0.75f, true);
        this.limit = limit;
    }

    public V save(K key, V val) {
        return put(key, val);
    }

    public V getOne(K key) {
        return get(key);
    }

    public boolean exists(K key) {
        return containsKey(key);
    }

    /**
     * 判断节点数是否超限
     * 通过重写removeEldestEntry方法，实现LRU。因为在插入的时候，LinkedHashMap通过这个方法是否需要删除双向链表头部的元素。
     * @param eldest
     * @return 超限返回 true，否则返回 false
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > limit;
    }
}
```



```java
public class LinkedHashMapDemo {

    public static void main(String[] args) {

        LinkedHashMap<Integer, Integer> linkedHashMap = new LinkedHashMap<>();

        for (int i = 10; i >= 0; i--) {
            linkedHashMap.put(i, i);
        }

        System.out.println(linkedHashMap);



        HashMap<Integer, Integer> hashMap = new HashMap<>();

        for (int i = 10; i >= 0; i--) {
            hashMap.put(i, i);
        }

        System.out.println(hashMap);
    }

    private void accessOrderDemo() {

        LinkedHashMap<Integer, Integer> map =
                new LinkedHashMap<>(20, 0.75f, true);

        for (int i = 0; i < 10; i++) {
            map.put(i, i);
        }

        System.out.println(map);

        map.get(4);

        System.out.println(map);
    }

}
```



## 参考

[LinkedHashMap源码解析](https://segmentfault.com/a/1190000012964859#articleHeader0)

