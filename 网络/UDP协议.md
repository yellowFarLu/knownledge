# UDP



## 概述

UDP是简单的，面向数据报的运输层协议。进程的每个输出操作都正好生成一个UDP数据，并组装成一个IP数据报。

![image-20200706191345789](https://tva1.sinaimg.cn/large/007S8ZIlgy1gghgssqp64j30bw05kglw.jpg)

UDP不提供可靠性：它将应用程序传给IP层的数据发送出去，但是不保证它们可以到达目的地。









## UDP 首部格式

![image-20200706191624761](https://tva1.sinaimg.cn/large/007S8ZIlgy1gghgvi9dm5j30hn071wf2.jpg)

首部字段只有 8 个字节，包括源端口、目的端口、长度、检验和。12 字节的伪首部是为了计算检验和临时添加的。

端口号表示发送进程和接收进程。由于IP层已经把IP数据报分配给TCP或UDP（根据IP首部中协议字段值），因此TCP端口号由TCP来查看，而UDP端口号由UDP来查看。**TCP端口号与UDP端口号是相互独立的。**
UDP长度字段指的是UDP首部和UDP数据的字节长度。该字段的最小值为 8字节。这个UDP长度是有冗余的。IP数据报长度指的是数据报全长，因此UDP数据报长度是全长减去IP首部的长度。







## 参考

[UDP概述](https://blog.csdn.net/china_jeffery/article/details/78923428)

[TCP/IP卷一](http://www.52im.net/topic-tcpipvol1.html)

[TCP/IP学习笔记](https://blog.csdn.net/lh470342237/article/details/76599052?locationNum=10&fps=1)

