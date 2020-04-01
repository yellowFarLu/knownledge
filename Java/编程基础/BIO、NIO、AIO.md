

# BIO、NIO、AIO





## 同步阻塞IO（BIO）



### 原理

同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。

![image-20200401115955279](https://tva1.sinaimg.cn/large/00831rSTgy1gde4rt054jj311w0ac786.jpg)

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

Jdk1.5创造了一个假的NIO，用一个线程池来限定了线程数量，但只是解决了服务器端不会因为并发太多而死掉，但解决不了并发大而响应越来越慢的，到时你也会怀疑你是不是真的用了一个假的nio!!!!!!! 
为了解决这个问题，就要用三板斧来解决!



### 适用场景

只适用于连接数比较小的场景。比如说政府一些网站，访问量比较少，可以使用基于BIO的web服务器实现。









## 同步非阻塞IO（NIO）

针对BIO的几个问题：

- 线程数量多，导致占用内存高，线程上下文切换开销大
- IO请求会阻塞线程，导致并发高的情况下，响应慢

由此产生了NIO。

服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。



### Buffer 缓冲区 

Buffer是一个抽象的对象，下面还有ByteBuffer，IntBuffer，LongBuffer等子类，相比老的IO将数据直接读/写到Stream对象，NIO是将所有数据都用到缓冲区处理，相当于批量操作数据，等数据累计到一定值，再读取/写入，从而提高性能。

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

如自来水管一样，支持网络数据从Channel中读写，通道和流最大的不同是：通道是双向的，而流是一个方向上移动(InputStream/OutputStream)，通道可用于读/写或读写同时进行，它还可以和下面要讲的selector结合起来，有多种状态位，方便selector去识别. 通道分两类，一：网络读写（selectableChannel)，另一类是文件操作(FileChannel)，我们常用的是上面例子中的网络读写!



### Selector 多路复用选择器 

多路复用选择器Selector会不断轮询注册在其上的通道(Channel)，如果某个通道发生了读写操作，这个通道处于就绪状态，会被selector轮询出来，然后通过selectionKey可以取得就绪的Channel集合，从而进行后续的IO操作。
一个多路复用器(Selector)可以负责成千上万个Channel，没有上限，这也是JDK使用epoll代替了传统的selector实现，获得连接句柄没有限制。这也意味着我们只要一个线程负责selector的轮询，就可以接入成千上万个客户端，这是JDK，NIO库的巨大进步。

![image-20200401121006395](https://tva1.sinaimg.cn/large/00831rSTgy1gde52cshghj311g0c0jx1.jpg)

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

public class Server implements Runnable{
    //1 多路复用器（管理所有的通道）
    private Selector seletor;
    //2 建立缓冲区
    private ByteBuffer readBuf = ByteBuffer.allocate(1024);
    //3 
    private ByteBuffer writeBuf = ByteBuffer.allocate(1024);
    public Server(int port){
        try {
            //1 打开路复用器
            this.seletor = Selector.open();
            //2 打开服务器通道
            ServerSocketChannel ssc = ServerSocketChannel.open();
            //3 设置服务器通道为非阻塞模式
            ssc.configureBlocking(false);
            //4 绑定地址
            ssc.bind(new InetSocketAddress(port));
            //5 把服务器通道注册到多路复用器上，并且监听阻塞事件
            ssc.register(this.seletor, SelectionKey.OP_ACCEPT);

            System.out.println("Server start, port :" + port);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while(true){
            try {
                //1 必须要让多路复用器开始监听
                this.seletor.select();
                //2 返回多路复用器已经选择的结果集
                Iterator<SelectionKey> keys = this.seletor.selectedKeys().iterator();
                //3 进行遍历
                while(keys.hasNext()){
                    //4 获取一个选择的元素
                    SelectionKey key = keys.next();
                    //5 直接从容器中移除就可以了
                    keys.remove();
                    //6 如果是有效的
                    if(key.isValid()){
                        //7 如果为阻塞状态
                        if(key.isAcceptable()){
                            this.accept(key);
                        }
                        //8 如果为可读状态
                        if(key.isReadable()){
                            this.read(key);
                        }
                        //9 写数据
                        if(key.isWritable()){
                            //this.write(key); //ssc
                        }
                    }

                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void write(SelectionKey key){
        //ServerSocketChannel ssc =  (ServerSocketChannel) key.channel();
        //ssc.register(this.seletor, SelectionKey.OP_WRITE);
    }

    private void read(SelectionKey key) {
        try {
            //1 清空缓冲区旧的数据
            this.readBuf.clear();
            //2 获取之前注册的socket通道对象
            SocketChannel sc = (SocketChannel) key.channel();
            //3 读取数据
            int count = sc.read(this.readBuf);
            //4 如果没有数据
            if(count == -1){
                key.channel().close();
                key.cancel();
                return;
            }
            //5 有数据则进行读取 读取之前需要进行复位方法(把position 和limit进行复位)
            this.readBuf.flip();
            //6 根据缓冲区的数据长度创建相应大小的byte数组，接收缓冲区的数据
            byte[] bytes = new byte[this.readBuf.remaining()];
            //7 接收缓冲区数据
            this.readBuf.get(bytes);
            //8 打印结果
            String body = new String(bytes).trim();
            System.out.println("Server : " + body);

            // 9..可以写回给客户端数据 

        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    private void accept(SelectionKey key) {
        try {
            //1 获取服务通道
            ServerSocketChannel ssc =  (ServerSocketChannel) key.channel();
            //2 执行阻塞方法
            SocketChannel sc = ssc.accept();
            //3 设置阻塞模式
            sc.configureBlocking(false);
            //4 注册到多路复用器上，并设置读取标识
            sc.register(this.seletor, SelectionKey.OP_READ);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {

        new Thread(new Server(7788)).start();;
    }
}
```



### 适用场景

NIO方式适用于 连接数目多 且 连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。









## 异步非阻塞 AIO (NIO.2)



### 原理

服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由操作系统先完成了再通知服务器应用去启动线程进行处理。

AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用操作系统参与并发操作，编程比较复杂，JDK7开始支持。

I/O属于底层操作，需要操作系统支持，并发也需要操作系统的支持，所以性能方面不同操作系统差异会比较明显。另外NIO的非阻塞，需要一直轮询，也是一个比较耗资源的。所以出现AIO。

如果你理解了Java NIO ,下面讲的netty也是水到渠成的事,只想说,深水区已过了!

差点忘记还要补下AIO的,这个比NIO先进的技术,最终实现了netty。

这是神一样存在的java nio框架, 这个偏底层的东西,可能你接触较少却又无处不在,比如: 

![image-20200401121211515](https://tva1.sinaimg.cn/large/00831rSTgy1gde54j3nlij31gu0gwq8g.jpg)



### 适用场景

适用于连接数比较多且连接比较长（重操作）的架构，比较相册服务器，充分调用OS参与并发操作，编程比较复杂，jdk7开始支持；







## 参考

[BIO、NIO、AIO原理](https://www.cnblogs.com/barrywxx/p/8430790.html?from=singlemessage)

[Netty5 用户指南](http://ifeve.com/netty5-user-guide/)

[BIO、NIO、AIO适用场景](https://www.cnblogs.com/hujinshui/p/10368985.html)

