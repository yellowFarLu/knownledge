# nagle 算法

简单来讲 nagle 算法讲的是避免发送端频繁的发送小包给对方。

Nagle 算法要求，当一个 TCP 连接中有在传数据（已经发出但还未确认的数据）时，小于 MSS 的报文段就不能被发送，直到所有的在传数据都收到了 ACK。同时收到 ACK 后，TCP 还不会马上就发送数据，会收集小包合并一起发送。网上有人想象的把 Nagle 算法说成是「hold 住哥」，我觉得特别形象。

算法思路如下：

```java
if there is new data to send
  if the window size >= MSS and available data is >= MSS
    send complete MSS segment now
  else
    if there is unconfirmed data still in the pipe
      enqueue data in the buffer until an acknowledge is received
    else
      send data immediately
    end if
  end if
end if
```

默认情况下 Nagle 算法都是启用的，Java 可以通过 `setTcpNoDelay(true);`来禁用 nagle 算法。

举个例子：

```java
public class NagleClient {
    public static void main(String[] args) throws Exception {
        Socket socket = new Socket();
      
        // 默认为false，即使用nagle算法；true则为禁用nagle算法
        ## socket.setTcpNoDelay(true);
      
        SocketAddress address = new InetSocketAddress("c1", 9999);
        socket.connect(address);
        OutputStream output = socket.getOutputStream();
        byte[] request = new byte[10];
    
        for (int i = 0; i < 5; i++) {
            output.write(request);
        }
        TimeUnit.SECONDS.sleep(1);
        socket.close();
    }
}
```

开启nagle算法的结果：

可以看到除了第一个包是单独发送，后面的四个包合并到了一起



禁用nagle算法的结果：

几乎同一瞬间分 5 次把数据发送了出去，不管之前发出去的包有没有收到 ACK

![image-20191003220958290](https://tva1.sinaimg.cn/large/006y8mN6gy1g7ld8qkv2fj30ya0t2qbm.jpg)



## 典型的小包场景：SSH

一个典型的大量小包传输的场景是用 ssh 登录另外一台服务器，每输入一个字符，服务端也随即进行回应，客户端收到了以后才会把输入的字符和响应的内容显示在自己这边。比如登录服务器后输入`ls`然后换行，中间包交互的过程如下图

![image-20191003221410480](https://tva1.sinaimg.cn/large/006y8mN6gy1g7ldd422ssj30iu0sijye.jpg)

1. 客户端输入`l`，字符 `l` 被加密后传输给服务器
2. 服务器收到`l`包，回复被加密的 `l` 及 ACK
3. 客户端输入`s`，字符 `s` 被加密后传输给服务器
4. 服务器收到`s`包，回复被加密的 `s` 及 ACK
5. 客户端输入 enter 换行符，换行符被加密后传输给服务器
6. 服务器收到换行符，回复被加密的换行符及 ACK
7. 服务端返回执行 ls 的结果
8. 客户端回复 ACK



## Nagle 算法的意义在哪里

Nagle 算法的作用是减少小包在客户端和服务端直接传输，一个包的 TCP 头和 IP 头加起来至少都有 40 个字节，如果携带的数据比较小的话，那就非常浪费了。就好比开着一辆大货车运一箱苹果一样。

![image-20191003221446761](https://tva1.sinaimg.cn/large/006y8mN6gy1g7lddqh51ej30y00e8tad.jpg)

Nagle 算法在通信时延较低的场景下意义不大。在 Nagle 算法中 ACK 返回越快，下次数据传输就越早。

假设 RTT 为 10ms 且没有延迟确认（这个后面会讲到），那么你敲击键盘的间隔大于 10ms 的话就不会触发 Nagle 的条件：只有接收到所有的在传数据的 ACK 后才能继续发数据，也即如果所有的发出去的包 ACK 都收到了，就不用等了。如果你想触发 Nagle 的停等（stop-wait）机制，1s 内要输入超过 100 个字符。因此如果在局域网内，Nagle 算法基本上没有什么效果。

如果客户端到服务器的 RTT 较大，比如多达 200ms，这个时候你只要1s 内输入超过 5 个字符，就有可能触发 Nagle 算法了。

**Nagle 算法是时代的产物**：Nagle 算法出现的时候网络带宽都很小，当有大量小包传输时，很容易将带宽占满，出现丢包重传等现象。因此对 ssh 这种交互式的应用场景，选择开启 Nagle 算法可以使得不再那么频繁的发送小包，而是合并到一起，代价是稍微有一些延迟。现在的 ssh 客户端已经默认关闭了 Nagle 算法。



Nagle 算法是应用在发送端的，简而言之就是，对发送端而言：

- 当第一次发送数据时不用等待，就算是 1byte 的小包也立即发送
- 后面发送数据时需要累积数据包直到满足下面的条件之一才会继续发送数据：
  - 数据包达到最大段大小MSS
  - 接收端收到之前数据包的确认 ACK