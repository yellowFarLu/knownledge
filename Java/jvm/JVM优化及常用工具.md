# JVM的优化



## GC日志

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9haxqpaijj31ji06g0tq.jpg" alt="image-20191201162837299" style="zoom:100%;" />

- 最前面的数字表示GC的发生时间，这个数字的含义是Java虚拟机启动以来经过的秒数。

- GC日志的开头“[GC”和 “[Full GC”说明了这次垃圾收集的停顿类型，如果有“Full”，说明这次垃圾收集发生了“Stop the world”。
  - 一般发生分配担保失败之类的问题，才会发生Full GC，才会发生Stop the world
  - 如果是System.gc()触发的垃圾收集，这里会展示"[Full GC(System)"
- "[DefNew"表示GC发生的区域，这里显示的区域名称与使用的垃圾收集器相关
  - 如果使用Serial收集器，则显示为"[DefNew"
  - 如果使用ParNew收集器，则显示为“[ParNew”
  - 如果使用Parallel Scavenge收集器，则显示为“[PSYoungGen”
  - 老年代和永久代同理，也是由收集器决定的
- 方括号内的"3324k->152k(3712k)"的含义是“GC前该内存区域已使用空间 -> GC后该内存区域已使用空间(该内存区域总容量)”
- “0.0025925 secs”表示该内存区域GC所占用时间，单位是秒
- 方括号之外的"3324k->152k(11904k)" 表示 “GC前Java堆已使用容量 -> GC后Java堆已使用容量(Java堆总容量)”





## 垃圾收集器参数

![image-20191201170221603](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hbwumrgbj31cb0u0b29.jpg)

![image-20191201170238863](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hbx56fa9j31gq082qcw.jpg)

![image-20191201170255821](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hbxg2b7tj31ho0iu7ox.jpg)

```java
// 放一些参数在这里，方便复制使用

-XX:+UseConcMarkSweepGC  // 使用CMS收集器
-XX:+UseG1GC     // 使用G1收集器  
```



  





## 情况

1、一般来说，当survivor区不够大或者占用量达到50%，就会把一些对象放到老年区。通过设置合理的eden区，survivor区及使用率，**可以将年轻对象保存在年轻代**，从而避免full GC，使用-Xmn设置年轻代的大小



2、**对于占用内存比较多的对象，一般会选择在老年代分配内存**。如果在年轻代给大对象分配内存，年轻代内存不够了，就要在eden区移动大量对象到老年代，然后这些移动的对象可能很快消亡，因此导致full GC。通过设置参数：-XX:PretenureSizeThreshold=1000000，单位为B，标明对象大小超过1M时，在老年代(tenured)分配内存空间。



3、一般情况下，年轻对象放在eden区，当第一次GC后，如果对象还存活，放到survivor区，此后，每GC一次，年龄增加1，**当对象的年龄达到阈值，就被放到tenured老年区**。这个阈值可以同构-XX:MaxTenuringThreshold设置。如果想让对象留在年轻代，可以设置比较大的阈值。



4、设置最小堆和最大堆： -Xmx（最大）  和  -Xms（最小）  稳定的堆大小堆垃圾回收是有利的，获得一个稳定的堆大小的方法是设置-Xms和-Xmx的值一样，即最大堆和最小堆一样，如果这样子设置，系统在运行时堆大小理论上是恒定的，**稳定的堆空间可以减少GC次数（我的理解是不会动态减少堆内存，则内存足够分配的概率比较高，从而减少GC）**，因此，很多服务端都会将这两个参数设置为一样的数值。稳定的堆大小虽然减少GC次数，**但是增加每次GC的时间（假如堆只使用了1G，但是堆大小固定为2G，因此每次都要扫描整个堆的内存，无论是否使用，因此增加时间）**，因为每次GC要把堆的大小维持在一个区间内。



5、一个不稳定的堆并非毫无用处。在系统不需要使用大内存的时候，压缩堆空间，使得GC每次应对一个较小的堆空间，加快单次GC次数。基于这种考虑，JVM提供两个参数，用于压缩和扩展堆空间。

（1）-XX:MinHeapFreeRatio 参数用于设置堆空间的最小空闲比率。默认值是40，当堆空间的空闲内存比率小于40，JVM便会扩展堆空间

（2）-XX:MaxHeapFreeRatio 参数用于设置堆空间的最大空闲比率。默认值是70， 当堆空间的空闲内存比率大于70，JVM便会压缩堆空间。

（3）当-Xmx和-Xmx相等时，上面两个参数无效



6、通过增大吞吐量提高系统性能，可以通过设置并行垃圾回收收集器。

（1）-XX:+UseParallelGC:年轻代使用并行垃圾回收收集器。这是一个关注吞吐量的收集器，可以尽可能的减少垃圾回收时间。

（2）-XX:+UseParallelOldGC:设置老年代使用并行垃圾回收收集器。



7、尝试使用大的内存分页：使用大的内存分页增加CPU的内存寻址能力，从而系统的性能。-XX:+LargePageSizeInBytes参数设置内存页的大小



8、使用非占用的垃圾收集器。-XX:+UseConcMarkSweepGC老年代使用CMS收集器降低停顿。

9、-XXSurvivorRatio=3，表示survivor:eden = 2:3







## JVM性能调优的工具

- jps
  - JVM Process Status，显示所有Java虚拟机进程
- jstat
  - 用于收集虚拟机各方面的运行数据
- jinfo
  - 显示虚拟机配置信息
- jmap
  - 生成虚拟机内存转储快照，该快照文件称为heapdump文件
  - 也就是我们常说的把内存运行情况dump下来
- jhat
  - 用于分析heapdump文件，它会建议一个HTTP服务器，让用户可以在浏览器上查看分析结果
- jstack
  - 显示虚拟机线程快照，从而查看虚拟机内线程运行情况



### jps

jps可以列出正在运行的虚拟机进程。可以展示的字段有：虚拟机进程ID（本地虚拟机唯一ID，即LVMID）、展示执行主类名称（执行main()方法的类名）。

对于本地虚拟机进程来说，LVMID和操作系统进程ID是一致的。

![image-20191201191821286](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hfue7b94j318q0d2wmv.jpg)





### jstat

jstat用于监控虚拟机的运行信息。

它可以显示虚拟机中类加载、内存、垃圾收集、JIT编译等运行数据。

jstat的命令格式为：

```java
 jstat [-命令选项] [vmid] [间隔时间/毫秒] [查询次数]
```

查询间隔和查询次数如果不写，则只查询1次。

示例：

查询虚拟机ID为5426的进程的GC情况，250毫秒查询1次，总共查询2次

```java
jstat -gc 5426 250 2
```



**命令选项**代表着用户希望查询的虚拟机信息，主要分为3类：类加载、垃圾收集、运行时编译情况

![image-20191201192649869](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hg36a6ovj31gm0u0kjb.jpg)



jstat查询的结果各个字段参考：[jstat查看GC情况](https://www.cnblogs.com/yjd_hycf_space/p/7755633.html)，也可以参看下面的图片：

![image-20191201193108557](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hg7nwg8nj31km0a2gqx.jpg)



**-gccause的情况下，会输出LGCC，表示上一次发生GC的原因：**

![image-20191208164015429](https://tva1.sinaimg.cn/large/006tNbRwgy1g9pem05r6oj318c03ewf7.jpg)



**查看YGC次数**

![image-20200411134313948](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdpryc2qbcj31om0cyjvb.jpg)

图片里面的箭头指向YGC的次数







### jinfo

jinfo用于实时查看和调整虚拟机各项参数。

使用jps -v可以查看虚拟机启动时显式指定的参数列表，但是如果想知道未被显式指定的参数，可以使用jinfo -flag。

jinfo命令格式

```java
jinfo [参数名] pid
```



### jmap

jmap命令用于生成转储快照，一般称为heapdump或者dump文件。

除此之外，jmap还可以查询finalize执行队列，Java堆和永久代的详细信息（如空间使用率、当前用的是哪种收集器）。

jmap的命令格式

```java
jmap [option] pid
```

![image-20191201214726412](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hk5h81d4j31h60sunjx.jpg)

例子：

```java
jmap -dump:format=b,file=my.bin 5426
```





### jhat

jhat是虚拟机dump文件的分析工具。jhat内置了一个微型的HTTP服务器，生成dump的分析结果后，可以在浏览器查看。

在实际工作中，一般不会直接使用jhat来分析dump文件，原因有二：

- 一般不会直接在部署应用程序的服务器上解析dump文件，因为会耗费资源。因此会把dump文件拷贝到其他机器上进行分析。既然都要到其他机器上面进行分析了，就没有必要受到命令行工具的限制了。
- jhat的分析功能相对来说比较简陋



示例

```java
jhat my.bin
// my.bin就是dump文件的名称
```

执行完命令之后，在浏览器输入 http://localhost:7000/ 就可以看到分析结果了。



### jstack

jstack用于生成当前虚拟机的线程快照。

线程快照就是当前虚拟机每一条线程正在执行的方法的堆栈集合。

生成线程快照的主要目的是 **定位线程出现长时间停顿的原因**。如线程死锁、死循环、请求外部资源导致的长时间等待都是导致线程长时间停顿的常见原因。

线程出现停顿的时候通过jstack就可以分析出线程在做什么，从而知道为什么停顿。

jstack命令格式

```java
jstack [option] pid
```

![image-20191201220308500](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hkltaeuoj31bs0e0qaq.jpg)



示例

```java
jstack -l 5426
```







## JDK可视化工具



### JConsole

JConsole是一款基于JMX的可视化监视、管理工具。



**1、启动JConsole**

通过命令行输入

```java
JConsole
```

即可以启动JConsole。

![image-20191201221216962](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hkvbi2qxj31320u0wir.jpg)



**2、内存监控**

“内存”标签页相当于可视化的jstat命令。



**3、线程监控**

“线程”标签页相当于可视化的jstack命令。遇到线程停顿时可以使用这个标签页功能进行分析。





### VisualVM

VisualVM是目前最强大的监控和故障处理工具。

![image-20191201224949306](https://tva1.sinaimg.cn/large/006tNbRwgy1g9hlydq9a6j31jk0qg11g.jpg)



**1、启动VisualVM**

在命令行输入

```java
jvisualvm
```

即可启动visualVM











## 参考

《深入理解Java虚拟机》

http://www.51testing.com/html/95/115295-804841.html

https://blog.csdn.net/u010663871/article/details/73603460

https://blog.csdn.net/softwave/article/details/6238747



