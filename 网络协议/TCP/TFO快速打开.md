# TFO

要发数据先得有三次包交互建连。三次握手带来的延迟使得创建一个新 TCP 连接代价非常大，所有有了各种连接重用的技术。

但是连接并不是想重用就重用的，在不重用连接的情况下，如何减少新建连接代理的性能损失呢？

于是人们提出了 TCP 快速打开（TCP Fast Open，TFO），尽可能降低握手对网络延迟的影响。今天我们就讲讲这其中的原理。



##概念

TFO 是在原来 TCP 协议上的扩展协议，它的主要原理就在发送第一个 SYN 包的时候就开始传数据了，不过它要求当前客户端之前已经完成过「正常」的三次握手。快速打开分两个阶段：**请求 Fast Open Cookie** 和 **真正开始 TCP Fast Open**。



**请求 Fast Open Cookie 的过程如下：**

- 客户端发送一个 SYN 包，头部包含 Fast Open 选项，且该选项的Cookie 为空，这表明客户端请求 Fast Open Cookie
- 服务端收取 SYN 包以后，生成一个 cookie 值（一串字符串）
- 服务端发送 SYN + ACK 包，在 Options 的 Fast Open 选项中设置 cookie 的值
- 客户端缓存服务端的 IP 和收到的 cookie 值

![image-20191003115201418](https://tva1.sinaimg.cn/large/006y8mN6gy1g7kvdru3xuj30w60tk473.jpg)



第一次过后，客户端就有了缓存在本地的 cookie 值，后面的握手和数据传输过程如下：

- 客户端发送 SYN 数据包，里面包含数据和之前缓存在本地的 Fast Open Cookie。（注意我们此前介绍的所有 SYN 包都不能包含数据）
- 服务端检验收到的 TFO Cookie 和传输的数据是否合法。如果合法就会返回 SYN + ACK 包进行确认并将数据包传递给应用层，如果不合法就会丢弃
- 服务端程序收到数据以后，可以在握手完成之前发送响应数据给客户端了
- 客户端发送 ACK 包，确认第二步的 SYN 包和数据（如果有的话）
- 后面的过程就跟非 TFO 连接过程一样了

![image-20191003115409605](https://tva1.sinaimg.cn/large/006y8mN6gy1g7kvfzmgzzj30uu0ts7dj.jpg)





## TCP Fast Open 的优势

在开启 TCP Fast Open以后，从第二次请求开始，就可以在一个 RTT 时间拿到响应的数据。

![image-20191003115524747](https://tva1.sinaimg.cn/large/006y8mN6gy1g7kvhanbqyj312q0n84c3.jpg)

