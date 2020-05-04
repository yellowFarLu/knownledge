# Netty入门示例



## 背景

使用Netty实现时间服务器，作为Netty入门示例。

功能：客户端向服务端请求当前的时间



## 服务端代码

```java
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
```

```java
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

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        // ByteBuf类似NIO中ByteBuffer，不过它提供了更加强大和灵活的能力
        ByteBuf buf = (ByteBuf)msg;

        // readableBytes获取缓冲区可读的字节数
        byte[] req = new byte[buf.readableBytes()];

        // 将缓冲区中的字节数组赋值到req中
        buf.readBytes(req);

        String body = new String(req, StandardCharsets.UTF_8);
        System.out.println("server receive req=" + body);
        String currentTimeStr = new Date().toString();
        ByteBuf response = Unpooled.copiedBuffer(currentTimeStr.getBytes(StandardCharsets.UTF_8));

        // 异步发送应答消息给客户端
        ctx.write(response);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        /*
         * flush方法作用是将发送缓冲区中的消息写入到SocketChannel发送给对方。
         * 从性能的角度考虑，为了防止频繁的唤醒Selector进行消息的发送，Netty的write方法
         * 并不直接将消息写入SocketChannel，调用write方法仅将消息放入到发送缓冲区中，
         * 再通过调用flush方法，将发送缓冲区中的消息全部写到SocketChannel。
         */
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```





## 客户端代码

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
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
```

```java
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.nio.charset.StandardCharsets;

/**
 * @author huangy on 2020-05-04
 */
public class TimeClientHandler extends ChannelHandlerAdapter {

    private final ByteBuf firstMessage;

    public TimeClientHandler() {
        byte[] req = "Client query time".getBytes(StandardCharsets.UTF_8);
        this.firstMessage = Unpooled.buffer(req.length);
        firstMessage.writeBytes(req);
    }

    /**
     * 客户端和服务端建立TCP链路成功以后，会回调channelActive
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(firstMessage);
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
        System.out.println("get time is " + body);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println(cause.getMessage());
        ctx.close();
    }
}
```



