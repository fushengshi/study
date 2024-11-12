# Netty

Netty 基于 NIO（NIO 是⼀种同步⾮阻塞的 I/O 模型，在 Java 1.4 中引⼊了 NIO），使⽤ Netty 可以极⼤地简化 TCP 和 UDP 套接字服务器等⽹络编程，并且性能以及安全性等很多⽅⾯都⾮常优秀。

## 线程模型

Netty 采用了 Reactor 线程模型的设计，核心原理是 Selector 负责监听 I/O 事件，在监听到 I/O 事件之后，分发（Dispatch）给相关线程进行处理。

![image-20240610225325875](https://gitee.com/fushengshi/image/raw/master/image-20240610225325875.png)

**主从多线程模型**

Netty 抽象出两组线程池：BossGroup 专门用于接收客户端的连接，WorkerGroup 专门用于网络的读写。

BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup，相当于一个事件循环组，其中包含多个事件循环 ，每一个事件循环是 NioEventLoop。NioEventLoop 表示一个不断循环的、执行处理任务的线程，每个 NioEventLoop 都有一个Selector 对象与之对应，用于监听绑定在其上的连接，这些连接上的事件由 Selector 对应的这条线程处理。每个 NioEventLoopGroup 可以含有多个 NioEventLoop，也就是多个线程。

- 每个 Boss NioEventLoop 会监听 Selector 上连接建立的 `accept` 事件，然后处理 `accept` 事件与客户端建立网络连接，生成相应的 NioSocketChannel 对象，一个 NioSocketChannel 就表示一条网络连接。之后会将 NioSocketChannel 注册到某个 Worker NioEventLoop 上的 Selector 中。

- 每个 Worker NioEventLoop 会监听对应 Selector 上的 `read/write` 事件，当监听到 `read/write` 事件的时候，会通过 Pipeline 进行处理。一个 Pipeline 与一个 Channel 绑定，在 Pipeline 上可以添加多个 ChannelHandler，每个 ChannelHandler 中都可以包含一定的逻辑，例如编解码等。Pipeline 在处理请求的时候，会按照指定的顺序调用 ChannelHandler。



## TCP 粘包/拆包

TCP粘包/拆包的原因：

在网络通信的过程中，每次可以发送的数据包大小是受多种因素限制的，如 MTU 传输单元大小、MSS 最大分段大小、滑动窗口等。

- 如果一次传输的网络包数据大小超过传输单元大小，那么我们的数据可能会拆分为多个数据包发送出去。

- 如果每次请求的网络包数据都很小，一共请求了 10000 次，TCP 并不会分别发送 10000 次，因为 TCP 采用的 Nagle 算法对此作出了优化。

  > Nagle 算法
  >
  > Nagle 算法可以理解为**批量发送**，也是平时编程中经常用到的优化思路，它是在数据未得到确认之前先写入缓冲区，等待数据确认或者缓冲区积攒到一定大小再把数据包发送出去。Netty 中为了使数据传输延迟最小化，默认禁用了 Nagle 算法。

由于拆包/粘包问题的存在，数据接收方很难界定数据包的边界在哪里，很难识别出一个完整的数据包。需要提供一种机制来识别数据包的界限，也是解决拆包/粘包的唯一方法：定义应用层的通信协议。

- 消息定长

  ```
  +------+------+------+------+
  | ABCD | EFGH | IJKL | M000 |
  +------+------+------+------+
  ```

  FixedLengthFrameDecoder类。

- 包尾增加特殊字符分割

  ```
  +-------------------------+
  | AB\nCDEF\nGHIJ\nK\nLM\n |
  +-------------------------+
  ```

  行分隔符类LineBasedFrameDecoder或自定义分隔符类DelimiterBasedFrameDecoder。

- 将消息分为消息头和消息体

  ```
  +-----+-------+-------+----+-----+
  | 2AB | 4CDEF | 4GHIJ | 1K | 2LM |
  +-----+-------+-------+----+-----+
  ```

  LengthFieldBasedFrameDecoder类。分为有头部的拆包与粘包、长度字段在前且有头部的拆包与粘包、多扩展头部的拆包与粘包。



## 零拷贝

- OS 层⾯

  通过 FileRegion 包装的 `FileChannel.tranferTo()` 实现⽂件传输，可以直接将⽂件缓冲区的数据发送到⽬标 Channel，避免了传统通过循环 write ⽅式导致的内存拷⻉问题。

- Netty 层⾯

  Netty 提供的 CompositeByteBuf 类, 可以将多个 ByteBuf 合并为⼀个逻辑上的 ByteBuf , 避免了各个 ByteBuf 之间的拷⻉。

  ByteBuf ⽀持 slice 操作, 因此可以将 ByteBuf 分解为多个共享同⼀个存储区域的 ByteBuf , 避免了内存的拷⻉。

  通过 wrap 操作，可以将 byte[] 数组、ByteBuf、ByteBuffer等包装成一个 Netty ByteBuf 对象, 进而避免了拷贝操作。



## 附录

### 应用

Dubbo、RocketMQ、Elasticsearch、gRPC、Spark 等等热⻔开源项⽬都⽤到了 Netty。

⼤部分微服务框架底层涉及到⽹络通信的部分都是基于 Netty 来做的，⽐如说Spring Cloud ⽣态系统中的⽹关 Spring Cloud Gateway。



### 代码示例

#### Java NIO 示例

```java
public class MyNIOServer {
    public static void main(String[] args) throws IOException {
        // 开启服务端Socket
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(5555));
        // 设置为非阻塞
        serverSocketChannel.configureBlocking(false);
        // 开启一个选择器
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            // 阻塞等待需要处理的事件发生
            selector.select();
            // 获取selector中注册的全部事件的 SelectionKey 实例
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            // 获取已经准备完成的key
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey next = iterator.next();
                // 当发现连接事件
                if (next.isAcceptable()) {
                    // 获取客户端连接
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    // 设置非阻塞
                    socketChannel.configureBlocking(false);
                    // 将该客户端连接注册进选择器 并关注读事件
                    socketChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("accepted connection from " + socketChannel);
                    // 如果是读事件
                } else if (next.isReadable()) {
                    ByteBuffer allocate = ByteBuffer.allocate(128);
                    // 获取与此key唯一绑定的channel
                    SocketChannel channel = (SocketChannel) next.channel();
                    // 开始读取数据
                    int read = channel.read(allocate);
                    if (read > 0) {
                        System.out.println(new String(allocate.array()));
                    } else if (read == -1) {
                        System.out.println("断开连接");
                        channel.close();
                    }
                }
                // 删除这个事件
                iterator.remove();
            }
        }
    }
}
```

> `epoll_create` 对应Java NIO代码中的 `Selector.open()`。
>
> ~~~`epoll_ctl` 对应Java NIO代码中的 `socketChannel.register(selector, xxx)`。~~~
>
> `epoll_wait` 对应Java NIO代码中的 `selector.select()`。



#### Netty 示例

```java
public final class MyNettyServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        final SslContext sslContext;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslContext = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslContext = null;
        }

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline channelPipeline = ch.pipeline();
                            if (sslContext != null) {
                                channelPipeline.addLast(sslContext.newHandler(ch.alloc()));
                            }
                            channelPipeline.addLast(new MyServerHandler());
                        }
                    });

            ChannelFuture channelFuture = serverBootstrap.bind(PORT).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

public class MyServerHandler extends ChannelInboundHandlerAdapter {
    private static final DefaultEventLoopGroup eventExecutors = new DefaultEventLoopGroup(16);

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 强转为netty的ByteBuf
        ByteBuf byteBuf = (ByteBuf) msg;
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello", CharsetUtil.UTF_8));

        eventExecutors.execute(new Runnable() {
            @Override
            public void run() {
                // 将一个耗时的任务转为异步执行，提交到NioEventLoop的taskQueue中
                try {
                    Thread.sleep(10 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ctx.writeAndFlush(Unpooled.copiedBuffer("execute finish", CharsetUtil.UTF_8));
            }
        });
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```











