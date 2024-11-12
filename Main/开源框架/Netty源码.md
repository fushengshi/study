# Netty 源码

![image-20240804193941732](https://gitee.com/fushengshi/image/raw/master/image-20240804193941732.png)

Netty 的整体功能模块结构：

- Core 核心层

  Core 核心层提供了底层网络通信的通用抽象和实现，包括可扩展的事件模型、通用的通信 API、支持零拷贝的 ByteBuf 等。

- Protocol Support 协议支持层

  协议支持层基本上覆盖了主流协议的编解码实现，如 HTTP、SSL、Protobuf、压缩、大文件传输、WebSocket、文本、二进制等主流协议，此外 Netty 还支持自定义应用层协议。Netty 丰富的协议支持降低了用户的开发成本，基于 Netty 可以快速开发 HTTP、WebSocket 等服务。

- Transport Service 传输服务层

  传输服务层提供了网络传输能力的定义和实现方法。它支持 Socket、HTTP 隧道、虚拟机管道等传输方式。Netty 对 TCP、UDP 等数据传输做了抽象和封装，用户可以更聚焦在业务逻辑实现上，而不必关系底层数据传输的细节。
  
  ![image-20240804194007664](https://gitee.com/fushengshi/image/raw/master/image-20240804194007664.png)

Netty 的逻辑处理架构为典型网络分层架构设计，共分为网络通信层、事件调度层、服务编排层。

- 网络通信层

  网络通信层的职责是执行网络 I/O 的操作。它支持多种网络协议和 I/O 模型的连接操作。当网络数据读取到内核缓冲区后，会触发各种网络事件，这些网络事件会分发给事件调度层进行处理。

  网络通信层的核心组件包含 BootStrap、ServerBootStrap、Channel 三个组件。

  - BootStrap

    Bootstrap 主要负责整个 Netty 程序的启动、初始化、服务器连接等过程，相当于一条主线串联了 Netty 的其他核心组件。

    Bootstrap 用于客户端引导，可用于连接远端服务器，只绑定一个 EventLoopGroup。

  - ServerBootStrap

    ServerBootStrap 用于服务端引导，用于服务端启动绑定本地端口，会绑定两个 EventLoopGroup。

  - Channel

    Channel 是网络通信的载体，提供了基本的 API 用于网络 I/O 操作，如 register、bind、connect、read、write、flush 等。

    Netty 自己实现的 Channel 是以 JDK NIO Channel 为基础的，相比较于 JDK NIO，Netty 的 Channel 提供了更高层次的抽象，屏蔽了底层 Socket 的复杂性，赋予了更强的功能。

- 事件调度层

  事件调度层的职责是通过 Reactor 线程模型对各类事件进行聚合处理，通过 Selector 主循环线程集成多种事件（I/O 事件、信号事件、定时事件等），实际的业务处理逻辑是交由服务编排层中相关的 Handler 完成。

  事件调度层的核心组件包括 EventLoopGroup、EventLoop。

  - EventLoopGroup

    EventLoopGroup 本质是一个线程池，包含一个或者多个 EventLoop。

  - EventLoop

    EventLoop 用于处理 Channel 生命周期内的所有 I/O 事件，如 accept、connect、read、write 等 I/O 事件。

    EventLoop 同一时间会与一个线程绑定，每个 EventLoop 负责处理多个 Channel。

    每新建一个 Channel，EventLoopGroup 会选择一个 EventLoop 与其绑定，Channel 在生命周期内可以对 EventLoop 进行多次绑定和解绑。

- 服务编排层

  服务编排层的职责是负责组装各类服务，它是 Netty 的核心处理链，用以实现网络事件的动态编排和有序传播。

  服务编排层的核心组件包括 ChannelPipeline、ChannelHandler、ChannelHandlerContext。

  - ChannelPipeline

    ChannelPipeline 是 Netty 的核心编排组件，负责组装各种 ChannelHandler，实际数据的编解码以及加工处理操作都是由 ChannelHandler 完成的。

    ChannelPipeline 可以理解为ChannelHandler 的实例列表，内部通过双向链表将不同的 ChannelHandler 链接在一起。

  - ChannelHandler

    数据的编解码工作以及其他转换工作实际都是通过 ChannelHandler 处理的。

    每创建一个 Channel 都会绑定一个新的 ChannelPipeline，ChannelPipeline 中每加入一个 ChannelHandler 都会绑定一个 ChannelHandlerContext。

  - ChannelHandlerContext 

    ChannelHandlerContext 用于保存 ChannelHandler 上下文，保存 ChannelPipeline 和 ChannelHandler 的关联关系。

    ChannelHandlerContext 可以实现 ChannelHandler 之间的交互，ChannelHandlerContext 包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等。



## 服务端启动

![image-20240804201840041](https://gitee.com/fushengshi/image/raw/master/image-20240804201840041.png)

```java
ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
```

- 创建服务端 Channel：本质是创建 JDK 底层原生的 Channel，并初始化几个重要的属性，包括 id、unsafe、pipeline 等。
- 初始化服务端 Channel：设置 Socket 参数以及用户自定义属性，并添加两个特殊的处理器 ChannelInitializer 和 ServerBootstrapAcceptor。
- 注册服务端 Channel：调用 JDK 底层将 Channel 注册到 Selector 上。
- 端口绑定：调用 JDK 底层进行端口绑定，并触发 channelActive 事件，把 OP_ACCEPT 事件注册到 Channel 的事件集合中。

ServerBootstrap 中 `bind()` 方法进行服务端启动。

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 初始化并注册 Channel
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        // initAndRegister() 的过程如果发生异常则直接返回。
        return regFuture;
    }

    if (regFuture.isDone()) {
        // initAndRegister() 如行完毕则调用 doBind0() 进行 Socket 绑定。
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // initAndRegister() 没有执行结束，添加一个回调监听，执行结束后通过 doBind0() 进行端口绑定。
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

`initAndRegister()` 可以分为三步：创建 Channel、初始化 Channel 和注册 Channel。

- 创建 Channel

  ReflectiveChannelFactory 通过反射创建 NioServerSocketChannel 实例。

  ```java
  public NioServerSocketChannel() {
      // 创建JDK的 java.nio.channels.ServerSocketChannel
      this(newSocket(DEFAULT_SELECTOR_PROVIDER)); 
  }
  
  public NioServerSocketChannel(ServerSocketChannel channel) {
      super(null, channel, SelectionKey.OP_ACCEPT);
      config = new NioServerSocketChannelConfig(this, javaChannel().socket());
  }
  
  protected AbstractChannel(Channel parent) {
      this.parent = parent;
      id = newId(); // Channel全局唯一id 
      unsafe = newUnsafe(); // unsafe操作底层读写
      pipeline = newChannelPipeline(); // pipeline负责业务处理器编排
  }
  ```

- 初始化 Channel

  ```java
  @Override
  void init(Channel channel) {
      ChannelPipeline p = channel.pipeline();
  	...
  
      // 添加特殊的 Handler 处理器
      p.addLast(new ChannelInitializer<Channel>() {
          @Override
          public void initChannel(final Channel ch) {
              final ChannelPipeline pipeline = ch.pipeline();
              ChannelHandler handler = config.handler();
              if (handler != null) {
                  // 添加 ServerSocketChannel 对应的 Handler
                  pipeline.addLast(handler);
              }
  
              // 异步方式向 Pipeline 添加一个处理器ServerBootstrapAcceptor，这个处理器专门接受新请求
              ch.eventLoop().execute(new Runnable() {
                  @Override
                  public void run() {
                      pipeline.addLast(new ServerBootstrapAcceptor(ch, currentChildGroup,
                      	currentChildHandler, currentChildOptions, currentChildAttrs, extensions));
                  }
              });
          }
      });
  }
  ```

  在初始化时，还没有将 Channel 注册到 Selector 对象上，所以还无法注册 Accept 事件到 Selector 上，所以事先添加了 ChannelInitializer 处理器，等待 Channel 注册完成后，再向 Pipeline 中添加 ServerBootstrapAcceptor 处理器。

  ChannelInitializer 的 `initChannel()` 方法会在 **注册 Cannel 步骤** `invokeHandlerAddedIfNeeded()` 调用，因为添加 ServerBootstrapAcceptor 是一个异步过程，需要 EventLoop 线程负责执行。而当前 EventLoop 线程正在执行 `register0()` 的注册流程，所以等到 `register0()` 执行完之后才能被添加到 Pipeline 当中。

- 注册 Channel

  Netty 会在线程池 EventLoopGroup 中选择一个 EventLoop 与当前 Channel 进行绑定，之后 Channel 生命周期内的所有 I/O 事件都由这个 EventLoop 负责处理，如 accept、connect、read、write 等 I/O 事件。

  ```java
  private void register0(ChannelPromise promise) {
      try {
          if (!promise.setUncancellable() || !ensureOpen(promise)) {
              return;
          }
          boolean firstRegistration = neverRegistered;
  
          // 调用JDK底层的 register() 进行注册
          doRegister();
  
          neverRegistered = false;
          registered = true;
  		// 触发 handlerAdded 事件
          // 执行 ChannelInitializer 的 initChannel()
          pipeline.invokeHandlerAddedIfNeeded();
  
          safeSetSuccess(promise);
  		// 触发 channelRegistered 事件
          pipeline.fireChannelRegistered();
  
          // 此时 Channel 还未注册绑定地址，所以处于非活跃状态
          if (isActive()) {
              if (firstRegistration) {
                  // Channel 当前状态为活跃时，触发 channelActive 事件
                  pipeline.fireChannelActive();
              } else if (config().isAutoRead()) {
                  beginRead();
              }
          }
      } catch (Throwable t) {
          closeForcibly();
          closeFuture.setClosed();
          safeSetFailure(promise, t);
      }
  }
  
  @Override
  protected void doRegister() throws Exception {
      boolean selected = false;
      for (;;) {
          try {
              // 调用JDK底层向 Selector 进行 Channel 注册
              // 第二个参数传入的是0，表示此时服务端 Channel对没有注册任何事件
              // 注册 OP_ACCEPT 事件在服务端 Channel 监听端口之后触发 channelActive 的 doBeginRead() 方法
              selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
              return;
          } catch (CancelledKeyException e) {
              if (!selected) {
                  eventLoop().selectNow();
                  selected = true;
              } else {
                  throw e;
              }
          }
      }
  }
  ```

整个服务端 Channel 注册的流程完成，ServerBootstrap 的 `bind()` 方法进行端口绑定 `doBind0()` 。

```java
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    ...

    boolean wasActive = isActive();
    try {
        // 调用JDK底层进行端口绑定
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                // 触发 channelActive 事件
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
}
```

完成端口绑定之后，Channel 处于活跃 Active 状态，然后会调用 `pipeline.fireChannelActive()` 方法触发 channelActive 事件。

```java
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();
    readIfIsAutoRead();
}

protected void doBeginRead() throws Exception {
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;
    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp); // 注册 OP_ACCEPT 事件到服务端 Channel 的事件集合
    }
}
```



## Reactor 线程模型

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240804201918636.png" alt="image-20240804201918636" style="zoom:80%;" />

Netty 事件处理流程：

- Netty 服务端启动后，BossEventLoopGroup 会负责监听客户端的 Accept 事件。

- 有客户端新连接接入时

  - BossEventLoopGroup 中的 NioEventLoop 首先会新建客户端 Channel。

  - 在 NioServerSocketChannel 中触发 channelRead 事件传播。

    NioServerSocketChannel 中包含了一种特殊的处理器 ServerBootstrapAcceptor。

  - 最终通过 ServerBootstrapAcceptor 的 `channelRead()` 方法将新建的客户端 Channel 分配到 WorkerEventLoopGroup 中。

    WorkerEventLoopGroup 中包含多个 NioEventLoop，它会选择其中一个 NioEventLoop 与新建的客户端 Channel 绑定。

- 完成客户端连接注册之后，接收客户端的请求数据。

  当客户端向服务端发送数据时，NioEventLoop 会监听到 OP_READ 事件，然后分配 ByteBuf 并读取数据，读取完成后将数据传递给 Pipeline 进行处理。

  一般来说，数据会从 ChannelPipeline 的第一个 ChannelHandler 开始传播，将加工处理后的消息传递给下一个 ChannelHandler，整个过程是串行化执行。

NioEventLoop 的核心入口 `run()` 方法。

```java
@Override
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                // 判断是否有需要处理的 Channel，如果没有则返回 SelectStrategy.SELECT
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.BUSY_WAIT:
                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE;
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            // 执行selector.select
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                    default:
                     	// 如果存在就绪的 I/O 事件直接跳出，然后执行 I/O 事件处理和异步任务队列处理的逻辑
                }
            } catch (IOException e) {
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            // 如果 ioRatio == 100，则说明优先处理所有的 IO 任务，处理完所有的IO事件后才会处理所有的 Task 任务。
            if (ioRatio == 100) {
                try {
                    // I/O 事件处理
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // 异步任务队列处理
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                ranTasks = runAllTasks(0);
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("",);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // 解决JDK epoll 空轮询 Bug
                selectCnt = 0;
            }
        } catch () {
        } finally {
        }
    }
}
```

NioEventLoop 的 `run()` 方法是一个无限循环，没有任何退出条件，在不间断循环执行以下三件事情：

- 轮询 I/O 事件（select）：轮询 Selector 选择器中已经注册的所有 Channel 的 I/O 事件。

  `select()` 轮询注册的 I/O 事件，当没有 I/O 事件产生时，为了避免 NioEventLoop 线程一直循环空转，在获取 I/O 事件或者异步任务时需要阻塞线程，等待 I/O 事件就绪或者异步任务产生后才唤醒线程。

  >  Netty 在外部线程添加任务的时候，可以唤醒 select 阻塞操作。

- 处理 I/O 事件（processSelectedKeys）：处理已经准备就绪的 I/O 事件。

  AbstractNioChannel 的处理场景，通过获取 SelectionKey 上挂载的 attachment 判断 SelectionKey 的类型，不同的 SelectionKey 的类型又会调用不同的处理方法，然后通过 Pipeline 进行事件传播。

  ```java
  private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
      // 首先获取 Channel 的 NioUnsafe，所有的读写等操作都在 Channel 的 unsafe 类中操作。
      final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
  
      try {
          int readyOps = k.readyOps();
          // 如果是 OP_CONNECT 说明已经连接成功，并把注册的 OP_CONNECT 事件取消
          if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
              int ops = k.interestOps();
              ops &= ~SelectionKey.OP_CONNECT;
              k.interestOps(ops);
  
              unsafe.finishConnect();
          }
  
  		// 如果是 OP_WRITE 事件，说明可以继续向 Channel 中写入数据，当写完数据后用户自己把 OP_WRITE 事件取消。
          if ((readyOps & SelectionKey.OP_WRITE) != 0) {
             unsafe.forceFlush();
          }
  
          // 如果是 OP_READ 或 OP_ACCEPT 事件，则调用 unsafe.read() 进行读取数据。
          // unsafe.read() 中会调用到 ChannelPipeline 进行读取数据。
          if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
              unsafe.read();
          }
      } catch (CancelledKeyException ignored) {
          unsafe.close(unsafe.voidPromise());
      }
  }
  ```

  - OP_CONNECT 连接建立事件

    表示 TCP 连接建立成功，Channel 处于 Active 状态。处理 OP_CONNECT 事件首先将该事件从事件集合中清除，避免事件集合中一直存在连接建立事件，然后调用 `unsafe.finishConnect()` 方法通知上层连接已经建立。

    底层调用 `pipeline().fireChannelActive()` 方法，这时会产生一个 Inbound 事件在 Pipeline 中进行传播，依次调用 ChannelHandler 的 `channelActive()` 方法，通知各个 ChannelHandler 连接建立成功。

  - OP_WRITE 可写事件

    表示上层可以向 Channel 写入数据，通过执行 `ch.unsafe().forceFlush()` 操作，将数据冲刷到客户端，最终会调用 javaChannel 的 `write()` 方法执行底层写操作。

  - OP_READ 可读事件

    表示 Channel 收到了可以被读取的新数据。Netty 将 READ 和 Accept 事件进行了统一的封装，都通过 `unsafe.read()` 进行处理。

    `unsafe.read()` 的逻辑可以归纳为几个步骤：

    1. 从 Channel 中读取数据并存储到分配的 ByteBuf。
    2. 调用 `pipeline.fireChannelRead()` 方法产生 Inbound 事件，然后依次调用 ChannelHandler 的 `channelRead()` 方法处理数据。
    3. 调用 `pipeline.fireChannelReadComplete()` 方法完成读操作。
    4. 最终执行 `removeReadOp()` 清除 OP_READ 事件。

- 处理异步任务队列（runAllTasks）：处理任务队列中的非 I/O 任务。Netty 提供了 ioRatio 参数用于调整 I/O 事件处理和任务处理的时间比例。

  异步任务处理 `runAllTasks()` 的过程可以分为三步：合并定时任务到普通任务队列，然后从普通任务队列中取出任务并处理，最后进行收尾工作。

  

## ChannelPipeline 

![image-20240804202054645](https://gitee.com/fushengshi/image/raw/master/image-20240804202054645.png)

> 责任链模式。
>
> 构成一个由 ChannelHandlerContext 对象组成的双向链表，通过链表的指针移动（ctx = ctx.next）获取链表中的其它节点。
>
> Pipeline 维护链表的头尾节点。

- Pipeline 初始化

  ChannelPipeline 是在创建 Channel 时被创建的，它是 Channel 中非常重要的一个成员变量。

  ```java
  protected AbstractChannel(Channel parent) {
      this.parent = parent;
      id = newId(); // Channel全局唯一id 
      unsafe = newUnsafe(); // unsafe操作底层读写
      pipeline = newChannelPipeline(); // pipeline负责业务处理器编排
  }
  
  protected DefaultChannelPipeline(Channel channel) {
      this.channel = ObjectUtil.checkNotNull(channel, "channel");
      succeededFuture = new SucceededChannelFuture(channel, null);
      voidPromise =  new VoidChannelPromise(channel, true);
  
      tail = new TailContext(this);
      head = new HeadContext(this);
      head.next = tail;
      tail.prev = head;
  }
  ```

  当 ChannelPipeline 初始化完成后，会构成一个由 ChannelHandlerContext 对象组成的双向链表，默认 ChannelPipeline 初始化状态的最小结构仅包含 HeadContext 和 TailContext 两个节点。HeadContext 和 TailContext 属于 ChannelPipeline 中两个特殊的节点，它们都继承自 AbstractChannelHandlerContext

- Pipeline 添加 Handler

  在 Netty 客户端或者服务端启动时，就需要用户配置自定义实现的业务处理器。

  ```java
  bootstrap.group(bossGroup, workerGroup)
      .channel(NettyEventLoopFactory.serverSocketChannelClass())
      .childHandler(new ChannelInitializer<SocketChannel>() {
          @Override
          protected void initChannel(SocketChannel ch) throws Exception {
              ch.pipeline()
                  .addLast("decoder", adapter.getDecoder()) // ChannelInboundHandlerAdapter
                  .addLast("encoder", adapter.getEncoder()) // ChannelOutboundHandlerAdapter
                  .addLast("server-idle-handler", 
                           new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS)) // ChannelDuplexHandler
                  .addLast("handler", nettyServerHandler); // ChannelDuplexHandler
          }
      });
  ```
  ChannelPipeline 分为入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器，它们都会被 ChannelHandlerContext 封装，最终都是通过双向链表连接。

  ```java
  @Override
  public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
      final AbstractChannelHandlerContext newCtx;
      // 防止多线程并发操作 ChannelPipeline 底层双向链表。
      synchronized (this) {
          // 检查是否重复添加 Handler
          checkMultiplicity(handler);
  
          // 创建新的 DefaultChannelHandlerContext 节点
          newCtx = newContext(group, filterName(name, handler), handler);
  
          // 添加新的 DefaultChannelHandlerContext 节点到 ChannelPipeline
          addLast0(newCtx);
  		...
      }
      // 回调用户方法
      callHandlerAdded0(newCtx);
      return this;
  }
  
  // 双向链表的尾部插入新的节点
  private void addLast0(AbstractChannelHandlerContext newCtx) {
      AbstractChannelHandlerContext prev = tail.prev;
      newCtx.prev = prev;
      newCtx.next = tail;
  
      prev.next = newCtx;
      tail.prev = newCtx;
  }
  ```

- 数据在 Pipeline 中的运转

  ChannelPipeline 分为入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器。Inbound 事件和 Outbound 事件的传播方向相反，Inbound 事件的传播方向为 Head -> Tail，而 Outbound 事件传播方向是 Tail -> Head。

  以 Inbound 事件的传播为例：

  Netty Reactor 线程模型，NioEventLoop 会不断轮询 OP_ACCEPT 和 OP_READ 事件，当事件就绪时，NioEventLoop 会及时响应。NioEventLoop 中源码的入口 `processSelectedKey()`。

  ```java
  if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
      unsafe.read();
  }
  ```

  Netty 会不断从 Channel 中读取数据到分配的 ByteBuf 中，然后通过 `pipeline.fireChannelRead()` 方法触发 ChannelRead 事件的传播。Netty 首先会以 Head 节点为入参，直接调用一个静态方法 `invokeChannelRead()`。

  - 如果当前是在 Reactor 线程内部，会直接执行 `next.invokeChannelRead()` 方法。
  
- 如果是外部线程发起的调用，Netty 会把 `next.invokeChannelRead()` 调用封装成异步任务提交到任务队列（可以保证执行流程全部控制在当前 NioEventLoop 线程内部串行化执行，确保线程安全性）。
  
    > 详见 Netty 任务机制。
  
  ```java
  public final ChannelPipeline fireChannelRead(Object msg) {
      AbstractChannelHandlerContext.invokeChannelRead(head, msg);
      return this;
  }
  
  static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
      final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
      EventExecutor executor = next.executor();
      
      // Reactor 线程内部调用
      if (executor.inEventLoop()) {
          next.invokeChannelRead(m);
      } else {
          // SingleThreadEventExecutor
          executor.execute(new Runnable() { // 外部线程调用
              @Override
              public void run() {
                  next.invokeChannelRead(m);
              }
        });
      }
  }
  ```
  
  当前 ChannelHandlerContext 节点会取出自身对应的 Handler，执行 Handler 的 channelRead 方法。Inbound 事件是从 HeadContext 节点开始进行传播的，当前节点是 headContext，直接通过 `fireChannelRead()` 方法继续将读事件继续传播下去。
  
  ```java
  private void invokeChannelRead(Object msg) {
      if (invokeHandler()) {
          try {
              final ChannelHandler handler = handler();
              final DefaultChannelPipeline.HeadContext headContext = pipeline.head;
              if (handler == headContext) {
                  // 当前为头节点
                  headContext.channelRead(this, msg);
              } else if (handler instanceof ChannelDuplexHandler) {
                  ((ChannelDuplexHandler) handler).channelRead(this, msg);
              } else {
                  // 对于自定义handler，执行复写的channelRead（继承ChannelInboundHandlerAdapter复写channelRead）
                  ((ChannelInboundHandler) handler).channelRead(this, msg);
              }
          } catch (Throwable t) {
              invokeExceptionCaught(t);
          }
      } else {
          fireChannelRead(msg);
      }
  }
  
  // HeadContext
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) {
      ctx.fireChannelRead(msg);
  }
  
  @Override
  public ChannelHandlerContext fireChannelRead(final Object msg) {
      // 找到下一个节点，执行 invokeChannelRead
    invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
      return this;
  }
  ```

  Inbound 事件递归调用的流程结束，有以下两种情况：

  1. 用户自定义的 Handler 没有执行 `fireChannelRead()` 操作，则在当前 Handler 终止 Inbound 事件传播。
  
     用户自定义 Handler 中应该手动调用 `fireChannelRead()`，保证 Pipeline 流程执行。
  
     ```java
     public class DemoHandler extends ChannelInboundHandlerAdapter {
         private static final DefaultEventLoopGroup eventExecutors = new DefaultEventLoopGroup(16);
     
         @Override
         public void channelRead(ChannelHandlerContext ctx, Object msg) {
             ByteBuf byteBuf = (ByteBuf) msg;
             System.out.println("客户端发送的消息是：" + byteBuf.toString(CharsetUtil.UTF_8));
             ctx.writeAndFlush(Unpooled.copiedBuffer("hello", CharsetUtil.UTF_8));
     
             // 任务提交到 NioEventLoop 的 taskQueue 中
             // 也可用 ctx.channel().eventLoop().execute()
             eventExecutors.execute(() -> {
                 try {
                     Thread.sleep(10 * 1000);
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 }
                 ctx.writeAndFlush(Unpooled.copiedBuffer("task finish", CharsetUtil.UTF_8));
             });
     
             // 手动调用 fireChannelRead()，否则不会进入下一个handler
           ctx.fireChannelRead(msg);
         }
     }
     ```
  
  2. 如果用户自定义的 Handler 都执行了 `fireChannelRead()` 操作，Inbound 事件传播最终会在 TailContext 节点终止。



# 附录

## Dubbo NettyServer

```java
@Override
protected void doOpen() throws Throwable {
    // 创建ServerBootstrap
    bootstrap = new ServerBootstrap();
    // 创建boss EventLoopGroup
    bossGroup = NettyEventLoopFactory.eventLoopGroup(1, "NettyServerBoss");
    // 创建worker EventLoopGroup
    workerGroup = NettyEventLoopFactory.eventLoopGroup(
        getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
        "NettyServerWorker");

    // 创建NettyServerHandler，Netty中的ChannelHandler实现，
    final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);

    // 获取当前NettyServer创建的所有Channel，这里的channels集合中的 Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
    channels = nettyServerHandler.getChannels();

    // 初始化ServerBootstrap，指定boss和worker EventLoopGroup
    bootstrap.group(bossGroup, workerGroup)
        .channel(NettyEventLoopFactory.serverSocketChannelClass())
        .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
        .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
        .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                // 连接空闲超时时间
                int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                // NettyCodecAdapter中会创建Decoder和Encoder
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    ch.pipeline().addLast("negotiation", 
                    	SslHandlerInitializer.sslServerHandler(getUrl(), nettyServerHandler));
                }
                ch.pipeline()
                    // 注册Decoder和Encoder
                    .addLast("decoder", adapter.getDecoder()) // ChannelInboundHandlerAdapter
                    .addLast("encoder", adapter.getEncoder()) // ChannelOutboundHandlerAdapter
                    // 注册IdleStateHandler
                    .addLast("server-idle-handler", 
                             new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS)) // ChannelDuplexHandler
                    // 注册NettyServerHandler
                    .addLast("handler", nettyServerHandler); // ChannelDuplexHandler
            }
        });

    // 绑定指定的地址和端口
    ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    channelFuture.syncUninterruptibly(); // 等待bind操作完成
    channel = channelFuture.channel();
}
```



## Reactor 模式

- 单线程模式

  Reactor 单线程模型所有 I/O 操作都由一个线程完成，启动一个 EventLoopGroup 即可。

  ```java
  EventLoopGroup group = new NioEventLoopGroup(1);
  ServerBootstrap bootstrap = new ServerBootstrap();
  bootstrap.group(group)
  ```

- 多线程模式

  Reactor 单线程模型有严重的性能瓶颈，因此 Reactor 多线程模型出现了。在 Netty 中使用 Reactor 多线程模型与单线程模型非常相似，区别是 NioEventLoopGroup 可以自己手动设置固定的线程数（默认 2 倍 CPU 核数的线程）。

  ```java
  EventLoopGroup group = new NioEventLoopGroup();
  ServerBootstrap bootstrap = new ServerBootstrap();
  bootstrap.group(group)
  ```

- 主从多线程模式

  在大多数场景下，采用主从多线程 Reactor 模型。使用不同的 NioEventLoopGroup，主 Reactor 负责处理 Accept，然后把 Channel 注册到从 Reactor 上，从 Reactor 主要负责 Channel 生命周期内的所有 I/O 事件。

  ```java
  EventLoopGroup bossGroup = new NioEventLoopGroup();
  EventLoopGroup workerGroup = new NioEventLoopGroup();
  
  ServerBootstrap bootstrap = new ServerBootstrap();
  bootstrap.group(bossGroup, workerGroup)
  ```



## Netty 任务机制

Netty 能够保证 Channel 的操作都是线程安全的归功于 Netty 的任务机制。

NioEventLoop 内部有两个异步任务队列，分别为普通任务队列和定时任务队列。NioEventLoop 提供了 `execute()` 和 `schedule()` 方法用于向不同的队列中添加任务，`execute()` 用于添加普通任务，`schedule()` 方法用于添加定时任务。

例如，在 RPC 业务线程池里处理完业务请求后，可以根据用户请求拿到关联的 Channel，将数据写回客户端。

> 因为多个外部线程可能会并发操作同一个 Channel，这时候 Mpsc Queue 就可以保证线程的安全性。

```java
abstract class AbstractChannelHandlerContext implements ChannelHandlerContext, ResourceLeakHint {
    private void write(Object msg, boolean flush, ChannelPromise promise) {
        ...

        // ctx = ctx.prev;
        final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
        final Object m = pipeline.touch(msg, next);
        EventExecutor executor = next.executor();

        // Reactor 线程内部调用
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else { // 外部线程调用
            final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
            if (!safeExecute(executor, task, promise, m, !flush)) {
                task.cancel();
            }
        }
    }
}
```

- 如果是 Reactor 线程发起调用 `channel.write()` 方法，`inEventLoop()` 返回 true，此时直接在 Reactor 线程内部直接交由 Pipeline 进行事件处理。

- 如果是外部线程调用，会将写操作封装成一个 WriteTask，然后通过 `safeExecute()` 执行，`safeExecute()` 就是调用的 SingleThreadEventExecutor 的 `execute()` 方法，最终**将任务添加到 taskQueue 中**。

  ```java
  public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {
      @Override
      public boolean inEventLoop() {
          return inEventLoop(Thread.currentThread());
      }
  
      @Override
      public boolean inEventLoop(Thread thread) {
          return thread == this.thread;
      }
  
      private void execute(Runnable task, boolean immediate) {
          boolean inEventLoop = inEventLoop();
          // 加入任务队列
          addTask(task);
          if (!inEventLoop) {
              startThread();
              if (isShutdown()) {
                  boolean reject = false;
                  try {
                      if (removeTask(task)) {
                          reject = true;
                      }
                  } catch (UnsupportedOperationException e) {
                  }
                  if (reject) {
                      reject();
                  }
              }
          }
          if (!addTaskWakesUp && immediate) {
              wakeup(inEventLoop);
          }
      }
  
  }
  ```

添加到 taskQueue 的任务将在 NioEventLoop 的 `runAllTasks()` 中执行。



## JDK epoll 空轮询 Bug

因为 poll 和 epoll 对于突然中断的连接 socket 会对返回的 eventSet 事件集合置为 EPOLLHUP 或者 EPOLLERR，eventSet 事件集合发生了变化导致 Selector 被唤醒，如果仅仅因为这个原因唤醒且没有感兴趣的事件发生，就会变成空轮询。

Netty 引入了计数变量 selectCnt，用于记录 select 操作的次数，如果事件轮询时间小于 timeoutMillis，并且在该时间周期内连续发生超过 SELECTOR_AUTO_REBUILD_THRESHOLD（默认512）次空轮询，说明可能触发了 epoll 空轮询 Bug。Netty 通过重建新的 Selector 对象，将异常的 Selector 中所有的 SelectionKey 会重新注册到新建的 Selector，重建完成之后异常的 Selector 废弃。



## FastThreadLocal

- 高效查找

  FastThreadLocal 在定位数据的时候可以直接根据数组下标 index 获取，时间复杂度 O(1)。而 JDK 原生的 ThreadLocal 在数据较多时哈希表很容易发生 Hash 冲突，线性探测法在解决 Hash 冲突时需要不停地向下寻找，效率较低。

  FastThreadLocal 相比 ThreadLocal 数据扩容更加简单高效，FastThreadLocal 以 index 为基准向上取整到 2 的次幂作为扩容后容量，然后把原数据拷贝到新数组。而 ThreadLocal 由于采用的哈希表，所以在扩容后需要再做一轮 rehash。

- 安全性更高

  JDK 原生的 ThreadLocal 使用不当可能造成内存泄漏，只能等待线程销毁。在使用线程池的场景下，ThreadLocal 只能通过主动检测的方式防止内存泄漏，从而造成了一定的开销。

  然而 FastThreadLocal 不仅提供了 `remove()` 主动清除对象的方法，而且在线程池场景中 Netty 还封装了 FastThreadLocalRunnable，FastThreadLocalRunnable 最后会执行 `FastThreadLocal.removeAll()` 将 Set 集合中所有 FastThreadLocal 对象都清理掉，

> Thread 中维护一个 ThreadLocalMap，记录 ThreadLocal 与实例之间的映射关系。
>
> ```java
> public class Thread implements Runnable {
>        ThreadLocal.ThreadLocalMap threadLocals = null;
>        ...
> }
> 
> static class ThreadLocalMap {
>        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
>            table = new Entry[INITIAL_CAPACITY];
>            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
>            table[i] = new Entry(firstKey, firstValue);
>            size = 1;
>            setThreshold(INITIAL_CAPACITY);
>        }
>    
>        static class Entry extends WeakReference<ThreadLocal<?>> {
>            Object value;
> 
>            Entry(ThreadLocal<?> k, Object v) {
>                super(k);
>                value = v;
>            }
>        }
> }
> ```

FastThreadLocal 的实现与 ThreadLocal 非常类似，Netty 为 FastThreadLocal 量身打造了 FastThreadLocalThread 和 InternalThreadLocalMap 两个重要的类。

```java
public class FastThreadLocalThread extends Thread {
    private InternalThreadLocalMap threadLocalMap;
    ...
}

public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
    private InternalThreadLocalMap() {
        super(newIndexedVariableTable());
    }
}

class UnpaddedInternalThreadLocalMap {
    static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap 
        = new ThreadLocal<InternalThreadLocalMap>();
    
    static final AtomicInteger nextIndex = new AtomicInteger();
    Object[] indexedVariables;

    UnpaddedInternalThreadLocalMap(Object[] indexedVariables) {
        this.indexedVariables = indexedVariables;
    }
    // 省略其他代码
}
```

从 InternalThreadLocalMap 内部实现来看，与 ThreadLocalMap 一样都是采用数组的存储方式，但是 InternalThreadLocalMap 并没有使用线性探测法来解决 Hash 冲突，而是在 FastThreadLocal 初始化的时候分配一个数组索引 index（index为FastThreadLocal变量，index 的值采用原子类 AtomicInteger 保证顺序递增，通过调用 `InternalThreadLocalMap.nextVariableIndex()` 方法获得）。

在读写数据的时候通过数组下标 index 直接定位到 FastThreadLocal 的位置，时间复杂度为 O(1)。如果数组下标递增到非常大，那么数组也会比较大，所以 FastThreadLocal 是通过空间换时间的思想提升读写性能。

```java
public class FastThreadLocal<V> {
    private final int index;
    
    // FastThreadLocal.get() 的源码实现
    public final V get() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        // 直接通过FastThreadLocal分配的index获取
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }
        return initialize(threadLocalMap);
    }
}

// InternalThreadLocalMap中的indexedVariables中O(1)获取
public Object indexedVariable(int index) {
    Object[] lookup = this.indexedVariables;
    return index < lookup.length ? lookup[index] : UNSET;
}
```



## 无锁队列 Mpsc Queue

Mpsc 的全称是 Multi Producer Single Consumer，多生产者单消费者。Mpsc Queue 可以保证多个生产者同时访问队列是线程安全的，而且同一时刻只允许一个消费者从队列中读取数据。

Netty Reactor 线程中任务队列 taskQueue 必须满足多个生产者可以同时提交任务，所以 JCTools 提供的 Mpsc Queue 非常适合 Netty Reactor 线程模型。



## 时间轮 HashedWheelTimer

HashedWheelTimer 实现了接口 io.netty.util.Timer。

> 原理同 Dubbo HashedWheelTimer。



## 自定义通信协议

在 TCP 网络编程中，发送方和接收方的数据包格式都是二进制，发送方将对象转化成二进制流发送给接收方，接收方获得二进制数据后需要知道如何解析成对象，所以协议是**双方能够正常通信的基础**。

```lua
+---------------------------------------------------------------+
| 魔数 2byte | 协议版本号 1byte | 序列化算法 1byte | 报文类型 1byte  |
+---------------------------------------------------------------+
| 状态 1byte |        保留字段 4byte     |      数据长度 4byte     | 
+---------------------------------------------------------------+
|                   数据内容 （长度不定）                          |
+---------------------------------------------------------------+
```

- MessageToByteEncoder

  对象编码成字节流，MessageToByteEncoder 重写了 ChanneOutboundHandler 的 `write()` 方法，不需要关注拆包/粘包问题。

  ```java
  public class StringToByteEncoder extends MessageToByteEncoder<String> {
      @Override
      protected void encode(ChannelHandlerContext channelHandlerContext, String data, ByteBuf byteBuf) throws Exception {
          byteBuf.writeBytes(data.getBytes());
      }
  }
  ```

- ByteToMessageDecoder / ReplayingDecoder 将字节流解码为消息对象。

  解码器需要考虑拆包/粘包问题。由于接收方有可能没有接收到完整的消息，所以解码框架需要对入站的数据做缓冲操作，直至获取到完整的消息。
  
  - FixedLengthFrameDecoder
  
    消息定长
  
    ```
    +------+------+------+------+
    | ABCD | EFGH | IJKL | M000 |
    +------+------+------+------+
    ```
  
  - 行分隔符类LineBasedFrameDecoder或自定义分隔符类DelimiterBasedFrameDecoder
  
    包尾增加特殊字符分割
  
    ```
    +-------------------------+
    | AB\nCDEF\nGHIJ\nK\nLM\n |
    +-------------------------+
    ```
  
  - LengthFieldBasedFrameDecoder
  
    将消息分为消息头和消息体，分为有头部的拆包与粘包、长度字段在前且有头部的拆包与粘包、多扩展头部的拆包与粘包。
  
    ```
    +-----+-------+-------+----+-----+
    | 2AB | 4CDEF | 4GHIJ | 1K | 2LM |
    +-----+-------+-------+----+-----+
    ```
  



## Netty 的设计模式

- 单例模式

  SelectStrategy 对象的默认实现就是使用的饿汉式单例。MqttEncoder、ReadTimeoutException 等也是饿汉方式实现单例。

  ```java
  final class DefaultSelectStrategy implements SelectStrategy {
      static final SelectStrategy INSTANCE = new DefaultSelectStrategy();
      private DefaultSelectStrategy() { }
  
      @Override
      public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
          return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
      }
  }
  ```

- 工厂模式

  Netty 在创建 Channel 的时候使用的就是工厂方法模式。Netty 将反射和工厂方法模式结合在一起，只使用一个工厂类，然后根据传入的 Class 参数来构建出对应的 Channel，不需要再为每一种 Channel 类型创建一个工厂类。

  ```java
  public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
      private final Constructor<? extends T> constructor;
  
      public ReflectiveChannelFactory(Class<? extends T> clazz) {
          ObjectUtil.checkNotNull(clazz, "clazz");
          try {
              this.constructor = clazz.getConstructor();
          } catch (NoSuchMethodException e) {
              throw new IllegalArgumentException("");
          }
      }
  
      @Override
      public T newChannel() {
          try {
              return constructor.newInstance();
          } catch (Throwable t) {
              throw new ChannelException("");
          }
      }
  }
  ```

- 责任链模式

  ChannlPipeline 内部是由一组 ChannelHandler 实例组成的，内部通过双向链表将不同的 ChannelHandler 链接在一起。

- 观察者模式

  ChannelFuture 的 addListener 接口就是观察者模式的实现。`addListener()` 方法会将添加监听器添加到 ChannelFuture 当中，并在 ChannelFuture 执行完毕的时候立刻通知已经注册的监听器。ChannelFuture 是被观察者，`addListener()` 方法用于添加观察者。

  ```java
  ChannelFuture channelFuture = channel.writeAndFlush(object);
  
  channelFuture.addListener(future -> {
      if (future.isSuccess()) {
          // do something
      } else {
          // do something
      }
  });
  ```

- 装饰器模式

  装饰器模式是对被装饰类的功能增强，在不修改被装饰类的前提下，能够为被装饰类添加新的功能特性。

  Netty 中 WrappedByteBuf 是 ByteBuf 装饰器，WrappedByteBuf 有两个子类 UnreleasableByteBuf 和 SimpleLeakAwareByteBuf，真正实现对 ByteBuf 的功能增强。

  ```java
  final class UnreleasableByteBuf extends WrappedByteBuf {
      private SwappedByteBuf swappedBuf;
  
      UnreleasableByteBuf(ByteBuf buf) {
          super(buf instanceof UnreleasableByteBuf ? buf.unwrap() : buf);
      }
  
      @Override
      public boolean release() {
          return false;
      }
      // 省略
  }
  ```

- 策略模式

  策略模式针对同一个问题提供多种策略的处理方式，这些策略之间可以相互替换，在一定程度上提高了系统的灵活性。

  EventExecutorChooser 提供了不同的策略选择 NioEventLoop，`newChooser()` 方法会根据线程池的大小是否是 2 的幂次，以此来动态的选择取模运算的方式，从而提高性能。

  ```java
  public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {
      public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();
  
      private DefaultEventExecutorChooserFactory() { }
  
      @SuppressWarnings("unchecked")
      @Override
      public EventExecutorChooser newChooser(EventExecutor[] executors) {
          if (isPowerOfTwo(executors.length)) {
              return new PowerOfTwoEventExecutorChooser(executors);
          } else {
              return new GenericEventExecutorChooser(executors);
          }
      }
      // 省略
  }
  ```

- 建造者模式

  建造者模式通过链式调用来设置对象的属性，Netty 中 ServerBootStrap 和 Bootstrap 引导器是最经典的建造者模式实现，在构建过程中需要设置非常多的参数，例如配置线程池 EventLoopGroup、设置 Channel 类型、注册 ChannelHandler、设置 Channel 参数、端口绑定等。








