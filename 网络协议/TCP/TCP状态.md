# TCP状态



##CLOSED

这个状态是一个「假想」的状态，是 TCP 连接还未开始建立连接或者连接已经彻底释放的状态。因此`CLOSED`状态也无法通过 `netstat` 或者 `lsof` 等工具看到。

从 CLOSE 状态转换为其它状态有两种可能：主动打开（Active Open）和被动打开（Passive Open）

- 被动打开：一般来说，服务端会监听一个特定的端口，等待客户端的新连接，同时会进入`LISTEN`状态，这种被称为「被动打开」
- 主动打开：客户端主动发送一个`SYN`包准备三次握手，被称为「主动打开（Active Open）」





## LISTEN

服务端调用listen方法监听特定端口时进入到`LISTEN`状态，等待客户端发送 `SYN` 报文三次握手建立连接。

处于`LISTEN`状态的连接收到`SYN`包以后会发送 `SYN+ACK` 给对端，同时进入`SYN-RCVD`阶段



##SYN-SENT

客户端发送SYN报文后，就会进入SYN-SENT状态。同时开启定时器，如果超时没有收到ACK，就会重发SYN报文。



## SYN-RCVD

服务端收到`SYN`报文以后会回复 `SYN+ACK`，然后等待对端 ACK 的时候进入`SYN-RCVD`



##ESTABLISHED

`SYN-SENT`或者`SYN-RCVD`状态的连接收到对端确认`ACK`以后进入`ESTABLISHED`状态，连接建立成功。

`ESTABLISHED`状态的连接有两种可能的状态转换方式:

- 调用 close 方法主动关闭连接，这个时候会发送 FIN 包给对端，同时自己进入`FIN-WAIT-1`状态
- 收到对端的 FIN 包，执行被动关闭，收到 `FIN` 包以后会回复 `ACK`，同时自己进入`TIME_WAIT`状态



## FIN-WAIT-1

主动关闭的一方发送了 FIN 包，等待对端回复 ACK 时进入`FIN-WAIT-1`状态。

`FIN_WAIT-1`状态的切换如下几种情况：

- 当收到 `ACK` 以后，`FIN-WAIT-1`状态会转换到`FIN-WAIT-2`状态
- 当收到 `FIN` 以后，会回复对端 `ACK`，`FIN-WAIT-1`状态会转换到`CLOSING`状态（？？？）
- 当收到 `FIN+ACK` 以后，会回复对端 `ACK`，`FIN-WAIT-1`状态会转换到`TIME_WAIT`状态，跳过了`FIN-WAIT-2`状态





## FIN-WAIT-2

处于 `FIN-WAIT-1`状态的连接收到 ACK 确认包以后进入`FIN-WAIT-2`状态，这个时候主动关闭方的 FIN 包已经被对方确认，等待被动关闭方发送 FIN 包。



##CLOSE-WAIT

当有一方想关闭连接的时候，调用 close 等系统调用关闭 TCP 连接会发送 FIN 包给对端（被动关闭方），被动关闭方收到 FIN 包以后进入`CLOSE-WAIT`状态。

当被动关闭方有数据要发送给对端的时候，可以继续发送数据。当没有数据发送给对方时，也会调用 close 等系统调用关闭 TCP 连接，发送 FIN 包给主动关闭的一方，同时进入`LAST-ACK`状态



##TIME-WAIT

`TIME-WAIT`可能是所有状态中面试问的最频繁的一种状态了。当收到了被动关闭方的 FIN 包，发送确认 ACK 给对端，进入TIME-WAIT状态，然后开启 2MSL 定时器，定时器到期时进入 `CLOSED` 状态，连接释放。



首先，我们需要明确，**只有主动断开的那一方才会进入 TIME_WAIT 状态**，且会在那个状态持续 2 个 MSL（Max Segment Lifetime）。

为了讲清楚 TIME_WAIT，需要先介绍一下 MSL 的概念。

### MSL

MSL(Max Segement Lifetime)报文最大生存时间。这个值与 IP 报文头的 TTL 字段有密切的关系。

![image-20191003153404554](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l1subqluj311i0aa0xr.jpg)

IP 报文头中有一个 8 位的存活时间字段（Time to live, TTL）如下图。 这个存活时间存储的不是具体的时间，而是一个 IP 报文最大可经过的路由数，每经过一个路由器，TTL 减 1，当 TTL 减到 0 时这个 IP 报文会被丢弃。

一个报文从源主机到目的主机之间TTL 经过路由器不断减小的过程如下图所示：

假设初始的 TTL 为 12，经过下一个路由器 R1 以后 TTL 变为 11，后面每经过一个路由器以后 TTL 减 1

![image-20191003153522087](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l1u5spl1j311w074dk1.jpg)

从上面可以看到 TTL 说的是「跳数」限制而不是「时间」限制，尽管如此我们依然假设**最大跳数的报文在网络中存活的时间不可能超过 MSL 秒**。Linux 的套接字实现假设 MSL 为 30 秒，因此在 Linux 机器上 TIME_WAIT 状态将持续 60秒。（MSL是一个假设的时间，假设所有报文的存活时间不能超过MSL）



### TIME_WAIT 存在的原因是什么



####第一个原因

**让旧连接的包在网络中消逝**

数据报文可能在发送途中延迟但最终会到达，因此要等老的“迷路”的重复报文段在网络中过期失效，这样可以避免用**相同**源端口和目标端口创建新连接时收到旧连接姗姗来迟的数据包，造成数据错乱。 

比如下面的例子：![image-20191003153942310](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l1ynsiyzj30t00m412m.jpg)

假设客户端 10.211.55.2 的 61594 端口与服务端 10.211.55.10 的 8080 端口一开始建立了一个 TCP 连接。

假如服务端发送完 FIN 包以后不等待直接进入 CLOSED 状态，老连接 SEQ=3 的包因为网络的延迟。过了一段时间**相同**的 IP 和端口号又新建了另一条连接，这样 TCP 连接的四元组就完全一样了。恰好 SEQ 因为回绕等原因也正好相同，那么 SEQ=3 的包就无法知道到底是旧连接的包还是新连接的包了，造成新连接数据的混乱。

**TIME_WAIT 等待时间是 2 个 MSL，已经足够让一个方向上的包最多存活 MSL 秒就被丢弃，保证了在创建新的 TCP 连接以后，老连接姗姗来迟的包已经在网络中被丢弃消逝，不会干扰新的连接。**



#### 第二个原因

**处理最后 ACK 丢失的情况**

第二个原因是确保可靠实现 TCP 全双工终止连接。关闭连接的四次挥手中，最终的 ACK 由主动关闭方发出，如果这个 ACK 丢失，对端（被动关闭方）将重发 FIN，如果主动关闭方不维持 TIME_WAIT 直接进入 CLOSED 状态，则无法重传 ACK，被动关闭方因此不能及时可靠释放。

![image-20191003154304616](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2265dmyj30z00u0gwi.jpg)





###为什么时间是两个 MSL

- 1 个 MSL 确保四次挥手中主动关闭方最后的 ACK 报文最终能达到对端
- 1 个 MSL 确保对端没有收到 ACK 重传的 FIN 报文可以到达

2MS = 去向 ACK 消息最大存活时间（MSL) + 来向 FIN 消息的最大存活时间（MSL）





### TIME_WAIT 的问题

在一个非常繁忙的服务器上，如果有大量 TIME_WAIT 状态的连接会怎么样呢？

- 连接表无法复用
- socket 结构体内存占用

简而言之，就是占用系统资源，通俗一点来讲就是“占着茅坑不拉屎”。

假设主动断开的一方是客户端，对于 web 服务器而言，目标地址、目标端口都是固定值，客户端的 IP 也是固定的，那么能变化的就只有端口了，在一台 Linux 机器上，端口最多是 65535 个（ 2 个字节）。如果客户端与服务器通信全部使用短连接，不停的创建连接，接着关闭连接，会造成大量的 TCP 连接进入 TIME_WAIT 状态。



#### 应对 TIME_WAIT 的各种操作

针对 TIME_WAIT 持续时间过长的问题，Linux 新增了几个相关的选项，net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_tw_recycle。下面我们来说明一下这两个参数的用意。 这两个参数都依赖于 TCP 头部的扩展选项：timestamp



#### TCP 头部时间戳选项

时间戳选项（TCP Timestamps Option，TSopt）属于TCP首部的选项的一种。![image-20191003155502627](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2emqkjpj312k0c0423.jpg)

它由四部分构成：类别（kind）、长度（Length）、发送方时间戳（TS value）、回显时间戳（TS Echo Reply）。

时间戳选项类别（kind）的值等于 8，用来与其它类型的选项区分。

长度（length）等于 10。

两个时间戳相关的字段大小都是 4 字节。

如下图所示：

![image-20191003155559935](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2fmcu3rj310w08041a.jpg)

是否使用时间戳选项是在三次握手里面的 SYN 报文里面确定的。

![image-20191003155815048](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2hynr9kj30ko0vu143.jpg)

有几个需要说明的点

- 时间戳是一个单调递增的值，与我们所知的 epoch 时间戳不是一回事。这个选项不要求两台主机进行时钟同步
- timestamps 是一个双向的选项，如果只要有一方不开启，双方都将停用 timestamps。

有了这个选项，我们来看一下 tcp_tw_reuse 选项



####tcp_tw_reuse 选项

缓解紧张的端口资源，一个可行的方法是重用“浪费”的处于 TIME_WAIT 状态的连接，当开启 net.ipv4.tcp_tw_reuse 选项时，处于 TIME_WAIT 状态的连接可以被重用。下面把主动关闭方记为 A， 被动关闭方记为 B，它的原理是：

- 如果主动关闭方 A 收到的包时间戳比当前存储的时间戳小，说明是一个迷路的旧连接的包，直接丢弃掉
- 如果因为 ACK 包丢失导致被动关闭方还处于`LAST-ACK`状态，这时 A 发送SYN 包想三次握手建立连接，这个时候处于 `LAST-ACK` 阶段的被动关闭方 B 会回复 FIN，因为这时 A 处于`SYN-SENT`阶段会回以一个 RST 包给 B，B 这端的连接会进入 CLOSED 状态，A 因为没有收到 SYN 包的 ACK，会重传 SYN，后面就一切顺利了。

![image-20191003160846045](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2swg01hj310y0u0n7q.jpg)





#### tcp_tw_recyle 选项

tcp_tw_recyle 是一个比 tcp_tw_reuse 更激进的方案， 系统会缓存每台主机（即 IP）连接过来的最新的时间戳。对于新来的连接，如果发现 SYN 包中带的时间戳与之前记录的来自同一主机的同一连接的分组所携带的时间戳相比更旧，则直接丢弃。如果更新则接受复用 TIME-WAIT 连接。

这种机制在客户端与服务端一对一的情况下没有问题，如果经过了 NAT 或者负载均衡，问题就很严重了。

什么是 NAT呢？

![image-20191003161103516](https://tva1.sinaimg.cn/large/006y8mN6gy1g7l2va62kwj311a0qkgve.jpg)

NAT（Network Address Translator）的出现是为了缓解 IP 地址耗尽的临时方案，IPv4 的地址是 32 位，全部利用最 多只能提 42.9 亿个地址，去掉保留地址、组播地址等剩下的只有 30 多亿，互联网主机数量呈指数级的增长，如果给每个设备都分配一个唯一的 IP 地址，那根本不够。于是 1994 年推出的 NAT 规范，NAT 设备负责维护局域网私有 IP 地址和端口到外网 IP 和端口的映射规则。

它有两个明显的优点

- 出口 IP 共享：通过一个公网地址可以让许多机器连上网络，解决 IP 地址不够用的问题
- 安全隐私防护：实际的机器可以隐藏自己真实的 IP 地址 当然也有明显的弊端：NAT 会对包进行修改，有些协议无法通过 NAT。

当 tcp_tw_recycle 遇上 NAT 时，因为客户端出口 IP 都一样，会导致服务端看起来都在跟同一个 host 打交道。不同客户端携带的 timestamp 只跟自己相关，如果一个时间戳较大的客户端 A 通过 NAT 与服务器建连，时间戳较小的客户端 B 通过 NAT 发送的包服务器认为是过期重复的数据，直接丢弃，导致 B 无法正常建连和发数据。













##LAST-ACK

`LAST-ACK` 顾名思义等待最后的 ACK。是被动关闭的一方，发送 FIN 包给对端 等待 ACK 确认时的状态。

当收到 ACK 以后，进入 `CLOSED` 状态，连接释放。



## CLOSING

`CLOSING`状态在「同时关闭」的情况下出现。这里的同时关闭中的「同时」其实并不是时间意义上的同时，而是指的是在发送 FIN 包还未收到确认之前，收到了对端的 FIN 的情况。