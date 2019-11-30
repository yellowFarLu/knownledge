# RST包

在 TCP 协议中 RST 表示复位，用来**异常的**关闭连接，发送 RST 关闭连接时，不必等缓冲区的数据都发送出去，直接丢弃缓冲区中的数据，连接释放进入`CLOSED`状态。而接收端收到 RST 段后，也不需要发送 ACK 确认。



## RST 常见的几种情况



**端口未监听**

这种情况很常见，比如 web 服务进程挂掉或者未启动，客户端使用 connect 建连，都会出现 "Connection Reset" 或者"Connection refused" 错误。![image-20191003162158844](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l36ndbjij30z80fc10q.jpg)

这样机制可以用来检测对端端口是否打开，发送 SYN 包对指定端口，看会不会回复 SYN+ACK 包。如果回复了 SYN+ACK，说明监听端口存在，如果返回 RST，说明端口未对外监听，如下图所示

![image-20191003162223860](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l372s3qgj30z40k078r.jpg)



**一方突然断电重启，之前建立的连接信息丢失，另一方并不知道**

客户端和服务器一开始三次握手建立连接，中间没有数据传输进入空闲状态。这时候服务器突然断电重启，之前主机上所有的 TCP 连接都丢失了，但是客户端完全不知晓这个情况。等客户端有数据有数据要发送给服务端时，服务端这边并没有这条连接的信息，发送 RST 给客户端，告知客户端自己无法处理，你趁早死了这条心吧。

![image-20191003162337654](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l38d8bphj30to0ty10y.jpg)





**调用 close 函数，设置了 SO_LINGER 为 true**

如果设置 SO_LINGER 为 true，linger 设置为 0，当调用 socket.close() 时， close 函数会立即返回，同时丢弃缓冲区内所有数据并立即发送 RST 包重置连接。



## RST 包如果丢失了怎么办？

这是一个比较有意思的问题，首先需要明确 **RST 是不需要确认的**。 下面假定是服务端发出 RST。

在 RST 没有丢失的情况下，发出 RST 以后服务端马上释放连接，进入 CLOSED 状态，客户端收到 RST 以后，也立刻释放连接，进入 CLOSED 状态。

如下图所示

![image-20191003162459819](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l39s7lwzj30sa0lsdja.jpg)

如果 RST 丢失呢？

服务端依然是在发送 RST 以后马上进入`CLOSED`状态，因为 RST 丢失，客户端压根搞不清楚状况，不会有任何动作。等到有数据需要发送时，一厢情愿的发送数据包给服务端。因为这个时候服务端并没有这条连接的信息，会直接回复 RST。

如果客户端收到了这个 RST，就会自然进入`CLOSED`状态释放连接。如果 RST 依然丢失，客户端只是会单纯的数据丢包了，进入数据重传阶段。如果还一直收不到 RST，会在一定次数以后放弃。![image-20191003162538402](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l3agjykoj311i0logv7.jpg)



## Connection reset by peer

Broken pipe 与 Connection reset by peer 错误在网络编程中非常常见，出现的前提都是连接已关闭。

Connection reset by peer 这个错误很好理解，当一个连接已经关闭，读这个链接都会出现`Connection reset`。如果再次往这个连接写数据，就会出现`Broken pipe`。

那 Broken pipe 到底是什么呢？这就要从 SIGPIPE 信号说起。

当一个进程向某个已收到 RST 的套接字执行写操作时，内核向该进程发 送一个 SIGPIPE 信号。该信号的默认行为是终止进程，因此进程一般会捕获这个信号进行处理。不论该进程是捕获了该信号并从其信号处理函数返回，还是简单地忽略该信号，写操作都 将返回 EPIPE 错误（也就Broken pipe 错误）,这也是 Broken pipe 只在写操作中出现的原因。

![image-20191003162754580](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l3cttewej30lq0ocwia.jpg)