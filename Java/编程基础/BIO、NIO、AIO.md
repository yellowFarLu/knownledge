

# BIO、NIO、AIO





## 同步阻塞IO（BIO）



### 原理

同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。

![image-20200401115955279](https://tva1.sinaimg.cn/large/00831rSTgy1gde4rt054jj311w0ac786.jpg)



### 示例

```java
// 客户端
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
public class Client {
    final static String ADDRESS = "127.0.0.1";
    final static int PORT = 7788;
    public static void main(String[] args)  {
        Socket socket = null;
        BufferedReader in = null;
        PrintWriter out = null;
            socket = new Socket(ADDRESS， PORT);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream()， true);

            //向服务器端发送数据
            out.println("接收到客户端的请求数据...");
            out.println("接收到客户端的请求数据1111...");
            String response = in.readLine();
            System.out.println("Client: " + response);
         ...
    }
}
```

```java
// 服务端
public class Server {
    final static int PROT = 7788;
    public static void main(String[] args) {
        ServerSocket server = null;
            server = new ServerSocket(PROT);
            System.out.println(" server start .. ");
            //进行阻塞
            Socket socket = server.accept();
            //新建一个线程执行客户端的任务
            new Thread(new ServerHandler(socket)).start();
        }
            }

    ServerHandler.java  如下:
    public class ServerHandler implements Runnable{

    private Socket socket ;

    public ServerHandler(Socket socket){
        this.socket = socket;
    }

    @Override
    public void run() {
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
            out = new PrintWriter(this.socket.getOutputStream()， true);
            String body = null;
            while(true){
                body = in.readLine();
                if(body == null) break;
                System.out.println("Server :" + body);
                out.println("服务器端回送响的应数据.");
            }
    }
}
```

上面这个代码很简单，转换成图型说明就是web浏览器发一个请求过来，web服务器就要new 一个线程来处理这个请求，这是传统的请求处理模型。

这也就引来一个很大的问题，当请求越多，服务器端的启用线程也要越多，我们都知道linux(window)的文件句柄数有是限的，默认是1024，当然可以修改，上限好像是65536 (一个柄也相当于一个socket也相当于一个thread，linux查看文件句柄Unlimit -a)。

 其实在实际当中只要并发到1000上下响应请求就会很慢了，所以这种模型是有问题的，这种也就是同步阻塞IO编程（JAVA BIO） 。线程越多，所占用的内存越多，而且线程上下文切换所需的开销也越大。



上面的BIO是有问题的，这个问题无非就是服务器端的线程无限制的增长才会导致服务器崩掉，那我们就对征下药，加个线程池限制线程的生成，又可以复用空闲的线程，是的，在jdk1.5也是这样做的，下面是服务器端改进后的代码:

```java
public class Server {

    final static int PORT = 7788;

    public static void main(String[] args) {
        ServerSocket server = null;
        BufferedReader in = null;
        PrintWriter out = null;
            server = new ServerSocket(PORT);
            System.out.println("server start");
            Socket socket = null;
            HandlerExecutorPool executorPool = new HandlerExecutorPool(50， 1000);
            while(true){
                socket = server.accept();
                executorPool.execute(new ServerHandler(socket));
            }
        }
        }
}
```

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class HandlerExecutorPool {

    private ExecutorService executor;
    public HandlerExecutorPool(int maxPoolSize， int queueSize){
        this.executor = new ThreadPoolExecutor(
                Runtime.getRuntime().availableProcessors()，
                maxPoolSize， 
                120L， 
                TimeUnit.SECONDS，
                new ArrayBlockingQueue<Runnable>(queueSize));
    }

    public void execute(Runnable task){
        this.executor.execute(task);
    }
}
```

Jdk1.5创造了一个假的NIO，用一个线程池来限定了线程数量，但只是解决了服务器端不会因为并发太多而死掉，但解决不了**并发大而响应越来越慢的**，到时你也会怀疑你是不是真的用了一个假的nio!!!!!!! 
为了解决这个问题，就要用三板斧来解决!



### 适用场景

只适用于连接数比较小的场景。比如说政府一些网站，访问量比较少，可以使用基于BIO的web服务器实现。



### 缺点

- 同步阻塞IO（BIO）通常会导致通信线程长时间被阻塞。

  









## 同步非阻塞IO（NIO）

针对BIO的几个问题：

- 线程数量多，导致占用内存高，线程上下文切换开销大
- IO请求会阻塞线程，导致并发高的情况下，响应慢

由此产生了NIO。

服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。



### Buffer 缓冲区 

Buffer是一个对象，下面还有ByteBuffer，IntBuffer，LongBuffer等子类。

![image-20200503160426108](https://tva1.sinaimg.cn/large/007S8ZIlgy1gefbo1dqh5j318c0p0gsu.jpg)

它包含一些将要写入或者读出的数据。相比老的IO将数据直接读/写到Stream对象，NIO是将所有数据都用到缓冲区处理，相当于批量操作数据，等数据累计到一定值，再读取/写入，从而提高性能。

它本质上是一个数组，提供了位置，容量，上限等操作方法，还是直接看代码代码来得直接。

```java
InetSocketAddress address = new InetSocketAddress("127.0.0.1"， 7788);//创建连接的地址
        SocketChannel sc = null;//声明连接通道
        ByteBuffer buf = ByteBuffer.allocate(1024);//建立缓冲区
            sc = SocketChannel.open();//打开通道
            sc.connect(address);//进行连接
            while(true){
                //定义一个字节数组，然后使用系统录入功能：
                byte[] bytes = new byte[1024];
                System.in.read(bytes);
                buf.put(bytes);//把数据放到缓冲区中
                buf.flip();//对缓冲区进行复位
                sc.write(buf);//写出数据
                buf.clear();//清空缓冲区数据
            }
...
```



### Channel 通道

Channel是一个通道，支持通过Channel来读取或者写入数据，如自来水管一样，网络数据通过Channel来读取或写入。

通道和流最大的不同是：通道是双向的，而流是一个方向上移动，通道可用于读、写或者同时用于读写。

因为Channel是全双工的，所以它比流能更好的映射操作系统底层的API。特别是在UNIX网络编程模型中，底层操作系统的通道都是全双工的，同时支持读写操作。

它还可以和下面要讲的selector结合起来，有多种状态位，方便selector去识别. 通道分两类，一：网络读写（selectableChannel)，另一类是文件操作(FileChannel)，我们常用的是上面例子中的网络读写。







### Selector 多路复用选择器

多路复用器Selecotr提供选择已经就绪的任务的能力。Selector会不断轮询注册在其上的通道(Channel)，如果某个通道有新的TCP连接、或者读写事件，这个通道就处于就绪状态，会被selector轮询出来，然后通过selectionKey可以取得就绪的Channel集合，从而进行后续的IO操作。

一个多路复用器(Selector)可以同时轮询多个Channel，没有上限，这也是JDK使用epoll代替了传统的selector实现，获得连接句柄没有限制。这也意味着我们只要一个线程负责selector的轮询，就可以接入成千上万个客户端，这是JDK NIO库的巨大进步。

![image-20200401121006395](https://tva1.sinaimg.cn/large/00831rSTgy1gde52cshghj311g0c0jx1.jpg)



### 示例

```java
// 服务端
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.Iterator;
import java.util.Set;

/**
 * @author huangy on 2020-05-03
 */
public class MultiplexerTimeServer implements Runnable {

    private Selector selector;

    private ServerSocketChannel serverSocketChannel;

    private volatile boolean stop;

    public MultiplexerTimeServer(int port) {
        try {

            // 初始化多路复用器
            selector = Selector.open();

            // 开启服务端通道
            serverSocketChannel = ServerSocketChannel.open();

            // 设置为非阻塞
            serverSocketChannel.configureBlocking(false);

            // 绑定监听端口（并且设置请求队列的长度为1024）
            serverSocketChannel.socket().bind(new InetSocketAddress(port), 1024);

            // 将serverSocketChannel注册到selector上，监听accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

            System.out.println("server has start in port="  + port);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        // 当没有停止服务器
        while (!stop) {
            try {

                // selector进行轮询
                selector.select();

                // 程序执行到这里，说明有事件就绪了，获取就绪事件对应的key
                Set<SelectionKey> selectionKeys = selector.selectedKeys();

                Iterator<SelectionKey> iterator = selectionKeys.iterator();

                // 处理就绪的IO事件
                while (iterator.hasNext()) {

                    SelectionKey key =  iterator.next();

                    // 要把该事件从集合中删除，避免重复处理
                    iterator.remove();

                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        e.printStackTrace();

                        if (key != null) {
                            /*
                             * 将这个key关联的通道和选择器取消注册，并且不会再监听这个key的事件
                             */
                            key.cancel();

                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    }
                }

            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        // 多路复用器关闭后，所有关联的资源都会被关闭
        if (selector != null) {
            try {
                selector.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            // 处理新接入的请求消息
            if (key.isAcceptable()) {
                ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
                // ssc.accept() 相当于完成了TCP三次握手
                SocketChannel sc = ssc.accept();

                // 新创建的SocketChannel要设置为非阻塞
                sc.configureBlocking(false);

                // 新的连接加到多路复用器上面
                sc.register(selector, SelectionKey.OP_READ);
            }

            if (key.isReadable()) {
                SocketChannel sc = (SocketChannel)key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);

                if (readBytes > 0) {
                    // 回环以重新读取数据
                    readBuffer.flip();
                    // remaining()返回剩余可以读取的字节数
                    byte[] bytes = new byte[readBuffer.remaining()];
                    // 把数据读取到bytes字节数组
                    readBuffer.get(bytes);
                    String body = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("This time server receive order : " + body);

                    // 回写数据
                    doWrite(sc, new Date().toString());
                }
            }
        }
    }

    private void doWrite(SocketChannel socketChannel, String response) throws IOException {
        if (response != null && response.trim().length() > 0) {
            byte[] bytes = response.getBytes(StandardCharsets.UTF_8);
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();

            /*
             * socketChannel是异步非阻塞的，它并不保证一次能把需要发送的字节数组发送完，
             * 此时会出现"写半包"问题。我们需要注册写操作，不断轮询selector将没有发送完的ByteBuffer发送完毕。
             * 此处没有处理"写半包"问题。
             */
            socketChannel.write(writeBuffer);
        }
    }

    public void stop() {
        this.stop = true;
    }
}


/**
 * NIO实现时间服务器
 * @author huangy on 2020-05-03
 */
public class TimeServer {

    public static void main(String[] args) {

        MultiplexerTimeServer server = new MultiplexerTimeServer(10301);

        new Thread(server, "MultiplexerTimeServer-001").start();
    }

}
```

```java
// 客户端
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Set;

/**
 * @author huangy on 2020-05-03
 */
public class TimeClientHandle implements Runnable {

    private String host;

    private int port;

    private Selector selector;

    private SocketChannel socketChannel;

    private volatile boolean stop;

    public TimeClientHandle(String host, int port) {
        this.host = host;
        this.port = port;

        try {
            selector = Selector.open();

            socketChannel = SocketChannel.open();

            socketChannel.configureBlocking(false);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        try {
            doConnect();
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }

        while (!stop) {
            try {

                selector.select();

                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();

                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    iterator.remove();

                    try {
                        handleInput(selectionKey);
                    } catch (Exception e) {
                        e.printStackTrace();
                        if (selectionKey != null) {
                            selectionKey.cancel();

                            if (selectionKey.channel() != null) {
                                selectionKey.channel().close();
                            }
                        }
                    }
                }


            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        if (selector != null) {
            try {
                selector.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {

        if (key.isValid()) {

            // 判断是否连接成功
            SocketChannel socketChannel = (SocketChannel)key.channel();
            if (key.isConnectable()) {
                if (socketChannel.finishConnect()) {
                    socketChannel.register(selector, SelectionKey.OP_READ);
                    doWrite(socketChannel);
                } else {
                    // 连接失败，进程退出
                    System.exit(1);
                }
            }

            if (key.isReadable()) {
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes= socketChannel.read(readBuffer);

                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("client get time=" + body);
                    this.stop = true;
                } else if (readBytes < 0) {
                    // 对端链路关闭
                    key.cancel();
                    socketChannel.close();
                } else {
                    // 读到0字节，忽略
                }
            }
        }
    }

    private void doConnect() throws IOException {
        // 尝试连接到服务器
        boolean connectResult = socketChannel.connect(new InetSocketAddress(host, port));


        if (connectResult) {
            // 如果连接成功，则注册到多路复用器上
            socketChannel.register(selector, SelectionKey.OP_READ);

            // 发送请求消息
            doWrite(socketChannel);

        } else {
            // 注册请求连接事件，继续连接
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
    }

    private void doWrite(SocketChannel socketChannel) throws IOException {
        byte[] bytes = "client query time".getBytes(StandardCharsets.UTF_8);
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        writeBuffer.put(bytes);
        writeBuffer.flip();
        socketChannel.write(writeBuffer);
        if (!writeBuffer.hasRemaining()) {
            System.out.println("client query time success");
        }
    }
}

/**
 * @author huangy on 2020-05-03
 */
public class TimeClient {

    public static void main(String[] args) {
        new Thread(new TimeClientHandle("127.0.0.1", 10301), "TimeClient").start();
    }

}
```









### 适用场景

NIO方式适用于 连接数目多 且 连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。









## 异步非阻塞 AIO (NIO.2)



### 原理

服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由操作系统先完成了再通知服务器应用去启动线程进行处理。对应UNIX网络编程中的事件驱动IO（AIO），他不需要通过多路复用器（Selector）对注册的通道进行轮询操作即可实现异步读写，从而简化了NIO的编程模型。

AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用操作系统参与并发操作，编程比较复杂，JDK7开始支持。

I/O属于底层操作，需要操作系统支持，并发也需要操作系统的支持，所以性能方面不同操作系统差异会比较明显。另外NIO的非阻塞，需要一直轮询，也是一个比较耗资源的。所以出现AIO。

如果你理解了Java NIO ,下面讲的netty也是水到渠成的事,只想说,深水区已过了!

差点忘记还要补下AIO的,这个比NIO先进的技术,最终实现了netty。

这是神一样存在的java nio框架, 这个偏底层的东西,可能你接触较少却又无处不在,比如: 

![image-20200401121211515](https://tva1.sinaimg.cn/large/00831rSTgy1gde54j3nlij31gu0gwq8g.jpg)



AsynchronousSocketChannel和AsynchronousServerSocketChannel都是由JDK底层的线程池负责回调并驱动读写操作。



### 示例

```java
// 服务端
public class TimeServer {

    public static void main(String[] args) {
        AsyncTimeServerHandler timeServer = new AsyncTimeServerHandler(8080);
        new Thread(timeServer, "AsyncTimeServerHandler").start();
    }

}


import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.concurrent.CountDownLatch;

/**
 * @author huangy on 2020-05-03
 */
public class AsyncTimeServerHandler implements Runnable {

    private int port;

    CountDownLatch latch;

    AsynchronousServerSocketChannel asynchronousServerSocketChannel;

    public AsyncTimeServerHandler(int port) {
        this.port = port;

        try {
            // 构建一个异步的服务端通道
            asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
            // 服务端通道绑定监听端口
            asynchronousServerSocketChannel.bind(new InetSocketAddress(this.port));
            System.out.println("time server start in port=" + this.port);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {

        // CountDownLatch的作用是 在完成一组操作之前，当前线程一直阻塞
        // 在本例中，我们让线程在此阻塞，防止服务端执行完成退出。
        // 在实际项目中，不需哟启动独立线程来处理AsynchronousServerSocketChannel，这里仅仅是demo演示
        latch = new CountDownLatch(1);

        doAccept();

        try {
            latch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 接收客户端连接。由于是异步操作，我们可以传递一个CompletionHandler的handler实例接收accept操作成功的
     * 通知消息。
     */
    public void doAccept() {
        asynchronousServerSocketChannel.accept(this, new AcceptCompletionHandler());
    }
}

class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, AsyncTimeServerHandler> {

    @Override
    public void completed(AsynchronousSocketChannel result, AsyncTimeServerHandler attachment) {

        /*
         * 从attachment成员变量获取asynchronousServerSocketChannel，然后继续调用accept方法。
         * 读者可能会疑惑，既然已经接收客户端成功了，为什么还要再次调用accept方法呢？
         * 原因是这样的：服务器可以接收成千上万的连接，那么接收到一个连接以后，会回调CompletionHandler的completed方法，
         * 应该在这个方法里面继续调用asynchronousServerSocketChannel的accept方法，来接收其他客户端连接。
         * 最终形成一个循环，每当接收一个客户端连接成功以后，再异步接收新的客户端连接。
         */
        attachment.asynchronousServerSocketChannel.accept(attachment, this);

        // 连接建立后，服务端读取客户端的消息
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        /*
         * read用于读取请求数据，下面解释下参数
         *
         * ByteBuffer dst：接收缓冲区，用于从异步Channel中读取数据包
         * A attachment：异步Channel携带的附件，通知回调的时候作为入参使用
         * CompletionHandler<Integer,? super A> handler ： 接收通知回调的业务handler
         */
        result.read(buffer, buffer, new ReadCompletionHandler(result));
    }

    @Override
    public void failed(Throwable exc, AsyncTimeServerHandler attachment) {
        exc.printStackTrace();
        attachment.latch.countDown();
    }
}

class ReadCompletionHandler implements CompletionHandler<Integer, ByteBuffer> {

    // 主要用于读取半包消息和应答
    private AsynchronousSocketChannel channel;

    public ReadCompletionHandler(AsynchronousSocketChannel channel) {
        this.channel = channel;
    }

    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        attachment.flip();
        byte[] body = new byte[attachment.remaining()];
        attachment.get(body);

        String req = new String(body, StandardCharsets.UTF_8);
        System.out.println("this time server recevice order, req=" + req);

        // 异步写
        doWrite(new Date().toString());
    }

    private void doWrite(String currentTime) {
        if ((currentTime != null) && currentTime.length() > 0) {
            byte[] bytes = currentTime.getBytes(StandardCharsets.UTF_8);
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();

            channel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {
                    // 如果没有发送完成，继续发送（处理半包问题）
                    if (attachment.hasRemaining()) {
                        channel.write(attachment, attachment, this);
                    }
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    try {
                        channel.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        try {
            channel.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
// 客户端
public class TimeClient {

    public static void main(String[] args) {

        // 在实际项目中，不需要独立的线程创建异步连接对象，因为底层都是通过JDK系统回调实现的
        new Thread(new AsyncTimeClientHandler("127.0.0.1", 8080),
                "AsyncTimeClientHandler").start();
    }

}


import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.CountDownLatch;

/**
 * @author huangy on 2020-05-04
 */
public class AsyncTimeClientHandler implements CompletionHandler<Void, AsyncTimeClientHandler>, Runnable {

    private AsynchronousSocketChannel client;

    private String host;

    private int port;

    private CountDownLatch latch;


    public AsyncTimeClientHandler(String host, int port) {
        this.host = host;
        this.port = port;

        try {
            client = AsynchronousSocketChannel.open();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);

        /*
         * 发起连接服务端的异步操作，下面解析下参数
         * A attachment :  AsynchronousSocketChannel的附件，用于回调的时候作为参数传递，调用者可以自定义
         * CompletionHandler<Void,? super A> handler ： 异步操作通知回调接口，由调用者实现
         */
        client.connect(new InetSocketAddress(host, port), this, this);

        try {
            // 防止异步操作还没有执行完，线程就退出了
            latch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }

        try {
            client.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 异步连接成功以后，回调该方法
     */
    @Override
    public void completed(Void result, AsyncTimeClientHandler attachment) {

        byte[] req = "Client query time".getBytes(StandardCharsets.UTF_8);
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();

        // 连接成功以后，通过write发送请求给服务端，由于是异步，如果没有写完，则通过CompletionHandler继续写
        client.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                if (buffer.hasRemaining()) {
                    // 没有写完，继续写
                    client.write(buffer, buffer, this);

                } else {
                    // 没有剩余的字节，证明写成功了，就开始异步读取结果，读取成功则回调CompletionHandler
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    client.read(readBuffer, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer buffer) {
                            buffer.flip();
                            byte[] bytes = new byte[buffer.remaining()];
                            buffer.get(bytes);
                            String body = new String(bytes, StandardCharsets.UTF_8);
                            System.out.println("receive server time=" + body);

                            // 读取到消息之后，让当前客户端线程退出
                            latch.countDown();
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer buffer) {
                            try {
                                client.close();
                                latch.countDown();
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                        }
                    });
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    client.close();
                    latch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }

    @Override
    public void failed(Throwable exc, AsyncTimeClientHandler attachment) {
        exc.printStackTrace();
        try {
            client.close();
            latch.countDown();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```





### 适用场景

适用于连接数比较多且连接比较长（重操作）的架构，比较相册服务器，充分调用OS参与并发操作，编程比较复杂，jdk7开始支持；







## 总结

![image-20200504094744032](https://tva1.sinaimg.cn/large/007S8ZIlgy1geg6eee5g4j31840gwwmp.jpg)









## 参考

[BIO、NIO、AIO原理](https://www.cnblogs.com/barrywxx/p/8430790.html?from=singlemessage)

[Netty5 用户指南](http://ifeve.com/netty5-user-guide/)

[BIO、NIO、AIO适用场景](https://www.cnblogs.com/hujinshui/p/10368985.html)

[select、poll、eopll的区别l](https://www.cnblogs.com/Anker/p/3265058.html)