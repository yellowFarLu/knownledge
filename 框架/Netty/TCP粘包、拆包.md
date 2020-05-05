# TCP粘包、拆包





## 粘包、拆包问题说明

TCP是个流协议，所谓“流”，就是没有界限的一串数据。大家可以想象河里面的河水，是连成一片的，没有界限。

TCP底层并不了解上层业务数据的具体含义，它会根据TCP缓冲区的情况进行包的划分，在业务上认为一个完整的包可能被TCP拆成多个小包进行发送；也可能把多个小包封装成一个大数据包进行发送，这就是所谓的TCP粘包、拆包问题。

我们可以通过图解对TCP粘包、拆分问题进行说明：

![image-20200504121645177](https://tva1.sinaimg.cn/large/007S8ZIlgy1gegapg6vb6j30vs0ga0xz.jpg)

假设客户端分别发送了2个数据包D1和D2给服务端，由于服务端一次读取到的字节数是不确定的，故可能存在以下4种情况：

- 服务端分两次读取到了两个独立的数据包，分别为D1和D2，没有粘包和拆包。
- 服务端一次接收到了两个数据包，即D1和D2粘合在一起，被称为TCP粘包。
- 服务端分两次读取到了两个数据包，第一次读取到了完整的数据包D1和D2的部分内容，第二次读取到了D2包的剩余内容，这被称为TCP拆包。
- 服务端分两次读取到了两个数据包，第一次读取到了D1部分内容，第二次读取到了D1剩余内容和D2数据包。

如果服务端TCP接收窗口非常小，而数据包D1和D2非常大，很可能会发生第5种可能，即服务端分多次才能接收完D1和D2，期间发生多次拆包。







## 粘包、拆包发生的原因

- 应用程序写入的字节大小大于发送缓冲区大小
- 进行MSS大小的TCP分段
- 以太网帧的payload大于MTU，进行IP分片
  - payload指的是数据包的原始数据，参考https://blog.csdn.net/Neo233/article/details/80032774

![image-20200504122439925](https://tva1.sinaimg.cn/large/007S8ZIlgy1gegaxols32j30zu0nqjy3.jpg)





## 解决思想

由于底层TCP无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的。这个问题只能通过上层的应用协议栈设计来解决。方法归纳如下：

- 消息定长：例如每个报文的大小固定长200字节，如果不够，则后面补偿空格。
  - 这个很好理解，如果消息太长了会拆包，太短了会粘包，那么就规定一个不长不短的长度，就可以避免拆包、粘包的问题了
- 在包尾增加回车换行符进行分割，例如FTP协议
- 将消息分为消息头和消息体，消息头中包含消息总长度的字段
- 更复杂的应用层协议





## 粘包代码示例

```java
// 服务端
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

/**
 * @author huangy on 2020-05-04
 */
public class TimeServer {

    public void bind(int port) throws Exception {
        /*
         * 配置服务端的NIO线程组
         * NioEventLoopGroup是个线程组，它包含了一组NIO线程，专门用于网络事件的处理。实际上它们就是Reactor线程组。
         * 这里创建2个线程组的原因是：
         * （1）一个用于服务端接收客户端的连接（bossGroup）
         * （2）另一个用于监听SocketChannel的网络读写事件（workerGroup）
         */
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            /*
             * ServerBootstrap是用于启动NIO服务端的辅助启动类，目的是降低服务端开发的复杂度
             */
            ServerBootstrap b = new ServerBootstrap();

            b.group(bossGroup, workerGroup)
                    // 绑定NIO管道
                    .channel(NioServerSocketChannel.class)
                    // 设置TCP参数，这里是设置请求队列的长度，元素个数超过该长度则拒绝请求
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    // IO事件的处理类，类似于Reactor模式中的handler类，主要用于处理网络IO事件
                    .childHandler(new ChildChannelHandler());

            // 绑定端口，同步等待成功
            ChannelFuture f = b.bind(port).sync();

            // 等待服务端链路关闭后，main函数才退出
            f.channel().closeFuture().sync();

        } finally {
            // 优雅退出，释放服务器资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {
        @Override
        protected void initChannel(SocketChannel socketChannel) throws Exception {
            socketChannel.pipeline().addLast(new TimeServerHandler());
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeServer().bind(8080);
    }
}



import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.nio.charset.StandardCharsets;
import java.util.Date;

/**
 * TimeServerHandler负责对网络事件进行读写操作
 * @author huangy on 2020-05-04
 */
public class TimeServerHandler extends ChannelHandlerAdapter {

    private int counter;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        // ByteBuf类似NIO中ByteBuffer，不过它提供了更加强大和灵活的能力
        ByteBuf buf = (ByteBuf)msg;

        // readableBytes获取缓冲区可读的字节数
        byte[] req = new byte[buf.readableBytes()];

        // 将缓冲区中的字节数组赋值到req中
        buf.readBytes(req);

        String body = new String(req, StandardCharsets.UTF_8)
                .substring(0, req.length - System.getProperty("line.separator").length());
        System.out.println("server receive req=" + body + "    counter=" + ++counter);

        String currentTimeStr;
        if ("QUERY TIME ORDER".equalsIgnoreCase(body)) {
            currentTimeStr = new Date().toString();
        } else {
            currentTimeStr = "BAD ORDER";
        }

        ByteBuf response = Unpooled.copiedBuffer(currentTimeStr.getBytes(StandardCharsets.UTF_8));

        // 异步发送应答消息给客户端
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
// 客户端
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

/**
 * @author huangy on 2020-05-04
 */
public class TimeClient {

    public void connet(int port, String host) throws Exception {
        // 配置客户端NIO线程组
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel channel) throws Exception {
                            channel.pipeline().addLast(new TimeClientHandler());
                        }
                    });

            // 发起异步连接事件
            ChannelFuture f = b.connect(host, port).sync();

            // 等待客户端链路关闭
            f.channel().closeFuture().sync();

        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeClient().connet(8080, "127.0.0.1");
    }
}


import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.nio.charset.StandardCharsets;

/**
 * @author huangy on 2020-05-04
 */
public class TimeClientHandler extends ChannelHandlerAdapter {

    private int counter;

    private byte[] req;

    public TimeClientHandler() {
        req = ("QUERY TIME ORDER" + System.getProperty("line.separator")).
                getBytes(StandardCharsets.UTF_8);
    }

    /**
     * 客户端和服务端建立TCP链路成功以后，会回调channelActive
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ByteBuf message = null;

        for (int i = 0; i < 100; i++) {
            message = Unpooled.buffer(req.length);
            message.writeBytes(req);

            ctx.writeAndFlush(message);
        }
    }

    /**
     * 服务端返回应答的时候，调用channelRead
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf)msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, StandardCharsets.UTF_8);
        System.out.println("get time is " + body + "    counter=" + ++counter);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println(cause.getMessage());
        ctx.close();
    }
}
```

```java
// 服务端输出
server receive req=QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUE    counter=1
server receive req=Y TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER
QUERY TIME ORDER    counter=2
  

// 客户端输出
get time is BAD ORDERBAD ORDER    counter=1  
```







## Netty解决方法

利用解码器，开发者不用考虑粘包、拆分问题。



### 解决代码示例

Netty使用LineBasedFrameDecoder和StringDecoder来解决粘包问题。

```java
// 服务端
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;

/**
 * @author huangy on 2020-05-04
 */
public class TimeServer {

    public void bind(int port) throws Exception {
        /*
         * 配置服务端的NIO线程组
         * NioEventLoopGroup是个线程组，它包含了一组NIO线程，专门用于网络事件的处理。实际上它们就是Reactor线程组。
         * 这里创建2个线程组的原因是：
         * （1）一个用于服务端接收客户端的连接（bossGroup）
         * （2）另一个用于监听SocketChannel的网络读写事件（workerGroup）
         */
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            /*
             * ServerBootstrap是用于启动NIO服务端的辅助启动类，目的是降低服务端开发的复杂度
             */
            ServerBootstrap b = new ServerBootstrap();

            b.group(bossGroup, workerGroup)
                    // 绑定NIO管道
                    .channel(NioServerSocketChannel.class)
                    // 设置TCP参数，这里是设置请求队列的长度，元素个数超过该长度则拒绝请求
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    // IO事件的处理类，类似于Reactor模式中的handler类，主要用于处理网络IO事件
                    .childHandler(new ChildChannelHandler());

            // 绑定端口，同步等待成功
            ChannelFuture f = b.bind(port).sync();

            // 等待服务端链路关闭后，main函数才退出
            f.channel().closeFuture().sync();

        } finally {
            // 优雅退出，释放服务器资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {
        @Override
        protected void initChannel(SocketChannel socketChannel) throws Exception {
            socketChannel.pipeline().addLast(new LineBasedFrameDecoder(1024));
            socketChannel.pipeline().addLast(new StringDecoder());
            socketChannel.pipeline().addLast(new TimeServerHandler());
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeServer().bind(8080);
    }
}

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.nio.charset.StandardCharsets;
import java.util.Date;

/**
 * TimeServerHandler负责对网络事件进行读写操作
 * @author huangy on 2020-05-04
 */
public class TimeServerHandler extends ChannelHandlerAdapter {

    private int counter;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        String body = (String)msg;

        System.out.println("server receive req=" + body + "    counter=" + ++counter);

        String currentTimeStr;
        if ("QUERY TIME ORDER".equalsIgnoreCase(body)) {
            currentTimeStr = new Date().toString();
        } else {
            currentTimeStr = "BAD ORDER";
        }

        currentTimeStr = currentTimeStr + System.getProperty("line.separator");

        ByteBuf response = Unpooled.copiedBuffer(currentTimeStr.getBytes(StandardCharsets.UTF_8));

        // 异步发送应答消息给客户端
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
// 客户端
package netty.fix_zhanbao;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;

/**
 * @author huangy on 2020-05-04
 */
public class TimeClient {

    public void connet(int port, String host) throws Exception {
        // 配置客户端NIO线程组
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel channel) throws Exception {
                            channel.pipeline().addLast(new LineBasedFrameDecoder(1024));
                            channel.pipeline().addLast(new StringDecoder());
                            channel.pipeline().addLast(new TimeClientHandler());
                        }
                    });

            // 发起异步连接事件
            ChannelFuture f = b.connect(host, port).sync();

            // 等待客户端链路关闭
            f.channel().closeFuture().sync();

        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeClient().connet(8080, "127.0.0.1");
    }
}

package netty.fix_zhanbao;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.nio.charset.StandardCharsets;

/**
 * @author huangy on 2020-05-04
 */
public class TimeClientHandler extends ChannelHandlerAdapter {

    private int counter;

    private byte[] req;

    public TimeClientHandler() {
        req = ("QUERY TIME ORDER" + System.getProperty("line.separator")).
                getBytes(StandardCharsets.UTF_8);
    }

    /**
     * 客户端和服务端建立TCP链路成功以后，会回调channelActive
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ByteBuf message = null;

        for (int i = 0; i < 100; i++) {
            message = Unpooled.buffer(req.length);
            message.writeBytes(req);

            ctx.writeAndFlush(message);
        }
    }

    /**
     * 服务端返回应答的时候，调用channelRead
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String body = (String)msg;
        System.out.println("get time is " + body + "    counter=" + ++counter);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println(cause.getMessage());
        ctx.close();
    }
}
```





### LineBasedFrameDecoder和StringDecoder原理分析

LineBasedFrameDecoder的工作原理是它一次遍历ByteBuf中的可读字节，判断看是否有“\n”或者“\r\n”， 如果有，就以此位置为结束位置，从可读索引到结束位置区间的字节就组成了一行。它是以换行符为结束标志的解码器，支持携带结束符或者不携带结束符两种解码方式，同时支持配置单行的最大长度。如果连续读取到最大长度后仍然没有出现换行符，就会抛出异常，同时忽略掉之前读到的异常码流。

StringDecoder的功能非常简单，就是将接收到的对象转换成字符串，然后继续调用后面的Handler。LineBasedFrameDecoder+StringDecoder组合就是按行切换的文本解码器，它被设计用来支持TCP的粘包和拆包。

当然，如果发送的消息不是以换行符结束的，该怎么办呢？或者没有回车换行符，靠消息头中的长度字段来分包怎么办？是不是需要自己写半包解码器？

答案是否定的，Netty提供了多种支持TCP粘包/拆包的解码器用来满足用户的不同诉求。





## 其他编码器

- DelimiterBasedFrameDecoder 分隔符解码器
  - 可以自动对采用分隔符作为码流结束符的消息进行解码。
  - 如果读取到指定长度还没有分隔符，则抛出异常。防止异常码流缺失结束符而导致的内存溢出，这是Netty编码器的可靠性保护。

- FixedLengthFrameDecoder 固定长度解码器
  - 能够按照指定消息对长度进行解码
  - 如果是很大的包，则按照给定长度进行分包
  - 如果是半包消息，则FixedLengthFrameDecoder会缓存半包消息并且等待下个包达到以后进行拼包，直到读取到一个完整的包。