# 第一章 IO基础模型

## 1.Linux网络 I/O 模型

1. 阻塞IO模型

   ![img](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/Untitled.png)

2. 非阻塞IO模型

应用层到内核，如果缓存区没有数据 则直接返回错误

![image-20210108170336724](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170336724.png)

1. I/O复用模型

![image-20210108170352750](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170352750.png)

1. 信号驱动I/O模型

![image-20210108170402257](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170402257.png)

1. 异步I/O

![image-20210108170427915](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170427915.png)

## 2.IO多路复用

# 第二章 NIO入门

2.1 BIO模型

BIO服务端通讯模型，通常由一个独立的Acceptor线程负责监听客户端的连接，它接收到客户端连接请求后为**每一个客户端创建一个新的线程**进行链路处理

**缺点**：当并发量增大，系统会发现线程堆栈溢出，创建线程失败等问题

![image-20210108170453387](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170453387.png)

2.2 伪异步I/O编程

伪异步IO：将客户端的Socket封装成一个Task投递到后端的线程池中进行处理，JDK的线程池维护一个队列和N个活跃线程。

优点：资源占用是可控的，无论多少个并发，都不会造成资源的耗尽和宕机

弊端：

![image-20210108170534857](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170534857.png)

NIO：即非阻塞IO

原理：NIO 是利用了单线程轮询事件的机制，通过高效地定位就绪的 Channel，来决定做什么，仅仅 select 阶段是阻塞的，可以有效避免大量客户端连接时，频繁线程切换带来的问题，应用的扩展能力有了非常大的提高。

NIO服务端序列图：

![image-20210108170600247](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170600247.png)

NIO客户端序列图：

![image-20210108170618423](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170618423.png)

**优点**：

1、客户端发起的连接操作都是异步，可以通过在多路复用器注册OP_CONNECT等待后续结果，不需要像之前的客户端那样被同步阻塞。

2、SocketChannel 读写都是异步的，如果没有可读数据它不会异步等待，直接返回。

3、线程模型的优化。由于JDK的Selector在Linux等主流操作系统上通过epoll实现，它没有连接句柄数的限制

2.4 AIO编程

NIO 2.0引入新的异步通道，并提供了异步文件通道和异步套接字通道的实现

***CompletionHandler\*** 接口的实现类作为操作完成的回调

NIO2.0 的异步套接字通道是真正的异步非阻塞I/O

2.5 4中I/O的对比

1、异步非阻塞I/O

NIO2.0 添加了异步的套接字通道 真正实现了异步I/O

2、多路复用器Selector

Java NIO 的核心是多路复用I/O技术，多路复用的核心是通过在Selector中轮询注册在其上的Channel，当发现某个或多个Channel处于就绪状态后，从阻塞状态返回就绪的Channel的选择键集合，进行I/O操作。

3、伪异步I/O

通过在通信线程和业务线程之间做个缓冲区，这个缓冲区用于隔离I/O线程和业务间的直接访问，这样业务线程就不会被I/O线程堵塞。

![image-20210108170750243](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170750243.png)

# 第三章 Netty入门应用

3.1 Netty服务器开发

```java
/**
 * 基于netty的时间服务器
 *
 * @author 陈一锋
 * @date 2021/1/6 14:09
 **/
public class TimeServer {

    public static void main(String[] args) throws Exception {
        int port = 9094;
        new TimeServer().bind(port);
    }

    private void bind(int port) throws Exception {
        //配置服务器NIO线程组
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG,1024)
                    .childHandler(new ChildChannelHandler());
            //绑定端口
            ChannelFuture f = b.bind(port).sync();
            //等待服务器监听端口关闭
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new TimeServerHandler());
        }
    }

}

/**
 * @author 陈一锋
 * @date 2021/1/6 14:17
 **/
public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf byteBuf = (ByteBuf) msg;
        byte[] req = new byte[byteBuf.readableBytes()];
        byteBuf.readBytes(req);
        String body = new String(req, UTF_8);
        System.out.println("The time server receive order:" + body);
        String currTime = "QUERY THE ORDER".equalsIgnoreCase(body) ?
                new Date(System.currentTimeMillis()).toString() :
                "BAD ORDER";
        ByteBuf resp = Unpooled.copiedBuffer(currTime.getBytes());
        ctx.write(resp);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

3.2 客户端开发

```java
/**
 * @author 陈一锋
 * @date 2021/1/6 14:43
 **/
public class TimeClient {

    public static void main(String[] args) throws Exception {
        int port = 9094;
        new TimeClient().connect(port, "127.0.0.1");
    }

    private void connect(final int port, final String host) throws Exception {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new TimeClientHandler());
                        }
                    });
            //发起异步连接操作
            ChannelFuture f = b.connect(host, port).sync();
            //等待客户端链路关闭
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    private class TimeClientHandler extends ChannelInboundHandlerAdapter {

        public final Logger LOGGER = Logger.getLogger(TimeClientHandler.class.getName());

        private final ByteBuf firstMessages;

        private TimeClientHandler() {
            byte[] req = "QUERY THE ORDER".getBytes();
            firstMessages = Unpooled.buffer(req.length);
            firstMessages.writeBytes(req);
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
						//这里是建立TCP连接后发送消息
            ctx.writeAndFlush(firstMessages);
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ByteBuf buf = (ByteBuf) msg;
            byte[] req = new byte[buf.readableBytes()];
            buf.readBytes(req);
            String body = new String(req, UTF_8);
            System.out.println("Now is:" + body);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            LOGGER.warning("Unexpected exception:" + cause.getMessage());
            ctx.close();
        }
    }

}
```

# 第四章 TCP粘包/拆包问题的解决之道

4.1.1 TCP粘包/拆包问题说明

![image-20210108170825205](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170825205.png)

4.1.2发生的原因:

1、应用程序write写入的字节大小大于套接口发送缓存区大小。

2、进行MSS大小的TCP分段

3、以太网帧的payload大于MTU进行IP分片

![image-20210108170844894](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170844894.png)

4.1.3 粘包问题的解决策略

![image-20210108170913437](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108170913437.png)

4.3 利用LineBasedFrameDecoder解决粘包问题

服务器

```java
public static void main(String[] args) throws Exception {
        int port = 9095;
        new TimeServer().bind(port);
    }

    private void bind(int port) throws Exception {
        //配置服务器NIO线程组
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChildChannelHandler());
            //绑定端口
            ChannelFuture f = b.bind(port).sync();
            //等待服务器监听端口关闭
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            //添加这两个解码器 LineBasedFrameDecoder  StringDecoder
            ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
            ch.pipeline().addLast(new StringDecoder());
            ch.pipeline().addLast(new TimeServerHandler());
        }
    }

/**
     * 消息处理器
     */
    class TimeServerHandler extends ChannelInboundHandlerAdapter {

        private int count;

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            //这种情况会发送粘包 拆包
//            ByteBuf byteBuf = (ByteBuf) msg;
//            byte[] req = new byte[byteBuf.readableBytes()];
//            byteBuf.readBytes(req);
//            String body = new String(req, UTF_8).substring(0, req.length - System.getProperty("line.separator").length());
            //使用编码器后接收的为msg就是删除回车换行符后的请求消息
            String body = (String) msg;
            System.out.println("The time server receive order:" + body + ";the counter is :" + ++count);

            String currTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ?
                    new Date(System.currentTimeMillis()).toString() :
                    "BAD ORDER";
            currTime = currTime + System.getProperty("line.separator");
            ByteBuf resp = Unpooled.copiedBuffer(currTime.getBytes());
            ctx.writeAndFlush(resp);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            ctx.close();
        }
    }
```

客户端

```java
/**
 * @author 陈一锋
 * @date 2021/1/6 14:43
 **/
public class TimeClient {

    public static void main(String[] args) throws Exception {
        int port = 9095;
        new TimeClient().connect(port, "127.0.0.1");
    }

    private void connect(int port, String host) throws Exception {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            //这里添加解码器
                            ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new TimeClientHandler());
                        }
                    });
            //发起异步连接操作
            ChannelFuture f = b.connect(host, port).sync();
            //等待客户端链路关闭
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    private class TimeClientHandler extends ChannelInboundHandlerAdapter {

        public final Logger LOGGER = Logger.getLogger(TimeClientHandler.class.getName());

        private byte[] req;

        private int counter;

        private TimeClientHandler() {
            req = ("QUERY TIME ORDER" + System.getProperty("line.separator")).getBytes();
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ByteBuf messages = null;
            for (int i = 0; i < 100; i++) {
                messages = Unpooled.buffer(req.length);
                messages.writeBytes(req);
                ctx.writeAndFlush(messages);
            }
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            //注释的为发生 粘包拆包时的代码
//            ByteBuf buf = (ByteBuf) msg;
//            byte[] req = new byte[buf.readableBytes()];
//            buf.readBytes(req);
//            String body = new String(req, UTF_8);
            String body = (String) msg;
            System.out.println("Now is:" + body + "; the counter is :" + ++counter);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            LOGGER.warning("Unexpected exception:" + cause.getMessage());
            ctx.close();
        }
    }
}
```

LineBaseFrameDecoder工作原理时它依次遍历ByteBuf中的可读字节，判断是否有“\n” "\r\n"，如果有，就以此位置为结束位置，从可读索引到结束位置区间的字节就组成了一行。

StringDecoder功能非常简单，将接受i到的对象转化成字符串，然后调用后面的Handler。

LineBaseFrameDecoder 和SringDecoder 组合就是换行切换的文本解码器。

# 第五章 分隔符和定长解码器的应用

TCP以流的方式进行数据传输，上层应用协议为了对消息进行区分，往往采用如下4种方式。

1. 消息长度固定，累计读到长度总和为定长LEN的报文后，就认为读取到了一个完整的消息；将计数器置位，重新开始读取下一个数据报。
2. 将回车换行符作为消息结束符，例如FTP协议。
3. 将特殊的分隔符作为消息的结束标志。
4. 通过在消息头中定义长度字段来标识消息的总长度。

5.1 DelimiterBasedFrameDecoder应用开发

通过对DelimiterBasedFrameDecoder，我们可以自动完成以分隔符为码流结束标识的消息的解码。

EchoServer服务器开发

```java
public class EchoServer {
    public static final int PORT = 9096;

    public static void main(String[] args) throws Exception {
        new EchoServer().bind(PORT);
    }

    public void bind(int port) throws Exception {
        //配置线程组
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes());
                            //假如注释掉这个解码器 则会出现TCP粘包问题
                            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, delimiter));
                            //此为固定长度解码器 如果消息为固定 则可以使用此解码器
                            //ch.pipeline().addLast(new FixedLengthFrameDecoder(20));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new EchoServerHandler());
                        }
                    });
            //绑定端口
            ChannelFuture f = b.bind(port).sync();
            //等待服务器监听端口关闭
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private class EchoServerHandler extends ChannelInboundHandlerAdapter {

        int counter;

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            String body = (String) msg;
            System.out.println("This is " + ++counter + " times receives client:[" + body + "]");
            body += "$_";
            ByteBuf echo = Unpooled.copiedBuffer(body.getBytes());
            ctx.writeAndFlush(echo);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close();
        }
    }
}
```

客户端开发

```java
/**
 * @author 陈一锋
 * @date 2021/1/7 11:48
 **/
public class EchoClient {
    public static void main(String[] args) throws Exception {
        new EchoClient().connect(EchoServer.PORT, "127.0.0.1");
    }

    private void connect(int port, String host) throws Exception {
        //创建线程组
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes());
                            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, delimiter));
														//定长解码器
                            //ch.pipeline().addLast(new FixedLengthFrameDecoder(1024));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new EchoClientHandler());
                        }
                    });
            ChannelFuture f = b.connect(host, port).sync();
            //等待客户端链路关闭
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    private class EchoClientHandler extends ChannelInboundHandlerAdapter {
        private int counter;

        static final String ECHO_REQ = "Hi,cyf.Welcome to Netty.$_";

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            for (int i = 0; i < 100; i++) {
                ctx.writeAndFlush(Unpooled.copiedBuffer(ECHO_REQ.getBytes()));
            }
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("This is " + ++counter + " times receiver server:[" + msg + "]");
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            ctx.flush();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close();
        }
    }
}
```

![image-20210108171125379](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210108171125379.png)

# 第六章 编解码技术

6.1 Java序列化的缺点

1. 无法跨语言。java序列化后的字节数组，别的语言无法进行反序列化。
2. 序列化后的码流太大。
3. 序列化性能太低

java序列化对比ByteBuffer通用二进制解码技术

```java
/**
 * @author 陈一锋
 * @date 2021/1/7 14:13
 **/
public class TestUserInfo {

    public static void main(String[] args) {
        UserInfo user = new UserInfo();
        user.buildUserId(11).buildUsername("welcome to netty");
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(out);) {
            objectOutputStream.writeObject(user);
            objectOutputStream.flush();
            byte[] b = out.toByteArray();
            // jdk serialize result length is 124
            System.out.println("jdk serializable length is:" + b.length);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // byteBuffer  result is 24
        System.out.println("the byte array serializable length is:" + user.codeC().length);

    }
}

@Getter
@Setter
public class UserInfo implements Serializable {

    private String userName;
    private int userId;

    public UserInfo buildUsername(String userName) {
        this.userName = userName;
        return this;
    }

    public UserInfo buildUserId(int userId) {
        this.userId = userId;
        return this;
    }

    /**
     * 编码
     *
     * @return /
     */
    public byte[] codeC() {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        byte[] value = this.userName.getBytes();
        buffer.putInt(value.length);
        buffer.put(value);
        buffer.putInt(this.userId);
        buffer.flip();
        value = null;
        byte[] result = new byte[buffer.remaining()];
        buffer.get(result);
        return result;
    }

}
```

# 第十一章 WebSocket 协议开发

11.1

**websocket 特点：**

- 单一的TCP连接,采用全双工模式通信
- 对代理、防火墙和路由器透明
- 无头部信息、对Cookie 和身份验证
- 无安全开销
- 通过“ping/pong”帧保持链路激活
- 服务器可以主动传递消息给客户端，不再需要客户端轮询

11.2 websocket连接建立

![image-20210113100813158](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210113100813158.png)

11.3 websocket 连接关闭

![image-20210113101325425](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210113101325425.png)

11.4 websocket 服务器开发

```java
package com.cyf.nettybook.protocol;

import cn.hutool.http.HttpStatus;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.*;
import io.netty.handler.codec.http.websocketx.*;
import io.netty.handler.stream.ChunkedWriteHandler;

import java.util.Date;

import static io.netty.util.CharsetUtil.UTF_8;

/**
 * websocket 文本服务器
 *
 * @author 陈一锋
 * @date 2021/1/12 20:33
 **/
public class WebsocketServer {

    private static final int PORT = 9091;

    public static void main(String[] args) throws InterruptedException {
        new WebsocketServer().run();
    }

    private void run() throws InterruptedException {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new HttpServerCodec());
                            // 这里目的是将HTTP消息多个部分组合成一条完整的HTTP消息
                            ch.pipeline().addLast(new HttpObjectAggregator(65536));
                            // 主要用于支持浏览器和服务端进行web socket通信
                            ch.pipeline().addLast(new ChunkedWriteHandler());
                        	/*
                            说明：
                            1、对应webSocket，它的数据是以帧（frame）的形式传递
                            2、浏览器请求时 ws://localhost:3000/webSocket 表示请求的uri
                            3、核心功能是将http协议升级为ws协议，保持长连接
                            这里如果添加此协议处理器 则自定义的WebSocketServerHandler 不再需要判断是否是http请求还是websocket 只需处理文本信息或						   二进制数据
                            */
                            // ch.pipeline().addLast(new WebSocketServerProtocolHandler("ws://localhost:9091/websocket","WebSocket", 							true, 65536 * 10));
                            ch.pipeline().addLast(new WebSocketServerHandler());
                        }
                    });
            ChannelFuture future = bootstrap.bind(PORT).sync();
            System.out.println("websocket server start success with port:" + PORT);
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    private class WebSocketServerHandler extends SimpleChannelInboundHandler<Object> {

        private WebSocketServerHandshaker handshaker;

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, Object msg) {
            if (msg instanceof FullHttpRequest) {
                handleHttpRequest(ctx, (FullHttpRequest) msg);
            } else if (msg instanceof WebSocketFrame) {
                handleWebsocketFrame(ctx, (WebSocketFrame) msg);
            }
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) {
            ctx.flush();
        }

        private void handleHttpRequest(ChannelHandlerContext ctx, FullHttpRequest request) {
            // 如果HTTP解码失败,返回http异常
            if (!request.decoderResult().isSuccess()
                    || !("websocket").equals(request.headers().get("Upgrade"))) {
                sendHttpResponse(ctx, request, new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.BAD_REQUEST));
                return;
            }
            //构建握手协议 本地测试
            WebSocketServerHandshakerFactory wsFactory = new WebSocketServerHandshakerFactory("ws://localhost:9091/websocket", null, false);
            handshaker = wsFactory.newHandshaker(request);
            if (handshaker == null) {
                //不支持websocket版本
                WebSocketServerHandshakerFactory.sendUnsupportedVersionResponse(ctx.channel());
            } else {
                handshaker.handshake(ctx.channel(), request);
            }
        }

        private void handleWebsocketFrame(ChannelHandlerContext ctx, WebSocketFrame webSocketFrame) {
            // 判断是否关闭链路的指令
            if (webSocketFrame instanceof CloseWebSocketFrame) {
                handshaker.close(ctx.channel(), ((CloseWebSocketFrame) webSocketFrame).retain());
                return;
            }
            // 判断是否是ping指令
            if (webSocketFrame instanceof PingWebSocketFrame) {
                ctx.channel().write(new PongWebSocketFrame(webSocketFrame.content().retain()));
                return;
            }
            // 本案例只支持文本 不支持二进制数据
            if (!(webSocketFrame instanceof TextWebSocketFrame)) {
                throw new UnsupportedOperationException(String.format("%s frame type not support", webSocketFrame.getClass().getName()));
            }

            //返回应答消息
            String req = ((TextWebSocketFrame) webSocketFrame).text();
            System.out.println("receive messages:" + req);
            ctx.channel().write(new TextWebSocketFrame(req + " ,欢迎使用Netty Websocket服务,现在时刻:" + new Date(System.currentTimeMillis()).toString()));
        }

        private void sendHttpResponse(ChannelHandlerContext ctx, FullHttpRequest req, FullHttpResponse resp) {
            //返回应答给客户端
            if (resp.status().code() != HttpStatus.HTTP_OK) {
                ByteBuf byteBuf = Unpooled.copiedBuffer(resp.status().toString(), UTF_8);
                ctx.channel().write(byteBuf);
                byteBuf.release();

            }
            //如果非keep-Alive 关闭连接
            ChannelFuture future = ctx.channel().writeAndFlush(resp);
            if (resp.status().code() != HttpStatus.HTTP_OK | !HttpUtil.isKeepAlive(req)) {
                //关闭连接
                future.addListener(ChannelFutureListener.CLOSE);
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            ctx.close();
        }
    }


}

```



73-79 会接受到http请求或者websocket请求 为http请求则是升级协议请求



11.5 客户端实现

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    Netty WebSocket 时间服务器
</head>
<br>
<body>
<br>
<script type="text/javascript">
    var socket;
    if (!window.WebSocket)
    {
        window.WebSocket = window.MozWebSocket;
    }
    if (window.WebSocket) {
        socket = new WebSocket("ws://localhost:9091/websocket");
        socket.onmessage = function(event) {
            var ta = document.getElementById('responseText');
            ta.value="";
            ta.value = event.data
        };
        socket.onopen = function(event) {
            var ta = document.getElementById('responseText');
            ta.value = "打开WebSocket服务正常，浏览器支持WebSocket!";
        };
        socket.onclose = function(event) {
            var ta = document.getElementById('responseText');
            ta.value = "";
            ta.value = "WebSocket 关闭!";
        };
    }
    else
    {
        alert("抱歉，您的浏览器不支持WebSocket协议!");
    }

    function send(message) {
        if (!window.WebSocket) { return; }
        if (socket.readyState == WebSocket.OPEN) {
            socket.send(message);
        }
        else
        {
            alert("WebSocket连接没有建立成功!");
        }
    }
</script>
<form onsubmit="return false;">
    <input type="text" name="message" value="Netty最佳实践"/>
    <br><br>
    <input type="button" value="发送WebSocket请求消息" onclick="send(this.form.message.value)"/>
    <hr color="blue"/>
    <h3>服务端返回的应答消息</h3>
    <textarea id="responseText" style="width:500px;height:300px;"></textarea>
</form>
</body>
</html>
```



# 第十二章 私有协议栈开发

## 12.1 协议栈的功能设计

Netty协议栈承载了业务内部的各模块之间的消息交互和服务调用，主要功能有：

1. 基于Netty的NIO通信协议，提供高性能的异步通信能力
2. 提供消息编解码框架，可以实现POJO序列化及反序列
3. 提供基于Ip地址的白名单接入认证机制
4. 链路的有效性校验机制
5. 链路的断连机制

12.1.2 通信模型

![image-20210113224538175](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210113224538175.png)

具体步骤如下：

1. Netty协议栈客户端发送握手请求消息，携带节点ID等有效身份认证信息
2. Netty协议栈服务端对握手请求消息进行合法性校验，包含节点ID有效性验证，节点重复登录校验和IP地址合法性校验，校验通过后，后悔登录成功的握手应答消息。
3. 链路建议成功后，客户端发送业务消息
4. 链路成功后，服务端发送心跳消息
5. 链路建立成功后，客户端发送心跳消息
6. 链路建立成功后，服务端发送业务员消息
7. 服务器退出时，服务端关闭连接，客户端感知对方关闭连接后，被动关闭客户端连接。

12.1.3 消息定义

Netty 协议栈消息定义包含两部分：

- 消息头
- 消息体

![image-20210114193111770](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210114193111770.png)

![image-20210114193125808](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210114193125808.png)

12.1.4 消息编解码

详细见下方代码

12.1.5 链路的建立

发送握手消息定义如下：

1. 消息头type字段为3
2. 可选附件为个数为0
3. 消息体为空
4. 握手消息长度为22个字节

服务端收到客户端握手请求消息后，如果ip校验通过 返回握手成功应答给客户端，握手应答消息定义如下：

1. 消息头type为4
2. 可选附件个数为0
3. 消息体为byte类型结果，0 表示认证成功， -1 表示认证失败

12.1.6 链路关闭

在一下情况发生，客户端和服务器需要关闭连接

1. 当对方宕机或者重启时，会主动关闭链接，另一方读取到操作系统的通知信息，得知对方REST链路，需要关闭连接，释放自身的句柄等资源。由于采用TCP全双工通信，通信双方都需要关闭连接，释放资源。
2. 消息读取过程钟发生I/O异常 需要主动关闭连接
3. 心跳消息读写过程钟发生I/O异常 需要主动关闭连接
4. 心跳超时 需要主动关闭连接
5. 发生编码异常等不可恢复错误时，需要主动关闭连接

12.1.7 可靠性设计

在运行中，可能因为网络波动、超时等情况发生，需要进行一些可靠性的设计：

1、**心跳设计**

1. 当网络处于空闲时间超过一段设定时间T，客户端主动发送Ping心跳消息给服务端
2. 如果下一个周期T到来客户端还没收到对方发来的Pong心跳应答消息或者业务消息 则心跳失败计数器+1
3. 重新收到心跳消息或业务消息 将心跳失败计时器清空
4. 服务端网络空闲持续达到T，服务端将心跳失败计数器+1，只要收到客户端消息 计数器清零
5. 服务器连续N次没有收到客户端的消息 则关闭链路 释放资源

2、**失败重连机制**

链路断开后，由客户端发起重连操作，如果失败 则间隔T个时间后再次重连 直至成功

3、**重复登录保护**

当客户端握手成功后，链路处于正常状态下，不允许客户端重复登录，以防止客户端在异常状态下反复重连导致句柄资源被耗尽。

4、**消息缓存重发**

无论客户端还是服务器，当发生链路中断后，在链路恢复之前，缓存在消息队列中待发送的消息不能丢失 待链路恢复之后 重新发送这些消息。

## 12.2 Netty协议栈开发

### 12.2.1 数据结构定义

> NettyMessage类结构

```java
/**
 * @author 陈一锋
 * @date 2021/1/14 19:58
 **/
@Data
public final class NettyMessage {
    /**
     * 消息头
     */
    private Header header;
    /**
     * 消息体
     */
    private Object body;
}

```

>  Header设计

```java
/**
 * 消息头
 *
 * @author 陈一锋
 * @date 2021/1/14 19:59
 **/
@Data
public final class Header {
    private int crcCode = 0xabef0101;
    /**
     * 消息长度
     */
    private int length;
    /**
     * 会话ID
     */
    private long sessionId;
    /**
     * 消息类型
     */
    private byte type;
    /**
     * 消息优先级
     */
    private byte priority;
    /**
     * 附件
     */
    private Map<String, Object> attachment = new HashMap<>();
}

```

### 12.2.2 消息编解码

**消息解码类**

主要是继承`MessageToByteEncoder`按照自定义的消息格式编码输出

```java
/**
 * 消息编码器
 *
 * @author 陈一锋
 * @date 2021/1/14 20:07
 **/
public final class NettyMessageEncoder extends MessageToByteEncoder<NettyMessage> {

    final private MarshallingEncoder marshallingEncoder;

    public NettyMessageEncoder() throws IOException {
        this.marshallingEncoder = new MarshallingEncoder();
    }

    @Override
    protected void encode(ChannelHandlerContext ctx, NettyMessage msg, ByteBuf out) throws Exception {
        if (msg == null || msg.getHeader() == null) {
            throw new Exception("The encode message is null");
        }

        Header header = msg.getHeader();
        out.writeInt(header.getCrcCode());
        out.writeInt(header.getLength());
        out.writeLong(header.getSessionId());
        out.writeByte(header.getType());
        out.writeByte(header.getPriority());
        out.writeInt(header.getAttachment().size());

        for (Map.Entry<String, Object> entry : header.getAttachment().entrySet()) {
            String key = entry.getKey();
            Object value = entry.getValue();
            out.writeInt(key.length());
            out.writeBytes(key.getBytes("UTF-8"));
            marshallingEncoder.encode(out, value);
        }

        if (msg.getBody() != null) {
            marshallingEncoder.encode(out, msg.getBody());
        } else {
            out.writeInt(0);
        }
        //设置总长度
        out.setInt(4, out.readableBytes() - 8);
    }

}
```

**消息编码器**

继承`LengthFieldBasedFrameDecoder`，这个是netty提供的长度解码器，可以通过自定义长度的方式实现解码，避免半包的情况

```java
/**
 * 解码器
 *
 * @author 陈一锋
 * @date 2021/1/14 21:34
 **/
public class NettyMessageDecoder extends LengthFieldBasedFrameDecoder {

    private MarshallingDecoder marshallingDecoder;

    public NettyMessageDecoder(int maxFrameLength, int lengthFieldOffset, int lengthFieldLength) throws IOException {
        super(maxFrameLength, lengthFieldOffset, lengthFieldLength);
        marshallingDecoder = new MarshallingDecoder();
    }

    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = (ByteBuf) super.decode(ctx, in);
        if (frame == null) {
            return null;
        }

        Header header = new Header();
        header.setCrcCode(frame.readInt());
        header.setLength(frame.readInt());
        header.setSessionId(frame.readLong());
        header.setType(frame.readByte());
        header.setPriority(frame.readByte());

        int size = frame.readInt();
        if (size > 0) {
            Map<String, Object> att = new HashMap<>(size);
            for (int i = 0; i < size; i++) {
                int keyLength = frame.readInt();
                byte[] keyBuf = new byte[keyLength];
                frame.readBytes(keyBuf);
                String key = new String(keyBuf, StandardCharsets.UTF_8);
                att.put(key, marshallingDecoder.decode(frame));
            }
            header.setAttachment(att);
        }
        NettyMessage message = new NettyMessage();
        message.setHeader(header);
        if (frame.readableBytes() > 4) {
            message.setBody(marshallingDecoder.decode(frame));
        }
        return message;
    }
}
```

### 12.2.3 握手和安全认证

在TCP链路建立成功后，握手消息的接入和安全认证在服务器进行处理

主要实现的逻辑在建立链接后发送消息，在接受的消息的时候判断是否握手成功

如果认证失败 则关闭链接

> LoginAuthReqHandler

```java
/**
 * 握手请求处理器
 *
 * @author 陈一锋
 * @date 2021/1/15 21:02
 **/
public class LoginAuthReqHandler extends ChannelInboundHandlerAdapter {


    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        NettyMessage message = buildReq();
        System.out.println("建立连接,发送握手请求数据:" + message.toString());
        ctx.writeAndFlush(message);
    }

    /**
     * 如果是握手应答消息
     * 判断是否认证成功
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        NettyMessage message = (NettyMessage) msg;
        if (message.getHeader() != null
                && message.getHeader().getType() == MessageType.LOGIN_RESP.getType()) {
            byte body = (byte) message.getBody();
            if (body != 0) {
                //握手失败,关闭连接
                ctx.close();
            } else {
                System.out.println("login is success:" + message);
                ctx.fireChannelRead(msg);
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.fireExceptionCaught(cause);
    }

    private NettyMessage buildReq() {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(MessageType.LOGIN_RES.getType());
        message.setHeader(header);
        return message;
    }
}
```



**服务端的握手接入主要是**：

接受到握手消息判断是否是重复连接以及在IP白名单上

如果有则验证成功

```java
/**
 * @author 陈一锋
 * @date 2021/1/16 16:22
 **/
public class LoginAuthResHandler extends ChannelInboundHandlerAdapter {

    private final Map<String, Boolean> nodeCache = new ConcurrentHashMap<>();

    /**
     * 白名单 简单的使用 正式使用可以读取配置文件
     */
    private static final List<String> WHITE_LIST;

    static {
        WHITE_LIST = new ArrayList<>();
        WHITE_LIST.add("127.0.0.1");
        WHITE_LIST.add("47.107,53,172");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof NettyMessage) {
            NettyMessage message = (NettyMessage) msg;
            if (message.getHeader() != null &&
                    message.getHeader().getType() == MessageType.LOGIN_RES.getType()) {
                // 握手请求
                String nodeIndex = ctx.channel().remoteAddress().toString();
                if (nodeCache.containsKey(nodeIndex)) {
                    //重复登录
                    System.out.println("This address has login:" + nodeIndex);
                    ctx.writeAndFlush(buildRes((byte) -1));
                } else {
                    //验证是否在白名单
                    InetSocketAddress inetSocketAddress = (InetSocketAddress) ctx.channel().remoteAddress();
                    String ip = inetSocketAddress.getAddress().getHostAddress();
                    if (WHITE_LIST.contains(ip)) {
                        // 回复成功消息
                        System.out.println("the address " + ip + " login successful");
                        ctx.writeAndFlush(buildRes((byte) 0));
                        nodeCache.put(nodeIndex, true);
                    } else {
                        ctx.writeAndFlush(buildRes((byte) -1));
                    }
                }
            } else {
                ctx.fireChannelRead(msg);
            }
        }
    }

    private NettyMessage buildRes(byte result) {
        Header header = new Header();
        header.setType(MessageType.LOGIN_RESP.getType());
        NettyMessage message = new NettyMessage();
        message.setHeader(header);
        message.setBody(result);
        return message;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        nodeCache.clear();
        super.exceptionCaught(ctx, cause);
    }
}
```

### 12.2.4 心跳检测机制

握手成功，客户端主动发送心跳信息

服务端收到心跳消息之后，返回心跳应答消息

如果握手成功消息，则启动无限循环定时器用于定时发送心跳消息

> 客户端心跳请求

```java
/**
 * 心跳请求处理
 *
 * @author 陈一锋
 * @date 2021/1/17 11:56
 **/
public class HeartBeatReqHandler extends ChannelInboundHandlerAdapter {

    private volatile ScheduledFuture<?> heartBeat;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof NettyMessage) {
            NettyMessage message = (NettyMessage) msg;
            if (message.getHeader() != null &&
                    message.getHeader().getType() == MessageType.LOGIN_RESP.getType()) {
                //登录验证通过发送心跳消息
                heartBeat = ctx.executor().scheduleAtFixedRate(new HeartBeatTask(ctx), 0, 5000, TimeUnit.MILLISECONDS);
            } else if (message.getHeader() != null &&
                    message.getHeader().getType() == MessageType.HEART_RESP.getType()) {
                // 收到心跳回复消息
                System.out.println("client receive server heart msg" + message);
            } else {
                ctx.fireChannelRead(msg);
            }
        }
    }

    private class HeartBeatTask implements Runnable {
        private final ChannelHandlerContext ctx;

        public HeartBeatTask(final ChannelHandlerContext ctx) {
            this.ctx = ctx;
        }

        @Override
        public void run() {
            NettyMessage message = buildHeartMsg();
            System.out.println("client send heart messages to server:" + message);
            ctx.writeAndFlush(message);
        }
    }

    private NettyMessage buildHeartMsg() {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(MessageType.HEART_RES.getType());
        message.setHeader(header);
        return message;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        if (heartBeat != null) {
            heartBeat.cancel(true);
            heartBeat = null;
        }
        super.exceptionCaught(ctx, cause);
    }
}
```



**服务器心跳**

```java
/**
 * 心跳回复请求
 *
 * @author 陈一锋
 * @date 2021/1/17 12:07
 **/
public class HeartBeatResHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof NettyMessage) {
            NettyMessage message = (NettyMessage) msg;
            if (message.getHeader() != null
                    && message.getHeader().getType() == MessageType.HEART_RES.getType()) {
                //返回心跳消息
                System.out.println("server receive client heart msg" + message);
                NettyMessage heartMsg = buildHeartMsg();
                System.out.println("server send msg to client:" + message);
                ctx.writeAndFlush(heartMsg);
            } else {
                ctx.fireChannelRead(msg);
            }
        }
    }

    private NettyMessage buildHeartMsg() {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(MessageType.HEART_RESP.getType());
        message.setHeader(header);
        return message;
    }
}
```

### 12.2.5 断线重连

客户端感知断连后，释放资源重写连接

```java
finally {
            //所有资源释放完毕后 清空资源 重新连接
            executor.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(5);
                    //发起重新连接
                    connect(NettyConstant.PORT, NettyConstant.IP);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
```

### 12.2.6 客户端代码

```java
/**
 * 客户端
 *
 * @author 陈一锋
 * @date 2021/1/17 12:19
 **/
public class NettyClient {
    private ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
    private NioEventLoopGroup group = new NioEventLoopGroup();

    public void connect(int port, String host) throws InterruptedException {
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new NettyMessageDecoder(1024 * 1024, 4, 4));
                            pipeline.addLast(new NettyMessageEncoder());
                            //当30秒没有读到消息关闭连接
                            pipeline.addLast(new ReadTimeoutHandler(50));
                            pipeline.addLast(new LoginAuthReqHandler());
                            pipeline.addLast(new HeartBeatReqHandler());

                        }
                    });
            ChannelFuture future = bootstrap.connect(new InetSocketAddress(NettyConstant.IP, NettyConstant.PORT)).sync();
            System.out.println("connect success,port:" + port);
            future.channel().closeFuture().sync();
        } finally {
            //所有资源释放完毕后 清空资源 重新连接
            executor.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(5);
                    //发起重新连接
                    connect(NettyConstant.PORT, NettyConstant.IP);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new NettyClient().connect(NettyConstant.PORT, NettyConstant.IP);
    }
}
```

### 12.2.7 服务端代码

```java
/**
 * @author 陈一锋
 * @date 2021/1/17 12:38
 **/
public class NettyServer {
    public void bind() throws InterruptedException {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new NettyMessageDecoder(1024 * 1024, 4, 4));
                            pipeline.addLast(new NettyMessageEncoder());
                            //当30秒没有读到消息关闭连接
                            pipeline.addLast(new ReadTimeoutHandler(50));
                            pipeline.addLast(new LoginAuthResHandler());
                            pipeline.addLast(new HeartBeatResHandler());
                        }
                    });
            ChannelFuture future = serverBootstrap.bind(NettyConstant.IP, NettyConstant.PORT).sync();
            System.out.println("server start success,port:" + NettyConstant.PORT);
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new NettyServer().bind();
    }
}
```



# 第十三章 服务端创建

![image-20210117221843514](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210117221843514.png)

Netty服务端创建的关键和原理解析。

**步骤1：** 创建ServerBoostrap实例。ServerBoostrap是Netty服务端的启动辅助类。因为需要设置多个参数，设计上采用了Builder模式。

**步骤2：**设置并绑定Reactor线程池。Netty的Reactor线程池是EventLoopGroup，它实际是个EventLoop数组。EventLoop的职责是处理所有注册到本线程的多路复选器Selector上的Channel，Selector的轮询操作由绑定的EventLoop线程run方法驱动。

**步骤3：**设置并绑定服务端Channel。作为NIO服务器，需要创建ServerSocketChannel，Netty对原生的NIO类进行封装，对应的实现类为NioServerSocketChannel。

**步骤4：**链路建立的时候创建并初始化ChannelPipeline。ChannelPipeline并不是NIO服务器必须的，本质是一个负责处理网络事件的职责链。典型的网络事件有：

1. 链路注册
2. 链路激活
3. 链路断开
4. 接收到请求消息
5. 请求消息接收并处理完毕
6. 发送应答消息
7. 链路发生异常
8. 发生用户自定义事件

**步骤5**：初始化ChannelPipeline完成之后，添加并设置ChildHandler。ChannelHandler是Netty提供用户扩展和定制的关键接口。比较实用的系统ChannleHandler总结如下：

1. 系统编解码框架 ByteToMessageCodec
2. 通用基于长度的半包解码器 LengthFieldBasedFrameDecoder
3. 码流日志打印Handler LoggingHandler
4. 链路空闲检测Handler IdleStateHandler
5. Base64 编解码 Base64Decoder 和 Base64Encoder

添加代码：

```java
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChildChannelHandler());xxxxxxxxxx5 1                    .childHandler(new ChildChannelHandler());            ServerBootstrap b = new ServerBootstrap();2            b.group(bossGroup, workerGroup)3                    .channel(NioServerSocketChannel.class)4                    .option(ChannelOption.SO_BACKLOG, 1024)5                    .childHandler(new ChildChannelHandler());java
```

**步骤6：**绑定并启动监听端口。在完成一系列初始化操作后会启动端口监听。并将ServerSockerChannel注册到Selector上监听客户端连接。

```java
            //绑定端口
            ChannelFuture f = b.bind(port).sync();
            //等待服务器监听端口关闭
            f.channel().closeFuture().sync();
```

**步骤7：**Selector轮询。



