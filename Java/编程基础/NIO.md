# NIO

新的输入/输出 (NIO) 库是在 JDK 1.4 中引入的，弥补了原来的 I/O 的不足，提供了高速的、面向块的 I/O。

NIO 实现了 IO 多路复用中的 Reactor 模型。



## 流与块

I/O 与 NIO 最重要的区别是数据打包和传输的方式，I/O 以流的方式处理数据，而 NIO 以块的方式处理数据。

面向流的 I/O 一次处理一个字节数据：一个输入流产生一个字节数据，一个输出流消费一个字节数据。为流式数据创建过滤器非常容易，链接几个过滤器，以便每个过滤器只负责复杂处理机制的一部分。不利的一面是，面向流的 I/O 通常相当慢。

面向块的 I/O 一次处理一个数据块，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

I/O 包和 NIO 已经很好地集成了，java.io.* 已经以 NIO 为基础重新实现了，所以现在它可以利用 NIO 的一些特性。例如，java.io.* 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。





## 通道

通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动(一个流必须是 InputStream 或者 OutputStream 的子类)，而**通道是双向的，可以用于读、写或者同时用于读写**。

通道包括以下类型：

- FileChannel：从文件中读写数据；
- DatagramChannel：通过 UDP 读写网络中数据；
- SocketChannel：通过 TCP 读写网络中数据；
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。



## 缓冲区

**发送给一个通道的所有数据都必须首先放到缓冲区中**，同样地，**从通道中读取的任何数据都要先读到缓冲区中**。也就是说，不会直接对通道进行读写数据，而是要先经过缓冲区。

缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer



### 缓冲区状态变量

- capacity：最大容量；
- position：当前已经读写的字节数；
- limit：还可以读写的字节数。

状态变量的改变过程举例：

- 新建一个大小为 8 个字节的缓冲区，此时 position 为 0，而 limit = capacity = 8。capacity 变量不会改变，下面的讨论会忽略它。

  <img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxsq3va5j30qg086gmf.jpg" alt="image-20191130120827151" style="zoom:50%;" />

- 从输入通道中读取 5 个字节数据写入缓冲区中，此时 position 为 5，limit 保持不变。

  <img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxta88wyj30po08saau.jpg" alt="image-20191130120859352" style="zoom:50%;" />

- 在将缓冲区的数据写到输出通道之前，需要先调用 flip() 方法，这个方法将 limit 设置为当前 position，并将 position 设置为 0。

  <img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxu1k15fj30po0840tg.jpg" alt="image-20191130120942578" style="zoom:50%;" />

- 从缓冲区中取 4 个字节到输出缓冲中，此时 position 设为 4

  <img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxufxslij30oy08e0tg.jpg" alt="image-20191130121005690" style="zoom:50%;" />

- 最后需要调用 clear() 方法来清空缓冲区，此时 position 和 limit 都被设置为最初位置。

  <img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxuuzropj30pu07g0tf.jpg" alt="image-20191130121030199" style="zoom:50%;" />





### 文件 NIO 实例

```java
public static void fastCopy(String src, String dist) throws IOException {

    /* 获得源文件的输入字节流 */
    FileInputStream fin = new FileInputStream(src);

    /* 获取输入字节流的文件通道 */
    FileChannel fcin = fin.getChannel();

    /* 获取目标文件的输出字节流 */
    FileOutputStream fout = new FileOutputStream(dist);

    /* 获取输出字节流的文件通道 */
    FileChannel fcout = fout.getChannel();

    /* 为缓冲区分配 1024 个字节 */
    ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    while (true) {

        /* 从输入通道中读取数据到缓冲区中 */
        int r = fcin.read(buffer);

        /* read() 返回 -1 表示 EOF */
        if (r == -1) {
            break;
        }

        /* 切换读写 */
        buffer.flip();

        /* 把缓冲区的内容写入输出文件中 */
        fcout.write(buffer);

        /* 清空缓冲区 */
        buffer.clear();
    }
}
```











## Selecor



### 概要

NIO 常常被叫做非阻塞 IO，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用。

NIO 实现了 IO 多路复用中的 Reactor 模型，**一个线程使用一个选择器 Selector 通过轮询的方式去监听多个通道 Channel 上的事件，从而让一个线程就可以处理多个事件**。

通过**配置监听的通道 Channel 为非阻塞**，那么当 Channel 上的 IO 事件还未到达时，selector就不会进入阻塞状态一直等待，而是继续轮询其它 Channel，找到 IO 事件已经到达的 Channel 执行。

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件，对于 IO 密集型的应用具有很好地性能。

应该注意的是，只有套接字 Channel 才能配置为非阻塞，而 FileChannel 不能，为 FileChannel 配置非阻塞也没有意义（FileChannel是本地读取数据，肯定不会阻塞）。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxwspn5zj30hy0cmaaz.jpg" alt="image-20191130121221604" style="zoom:67%;" />



### 流程

**创建选择器**

```java
Selector selector = Selector.open();
```



**将通道注册到选择器上**

```java
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
```

通道必须配置为非阻塞模式，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么当前线程就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。



在将通道注册到选择器上时，还需要指定要注册的具体事件，主要有以下几类：

- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE



**监听事件**

```java
int num = selector.select();
```

使用 select() 来监听到达的事件，它会一直阻塞直到有至少一个事件到达。



**获取到达的事件**

```java
Set<SelectionKey> keys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = keys.iterator();
while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key.isAcceptable()) {
        // ...
    } else if (key.isReadable()) {
        // ...
    }
    keyIterator.remove();
}
```



**事件循环**

因为服务器端有可能需要一直监听事件，因此服务器端处理事件的代码一般会放在一个死循环内。

```java
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```



### 套接字 NIO 实例

```java
public class NIOServer {

    public static void main(String[] args) throws IOException {

        Selector selector = Selector.open();

        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);

        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {

            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {

                    ServerSocketChannel ssChannel1 = (ServerSocketChannel) key.channel();

                    // 服务器会为每个新连接创建一个 SocketChannel
                    SocketChannel sChannel = ssChannel1.accept();
                    sChannel.configureBlocking(false);

                    // 这个新连接主要用于从客户端读取数据
                    sChannel.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {

                    SocketChannel sChannel = (SocketChannel) key.channel();
                    System.out.println(readDataFromSocketChannel(sChannel));
                    sChannel.close();
                }

                keyIterator.remove();
            }
        }
    }

    private static String readDataFromSocketChannel(SocketChannel sChannel) throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        StringBuilder data = new StringBuilder();

        while (true) {

            buffer.clear();
            int n = sChannel.read(buffer);
            if (n == -1) {
                break;
            }
            buffer.flip();
            int limit = buffer.limit();
            char[] dst = new char[limit];
            for (int i = 0; i < limit; i++) {
                dst[i] = (char) buffer.get(i);
            }
            data.append(dst);
            buffer.clear();
        }
        return data.toString();
    }
}
```

```java
public class NIOClient {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        String s = "hello world";
        out.write(s.getBytes());
        out.close();
    }
}
```





## 内存映射文件

内存映射文件 I/O 是一种读和写文件数据的方法，它比常规的基于流或者基于通道的 I/O 快得多。

向内存映射文件写入可能是危险的，只是改变数组的单个元素这样的简单操作，**就可能会直接修改磁盘上的文件**。修改数据与将数据保存到磁盘是没有分开的。

下面代码行将文件的前 1024 个字节映射到内存中，map() 方法返回一个 MappedByteBuffer，它是 ByteBuffer 的子类。因此，可以像使用其他任何 ByteBuffer 一样使用新映射的缓冲区，操作系统会在需要时负责执行映射。

```java
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);
```











## IO模型概念



### 阻塞 / 非阻塞

指访问某个函数时是否会阻塞线程



### 同步 / 异步

**同步 / 异步描述的是读写数据的方式**。

- 同步主要指由用户线程参与读写
- 异步指的是内核线程发起读写，用户线程只需要关注IO完成后的回调，不需要参与到具体的IO之中。



### 同步阻塞IO

用户线程参与读写，并且会阻塞用户线程。即传统的BIO。



### 同步非阻塞IO

用户线程来参与读写，但是用户线程调用读写方法是不阻塞的，立刻返回的。



### 异步非阻塞IO

用户调用读写方法是不阻塞的，立刻返回（而且用户不需要关注读写，只需要提供回调操作）。

内核线程在**完成**读写后回调用户提供的回调方法。

即经典的Proactor设计模式。



## 多路复用IO

即经典的Reactor设计模式，Java中的Selector和Linux中的epoll都是这种模型。

非阻塞 IO 有个问题，那就是线程要读数据，有多少读多少，结果读了一部分就返回了，线程如何知道何时才应该继续读。也就是当数据到来时，线程如何得到通知。写也是一样，如果缓冲区满了，写不完，剩下的数据何时才应该继续写，线程也应该得到通知。

在多路复用IO模型中，会有一个线程（Java中的Selector）不断去轮询多个socket的状态，**只有当socket真正有读写事件时，才调用工作线程去完成IO读写操作**。因此在多路复用IO模型中，只需要使用一个线程就可以管理多个socket，系统不需要建立新的进程或者线程，也不必维护这些线程和进程，并且只有在真正有socket读写事件进行时，才会让线程去读写，所以它大大减少了资源占用。



#### Reactor模式

**IO多路复用模型使用了Reactor设计模式实现了这一机制。**

![image-20191103165848900](https://tva1.sinaimg.cn/large/006y8mN6gy1g8kygkj5l7j317w0980xa.jpg)

有专门一个线程, 即 acceptor 线程用于监听客户端的TCP连接请求。

客户端连接的 IO 操作都是由一个特定的 NIO 线程池负责，**每个客户端连接都与一个特定的 NIO 线程绑定**，因此在这个客户端连接中的所有 IO 操作都是在同一个线程中完成的。

客户端连接有很多, 但是 NIO 线程数是比较少的, 因此**一个 NIO 线程可以同时绑定到多个客户端连接中**。





## select、poll、epoll

select，poll，epoll都是IO多路复用的机制。

I/O多路复用就通过其中一种机制，可以监视多个**文件描述符(fd)**，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。**但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。 

select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，**但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中**，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候**只要判断一下就绪链表是否为空就行了**，这节省了大量的CPU时间。这就是回调机制带来的性能提升。





## 参考

[IO多路复用](https://www.cnblogs.com/zwt1990/p/8821185.html)

[IO同步阻塞等概念](https://www.xuebuyuan.com/3206055.html)

[IO和NIO]([https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20IO.md](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java IO.md))

[select、poll、epoll](https://www.cnblogs.com/aspirant/p/9166944.html)