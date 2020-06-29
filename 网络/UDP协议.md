# UDP



## 概述

UDP是User Datagram Protocol的简称，中文名是用户数据报协议，是OSI参考模型中的传输层协议，它是一种无连接的传输层协议，提供简单不可靠信息传送服务。

 UDP是无连接的，尽最大可能交付，没有拥塞控制，面向报文（对于应用程序传下来的报文不合并也不拆分，只是添加 UDP 首部），支持一对一、一对多、多对一和多对多的交互通信。





## UDP 首部格式

![image-20191130170114926](https://tva1.sinaimg.cn/large/006tNbRwgy1g9g69e51ayj30w00kkwr3.jpg)

首部字段只有 8 个字节，包括源端口、目的端口、长度、检验和。12 字节的伪首部是为了计算检验和临时添加的。







## 参考

[UDP概述](https://blog.csdn.net/china_jeffery/article/details/78923428)

[TCP/IP卷一](http://www.52im.net/topic-tcpipvol1.html)



