# Socket



## Socket选项之SO_REUSEADDR

前面介绍到四次挥手的时候有讲到，**主动断开**连接的那一端需要等待 2 个 MSL 才能最终释放这个连接。一般而言，主动断开连接的都是客户端，如果是服务端程序重启或者出现 bug 崩溃，这时服务端会主动断开连接，如下图所示

![image-20191003120916587](https://tva1.sinaimg.cn/large/006y8mN6gy1g7kvvpls82j313j0u07cj.jpg)

因为要等待 2 个 MSL 才能最终释放连接，所以如果这个时候程序马上启动，就会出现`Address already in use`错误。要过 1 分钟以后才可以启动成功。如果你写了一个 web 服务器，崩溃以后被脚本自动拉起失败，需要等一分钟才正常，运维可能要骂娘了。因此，需要使用 SO_REUSEADDR 参数。

服务端主动断开连接以后，需要等 2 个 MSL 以后才最终释放这个连接，重启以后要绑定同一个端口，默认情况下，操作系统的实现都会阻止新的监听套接字绑定到这个端口上。

启用 SO_REUSEADDR 套接字选项可以解除这个限制，默认情况下这个值都为 0，表示关闭。在 Java 中，reuseAddress 不同的 JVM 有不同的实现，在我本机上，这个值默认为 1 允许端口重用。但是为了保险起见，写 TCP、HTTP 服务一定要主动设置这个参数为 1。



**为什么通常不会在客户端上出现**

因为客户端都是用的临时端口，这些临时端口与处于 TIME_WAIT 状态的端口恰好相同的可能性不大，就算相同换一个新的临时端口就好了。





##Socket选项之SO_LINGER

###关闭连接的两种方式

前面有介绍过有两种方式可以关闭 TCP 连接

- FIN：优雅关闭，发送 FIN 包表示自己这端所有的数据都已经发送出去了，后面不会再发送数据
- RST：强制连接重置关闭，无法做出什么保证

当调用 socket.close() 的时候会发生什么呢？

正常情况下

- 操作系统等所有的数据发送完才会关闭连接
- 因为是主动关闭，所以连接将处于 TIME_WAIT 两个 MSL

前面说了正常情况，那一定有不正常的情况下，如果我们不想等那么久才彻底关闭这个连接怎么办，这就是我们这篇文章介绍的主角 SO_LINGER



###SO_LINGER

Linux 的套接字选项SO_LINGER 用来改变socket 执行 close() 函数时的默认行为。

SO_LINGER 启用时，操作系统开启一个定时器，在定时器期间内发送数据给对端，超过定时时间后，就会直接 RST连接。

SO_LINGER 参数是一个 linger 结构体，代码如下

```
struct linger {
    int l_onoff;    /* linger active */
    int l_linger;   /* how many seconds to linger for */
};
```

第一个字段 l_onoff 用来表示是否启用 linger 特性，非 0 为启用，0 为禁用 ，linux 内核默认为禁用。这种情况下 **close 函数立即返回，操作系统负责把缓冲队列中的数据全部发送至对端**

第二个参数 l_linger 在 l_onoff 为非 0 （即启用特性）时才会生效。

- 如果 l_linger 的值为 0，那么调用 close，close 函数会立即返回，同时丢弃缓冲区内所有数据并立即发送 RST 包重置连接
- 如果 l_linger 的值为非 0，那么此时 close 函数在阻塞直到 l_linger 时间超时或者数据发送完毕，发送队列在超时时间段内继续尝试发送，如果发送完成则皆大欢喜，超时则直接丢弃缓冲区内容 并 RST 掉连接。



#### 示例

```java
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.setReuseAddress(true);
        serverSocket.bind(new InetSocketAddress("192.168.1.4", 9999));

        while (true) {
            Socket socket = serverSocket.accept();
            InputStream input = socket.getInputStream();
            ByteArrayOutputStream output = new ByteArrayOutputStream();
            byte[] buffer = new byte[1];
            int length;
            while ((length = input.read(buffer)) != -1) {
                output.write(buffer, 0, length);
            }
            String req = new String(output.toByteArray(), "utf-8");
            System.out.println(req.length());
            socket.close();
        }
    }
}
```



```java
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.net.SocketAddress;

public class Client {
    private static int PORT = 9999;
    private static String HOST = "192.168.1.4";

    public static void main(String[] args) throws Exception {
        Socket socket = new Socket();

        /*
         * 测试#1: 默认设置
         * close之后，客户端会发送完全部数据，然后等待2MSL
         */
//        socket.setSoLinger(false, 0);

        /*
         * 测试#2
         * close之后，立即关闭连接，如果缓冲区里面有数据也会抛弃
         */
//         socket.setSoLinger(true, 0);

        /*
         * 测试#3 close 函数不会立刻返回，
         * 如果在 1s 内数据传输结束，则皆大欢喜，
         * 如果在 1s 内数据没有传输完，就直接丢弃掉，同时 RST 连接
         */
        socket.setSoLinger(true, 1);

        SocketAddress address = new InetSocketAddress(HOST, PORT);
        socket.connect(address);

        OutputStream output = socket.getOutputStream();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10000; i++) {
            sb.append("hel");
        }
        byte[] request = sb.toString().getBytes("utf-8");
        output.write(request);
        long start = System.currentTimeMillis();
        socket.close();
        long end = System.currentTimeMillis();
        System.out.println("close time cost: " + (end - start));
    }
}
```

