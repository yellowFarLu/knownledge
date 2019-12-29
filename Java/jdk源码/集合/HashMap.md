# HashMap



## 概述

我们在一般情况下，都会使用无参构造方法创建 HashMap。但当我们对时间和空间复杂度有要求的时候，使用默认值有时可能达不到我们的要求，这个时候我们就需要手动调参。在 HashMap 构造方法中，可供我们调整的参数有两个，一个是初始容量 initialCapacity，另一个负载因子 loadFactor。通过设定这两个参数，可以进一步影响阈值大小。但初始阈值 threshold 仅由 initialCapacity 经过移位操作计算得出。他们的作用分别如下：

| initialCapacity | HashMap 初始容量                                             |
| --------------- | ------------------------------------------------------------ |
| loadFactor      | 负载因子                                                     |
| threshold       | 当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容 |

负载因子（loadFactor）。对于 HashMap 来说，负载因子是一个很重要的参数，该参数反应了 HashMap 桶数组的使用情况（假设键值对节点均匀分布在桶数组中）。通过调节负载因子，可使 HashMap 时间和空间复杂度上有不同的表现。

- 当我们调低负载因子时，HashMap 所能容纳的键值对数量变少。扩容时，重新将键值对存储新的桶数组里，键的键之间产生的碰撞会下降，链表长度变短。此时，HashMap 的增删改查等操作的效率将会变高，这里是典型的拿空间换时间。
- 相反，如果增加负载因子（负载因子可以大于1），HashMap 所能容纳的键值对数量变多，空间利用率高，但碰撞率也高。这意味着链表长度变长，效率也随之降低，这种情况是拿时间换空间。至于负载因子怎么调节，这个看使用场景了。一般情况下，我们用默认值就可以了。





## 与 Hashtable 的比较

- Hashtable 使用 synchronized 来进行同步。
- HashMap 可以插入键为 null 的 Entry。
- HashMap 的迭代器是 fail-fast 迭代器。
- HashMap 不能保证随着时间的推移 Map 中的元素次序是不变的。





## 源码



### put插入元素

- 找到桶的位置  
  - 使用(n - 1) & hash，这也说明了为什么数组长度是2的整数次幂
- 插入到链表尾部
- 如果是红黑树，则插入到红黑树中
- 插入完毕后，判断需不需要进行扩容



### 扩容

在 HashMap 中，桶数组的长度均是2的次幂，阈值大小为桶数组长度与负载因子的乘积。当 HashMap 中的键值对数量超过阈值时，进行扩容。

若size大于threshold的时候，就会进行扩容。HashMap 按当前桶数组长度的2倍进行扩容，阈值也变为原来的2倍（如果计算过程中，阈值溢出归零，则按阈值公式重新计算）。扩容之后，要重新计算键值对的位置，并把它们移动到合适的位置上去。

![image-20191015102021139](https://tva1.sinaimg.cn/large/006y8mN6gy1g7yo64l5n5j30kj0ce439.jpg)

rehash是非常耗时的。在编写程序中，尽量确定HashMap的初始容量大小，从而避免rehash。









## hashMap1.8相对于1.7做了哪些改进



### 1、插入元素的方式

插入元素的方式不同。

jdk1.7采用的是头插法，因为头插法只要在链表头部插入元素，时间复杂度为O(1)，尾插法要遍历到链表最后一个元素，才能插入，时间复杂度为O(n)。

jdk1.8采用的是尾插法。jdk1.8之所以改成尾插法，是因为使用头插法在高并发的情况下，由于逆序容易形成环形链，导致死循环。

通过采用尾插法，避免逆序，从而避免出现环形链，从而避免死循环。



#### jdk1.8的插入参考

![944365-c263d87e5eecdcb0](https://tva1.sinaimg.cn/large/006tNbRwgy1gaajtt2iqpj30tq1cmdkv.jpg)





#### jdk1.7参考

![944365-a5570a7fe29fb8b9](https://tva1.sinaimg.cn/large/006tNbRwgy1gaajsgalbnj30u00y70wz.jpg)





### 2、数据结构

- jdk1.7使用数组 + 链表的结构
- jdk1.8使用数组+链表+红黑树的结构。



### 3、扩容时数据存储位置的计算方式也不同

- jdk1.7的话，重新计算hash值，hash&(n - 1)的方式计算出元素的新下标
- jdk1.8的话，计算数据存储位置的方式进行了优化。扩容时，容量变为2倍了，那么对于(n-1)这个值来说，二进制多了一个1，那么假如进行 hash & (n -1)的运算
  - 那么如果参与运算的hash值对应的二进制位是0的话，进行&运算的结果是0，那么结果是就是原位置
  - 如果参与运算的hash值对应的二进制是1的话，进行&运算的结果是1，那么结果就是原位置 + 旧容量
  - 通过利用这个规律，巧妙的优化了数据存储位置的计算，只需要判断hash中新参与运算的位是0还是1即可，提高了扩容效率

![944365-2466f5db47fd7685](https://tva1.sinaimg.cn/large/006tNbRwgy1gadjbjlcfoj30oq0fumyr.jpg)

  ```java
/*
 * (e.hash & oldCap) == 0 这步很巧妙
 * 如果oldCap是16，则 00010000
 * 从而判断 hash值中新增参与运算的位是0还是1
*/
if ((e.hash & oldCap) == 0) {
  if (loTail == null)
    loHead = e;
  else
    loTail.next = e;
  loTail = e;
}
else {
  if (hiTail == null)
    hiHead = e;
  else
    hiTail.next = e;
  hiTail = e;
}
  ```







### 4、扩容与插入顺序不同

jdk1.7是先扩容再插入，jdk1.8是先插入再扩容。











## 问题



**hashmap扩容时每个entry需要再计算一次hash吗？**

首先明确1.7和1.8都是扩容2倍。

- 1.8之前（如1.7）会将每个entry进行rehash，也就是说需要重新计算hash，算法设计如此。

- 1.8之后不用，将算法进行了优化，**因为元素的位置要么是在原位置，要么是在原位置+旧数组容量的位置**，

所以只要把原来桶中的链表分为新旧两个表就好了，不用重新计算hash值。





**jdk1.8之前并发操作hashmap时为什么会有死循环的问题？**

jdk1.7中，链表插入新节点**采用的是头插法**，这样在多线程扩容迁移元素时，会将元素顺序改变，导致两个线程中出现元素的相互指向而形成环型的链表，然后调用get()方法的时候，就可能会产生死循环。

**jdk1.8 采用了尾插法**，从根源上杜绝了这种情况的发生





**链表转红黑树的个数？**

从 JDK 1.8 开始，一个桶存储的链表长度大于等于 8 时会将链表转换为红黑树。





**new一个hashMap，初始容量为1000，然后插入1000个元素，会不会触发扩容？如果是10000呢？**

首先初始容量为1000，那么数组的大小就是1024，那么此时 阈值=1024*0.75f = 768，这时候，当插入769个元素的时候，判断元素个数达到阈值，则触发扩容。

为什么是1024呢？因为有tableSizeFor方法就是计算出大于元素数组个数的最小2次幂。之所以要这样子做，是因为要保持数组的元素个数为2的次幂，从而能够使用 hash&(n-1)这种取模计算方式，通过利用位运算，来提高效率。

10000个元素的时候，同理。

```java
public HashMap(int initialCapacity, float loadFactor) {
        // 默认负载因此为0.75f
        this.loadFactor = loadFactor;
        // 默认阈值为 传入数组初始容量的 最小二次幂
        this.threshold = tableSizeFor(initialCapacity);
}
```

```java
final Node<K,V>[] resize() {

        Node<K,V>[] oldTab = table;
        // 获取旧的桶的个数（旧容量）
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 获取旧的阈值
        int oldThr = threshold;
        // 新的桶个数、新的阈值
        int newCap, newThr = 0;

        /*
         * 这一块是根据旧容量、旧阈值来计算新容量、新阈值
         */
        if (oldCap > 0) {
            // 有数组容量，说明之前已经初始化过了

            if (oldCap >= MAXIMUM_CAPACITY) {
                // 大于最大容量，无法扩容，那么就只能让阈值变为Integer.MAX_VALUE，从而避免下次扩容。
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 容量扩容2倍，并且扩容后小于最大容量，证明下次还可能扩容。但是不想它频繁的进行扩容，所以让阈值也扩容2倍
                newThr = oldThr << 1;
        }
        else if (oldThr > 0)
            /*
             * 旧的容量为0，但是旧阈值大于0的情况下，初始容量设置为旧阈值
             * 假如通过 new HashMap(1000) 这种情况下，一开始threshold计算为1024，然后通过这句代码，传递为新容量的大小
             */
            newCap = oldThr;
        else {
            // 旧容量、阈值都为0，则使用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            // 计算新的阈值，值=新的数组容量 * 负载因子
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;

		......
}
```





**为什么hashmap不用AVL树，用红黑树？**

- AVL是一种平衡二叉树，每个节点都存储着平衡因子，如果当前节点的左右子树的高度差超过1，则进行旋转调整，使得平衡因子小于等于1，从而让树的高度不会太高
- 红黑树的节点不是红色就是黑色，从根节点到子节点的所有路径上的黑色节点个数相同
- 红黑树的插入效率更快
  - 因为红黑树只需要更少的旋转次数就可以达到平衡
  - 而AVL数需要保证左右子树的高度差不超过1，因此插入的时候需要更多的旋转操作来保证平衡
- AVL树的查找更快
  - 因为AVL树需要保证左右子树的高度差不超过1，因此AVL更加要求平衡，因此树的高度一般来说会比红黑树低，因此查找效率就更快一些。
- 我理解：作者想提高HashMap扩容的效率，因为扩容的时候，相当于阻塞当前对hashMap的操作，因此需要更快的扩容完成，就选择使用红黑树。并且红黑树也能兼顾查找效率。





**为什么Map桶中个数超过8才转为红黑树?**

我们首先要明白为什么要转换，这个问题比较简单，因为Map中桶的元素初始化是链表保存的，其查找性能是O(n)，而树结构能将查找性能提升到O(log(n))。当链表长度很小的时候，即使遍历，速度也非常快，但是当链表长度不断变长，肯定会对查询性能有一定的影响，所以才需要转成树。

**那么为什么不是一开始就转换为红黑树？**因为链表元素个数非常少的时候，直接遍历查找和红黑树查找是差不多的，但是红黑树还要花费内存、CPU去维护这个数据结构，也就是说查找效率提升不大的情况下，会浪费内存、CPU资源（比如说指针、旋转操作）。

**那么为什么是8呢？** 根据源码的注解，链表中元素个数达到8的概率是非常低的，因此就能尽可能避免转成红黑树（避免查询效率不能提升的情况下，去维护红黑树）。





**为什么默认负载因子是0.75?**

- 空间和性能的折中，太大则hash冲突多，查询效率降低；太小则浪费
- 0.75也就是3/4，那么乘以16以后的值也是2的次幂，jdk1.8的代码中把数组初始大小设置为初始化阈值，因此可以保证数组的大小是2的次幂。





## 参考

https://www.cnblogs.com/zhangyinhua/p/7698642.html#_label1

https://cloud.tencent.com/developer/article/1338265

https://juejin.im/post/5d412a5be51d45620e0b99af

[HashMap源码分析]([http://www.tianxiaobo.com/2018/01/18/HashMap-%E6%BA%90%E7%A0%81%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90-JDK1-8/](http://www.tianxiaobo.com/2018/01/18/HashMap-源码详细分析-JDK1-8/))

[扩容](https://www.jianshu.com/p/1ff9f3dee207)

[HashMap死循环](https://www.jianshu.com/p/1e9cf0ac07f4)

[hashMap1.8相对于1.7做了哪些改进](https://blog.csdn.net/qq_36520235/article/details/82417949)

[hashMap1.8及1.7源码对比](https://www.jianshu.com/p/8324a34577a0?utm_source=oschina-app)

[hashMap1.7源码分析](https://www.jianshu.com/p/e5c8a814c0ca)

[为什么Java8中HashMap链表使用红黑树而不是AVL树](https://blog.csdn.net/21aspnet/article/details/88939297)