# LinkedHashMap

LinkedHashMap 继承自 HashMap，在 HashMap 基础上，通过维护一条双向链表，解决了 HashMap 不能保持遍历顺序和插入顺序一致的问题。除此之外，LinkedHashMap 对访问顺序也提供了相关支持。

（支持按照插入顺序、访问顺序来遍历元素）



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







### 类的属性

```java
public class LinkedHashMap<K,V>  extends HashMap<K,V> implements Map<K,V> {
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    // 版本序列号
    private static final long serialVersionUID = 3801124242820219131L;

    // 链表头结点
    transient LinkedHashMap.Entry<K,V> head;

    // 链表尾结点
    transient LinkedHashMap.Entry<K,V> tail;

    // 访问顺序
    // accessOrder默认为false，表示迭代顺序按照插入顺序访问。
    // accessOrder为true 表示迭代顺序按照元素的访问顺序进行，即不按照之前的插入顺序了
    final boolean accessOrder;
}
```





### newNode函数

```java
// 当桶中结点类型为HashMap.Node类型时，调用此函数
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // 生成Node结点
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    // 将该结点插入双链表末尾
    linkNodeLast(p);
    return p;
}
```

说明：此函数在HashMap类中也有实现，LinkedHashMap重写了该函数，所以当实际对象为LinkedHashMap，桶中结点类型为Node时，我们调用的是LinkedHashMap的newNode函数，而非HashMap的函数，newNode函数会在调用put函数时被调用。可以看到，除了新建一个结点之外，还把这个结点链接到双链表的末尾了，这个操作维护了插入顺序。

其中LinkedHashMap.Entry继承自HashMap.Node

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    // 前后指针
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

说明：在HashMap.Node基础上增加了前后两个指针域，注意，HashMap.Node中的next域也存在。



### newTreeNode函数

```java
// 当桶中结点类型为HashMap.TreeNode时，调用此函数
TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    // 生成TreeNode结点
    TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
    // 将该结点插入双链表末尾
    linkNodeLast(p);
    return p;
}
```

说明：当桶中结点类型为TreeNode时候，插入结点时调用的此函数，也会链接到末尾。



### afterNodeAccess函数

当一个节点被访问时，如果 accessOrder 为 true，则会将该节点移到链表尾部。也就是说指定为 LRU 顺序之后，在每次访问一个节点时，会将这个节点移到链表尾部，保证链表尾部是最近访问的节点，那么链表首部就是最近最久未使用的节点。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 若访问顺序为true，且访问的对象不是尾结点
    if (accessOrder && (last = tail) != e) {
        // 向下转型，记录p的前后结点
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // p的后结点为空
        p.after = null;
        // 如果p的前结点为空
        if (b == null)
            // a为头结点
            head = a;
        else // p的前结点不为空
            // b的后结点为a
            b.after = a;
        // p的后结点不为空
        if (a != null)
            // a的前结点为b
            a.before = b;
        else // p的后结点为空
            // 后结点为最后一个结点
            last = b;
        // 若最后一个结点为空
        if (last == null)
            // 头结点为p
            head = p;
        else { // p链入最后一个结点后面
            p.before = last;
            last.after = p;
        }
        // 尾结点为p
        tail = p;
        // 增加结构性修改数量
        ++modCount;
    }
}
```

说明：此函数在很多函数（如put）中都会被回调，LinkedHashMap重写了HashMap中的此函数。若访问顺序accessOrder为true，且访问的对象不是尾结点，则下面的图展示了访问前和访问后的状态，假设访问的结点为结点3

![616953-20160307085938194-117935380](https://ws1.sinaimg.cn/large/006tNc79gy1g22blttwikj30lw0bh748.jpg)

说明：从图中可以看到，结点3链接到了尾结点后面。



### transferLinks函数

```java
// 用dst替换src
private void transferLinks(LinkedHashMap.Entry<K,V> src,
                               LinkedHashMap.Entry<K,V> dst) {
    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    if (b == null)
        head = dst;
    else
        b.after = dst;
    if (a == null)
        tail = dst;
    else
        a.before = dst;
}
```

说明：此函数用dst结点替换src结点，示意图如下

![616953-20160307091450882-1342476579](https://ws4.sinaimg.cn/large/006tNc79gy1g22btw615nj30jo0bt74b.jpg)

说明：其中只考虑了before与after域，并没有考虑next域，next会在调用tranferLinks函数中进行设定。



### containsValue函数

```java
public boolean containsValue(Object value) {
    // 使用双链表结构进行遍历查找
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true;
    }
    return false;
}
```

说明：containsValue函数根据双链表结构来查找是否包含value，是按照插入顺序进行查找的，与HashMap中的此函数查找方式不同，HashMap是使用按照桶遍历，没有考虑插入顺序。



### afterNodeInsertion()

在 put 等操作之后执行，当 removeEldestEntry() 方法返回 true 时会移除最晚的节点，也就是链表首部节点 first。

evict 只有在构建 Map 的时候才为 false，在这里为 true。

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

emoveEldestEntry() 默认为 false，如果需要让它为 true，需要继承 LinkedHashMap 并且覆盖这个方法的实现，这在实现 LRU 的缓存中特别有用，通过移除最近最久未使用的节点，从而保证缓存空间足够，并且缓存的数据都是热点数据。









## 实现LRU

使用LinkedHashMap可以实现LRU算法，因为LinkedHashMap可以**设置按照访问顺序来遍历元素，让被访问的元素移动到双向链表尾部，那么双向链表头部就是最近最少被访问的。如果发现数量超过限制了，就删除双向链表头部的元素。**

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





### 实现LRU的其他方式

使用链表LinkedList来实现LRU，规定头部是最近最少访问的元素。

- 当访问一个元素的时候，判断该元素在不在链表中
- 如果不在，并且链表个数没有超过限制，则加入链表尾部
- 如果链表元素个数超过了限制，则删掉链表头部元素，元素加到链表尾部
- 如果元素在链表中，则删除该元素，并且加入链表尾部。

```java
import java.util.LinkedList;
import java.util.List;

/**
 * 通过链表实现LRU算法
 * @author huangy on 2020-04-11
 */
public class LruByList {

    /**
     * 缓存长度限制
     */
    private static final Integer MAX_COUNT = 3;

    /*
     * LRU缓存
     */
    private static final List<Node> cache = new LinkedList<>();

    public Integer getValueFromCache(String key) {
        Node tem = getNodeByKey(key);

        if (tem == null) {

            tem = new Node();
            tem.key = key;
            tem.value = getValueFromDB(key);

            if (cache.size() < MAX_COUNT) {
                cache.add(tem);

            } else  {
                cache.remove(0);
                cache.add(tem);
            }

        } else {
            cache.remove(tem);
            cache.add(tem);
        }

        return tem.value;
    }

    // 穿过缓存，从数据库读取数据
    private Integer getValueFromDB(String key) {
        return Integer.parseInt(key);
    }

    private Node getNodeByKey(String key) {
        for (Node node : cache) {
            if (node.key.equals(key)) {
                return node;
            }
        }
        return null;
    }

    /**
     * 查看缓存情况
     */
    public void viewCache() {
        System.out.println(cache);
    }

    public static void main(String[] args) {
        LruByList lruByList = new LruByList();

        for (int i = 0; i < 10; i++) {
            lruByList.getValueFromCache(String.valueOf(i));
        }

        lruByList.viewCache();
    }
}

class Node {

    String key;

    int value;

    @Override
    public boolean equals(Object obj) {

        if (obj instanceof Node) {
            Node otherNode = (Node)obj;
            return key.equals(otherNode.key);
        }

        return false;
    }

    @Override
    public int hashCode() {
        return key.hashCode();
    }

    @Override
    public String toString() {
        return "Node{" +
                "key='" + key + '\'' +
                ", value=" + value +
                '}';
    }
}
```



### 对比

链表实现LRU 和 LinkedHashMap实现LRU，明显是LinkedHashMap的效率更加优化，因为LinkedHashMap在查找元素是否在缓存中的时候，支持时间复杂度O(1)的查找效率，而链表是查找效率是O(n)。







## 参考

[LinkedHashMap源码解析](https://segmentfault.com/a/1190000012964859#articleHeader0)

