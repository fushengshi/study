# Tomcat

Tomcat 设计了两个核心组件连接器（Connector）和容器（Container），连接器负责处理 Socket 连接，负责网络字节流与 Request 和 Response 对象的转化。容器负责加载和管理 Servlet，以及具体处理 Request 请求。

![image-20240825181449027](https://gitee.com/fushengshi/image/raw/master/image-20240825181449027.png)

## Servlet

- Servlet 接口

  ```java
  public interface Servlet {
      void init(ServletConfig config) throws ServletException;
      
      ServletConfig getServletConfig();
      
      void service(ServletRequest req, ServletResponse res）throws ServletException, IOException;
      
      String getServletInfo();
      
      void destroy();
  }
  ```

  - `service()`，具体业务类在这个方法里实现处理逻辑。

    ServletRequest 用来封装请求信息，ServletResponse 用来封装响应信息，因此本质上这两个类是对通信协议的封装。

    > HTTP 协议中的请求和响应就是对应了 HttpServletRequest 和 HttpServletResponse 这两个类。可以通过 HttpServletRequest 来获取所有请求相关的信息，包括请求路径、Cookie、HTTP头、请求参数等。

  - `init()` 和 `destroy()`

    Servlet 容器在加载 Servlet 类的时候会调用 `init()` 方法，在卸载的时候会调用 `destroy()` 方法。

    > Spring MVC 中的 DispatcherServlet，就是在 `init()` 方法里创建了自己的 Spring 容器。

- Servlet 容器

  HTTP 服务器不直接跟业务类打交道，而是把请求交给 Servlet 容器去处理，Servlet 容器会将请求转发到具体的Servlet，如果这个 Servlet 还没创建，就加载并实例化这个 Servlet，然后调用这个 Servlet 的接口方法。Servlet 接口其实是Servlet容器跟具体业务类之间的接口。

- Web 应用

  根据 Servlet 规范，Web 应用程序有一定的目录结构，在这个目录下分别放置了 Servlet 的类文件、配置文件以及静态资源，Servlet 容器通过读取配置文件，找到并加载 Servlet。

  Servlet 规范里定义了 ServletContext 这个接口来对应一个 Web 应用。Web 应用部署好后，Servlet 容器在启动时会加载 Web 应用，并为每个 Web 应用创建唯一的 ServletContext 对象。

> Tomcat 和 Jetty 都按照 Servlet 规范的要求实现了 Servlet 容器，同时它们也具有 HTTP 服务器的功能。



## IO 多路复用模型

Tomcat 的网络模型是主从 Reactor 多线程模型。

| 支持的IO模型 | 特点                                                         |
| ------------ | ------------------------------------------------------------ |
| BIO          | 同步阻塞式 I/O，每一个请求都会创建一个线程，对性能开销大，不适合高并发场景。 |
| NIO          | 同步非阻塞式 I/O，基于多路复用 Selector 监测连接状态通知线程处理，达到非阻塞的目的，比传统的 BIO 效率高。Tomcat8.0 的默认方式。 |
| AIO          | 异步 I/O，基于各种回调处理。                                 |
| APR          | Apache Http 服务器的支持库，通过 JNI 技术调用本地库实现非阻塞 I/O。 |

### NioEndpoint

![image-20240825181543772](https://gitee.com/fushengshi/image/raw/master/image-20240825181543772.png)

NioEndpoint 利用 Java NIO API 实现了多路复用 I/O 模型，包含 LimitLatch、Acceptor、Poller、SocketProcessor 和 Executor 共5个组件。

- LimitLatch

  连接控制器，它负责控制最大连接数，NIO 模式下默认是10000，达到这个阈值后，连接请求被拒绝。

- Acceptor

  在一个单独的线程里，死循环调用 `accept()` 方法来接收新连接，一旦有新的连接请求到来，`accept()` 方法返回一个 Channel 对象，接着把 Channel 对象交给 Poller 去处理。

  ```java
  @Override
  public void run() {
      try {
          while (!stopCalled) {
              ...
              
              try {
                  U socket = null;
                  try {
  					// 接收SocketChannel，设置了 ServerSocketChannel 为阻塞模式，accept是阻塞的。
                      socket = endpoint.serverSocketAccept();
                  } catch (Exception ioe) {
                  }
                  errorDelay = 0;
                  if (!stopCalled && !endpoint.isPaused()) {
  					// setSocketOptions() 是关键方法
                      // 将SocketChannel注册到一个Poller上
                      if (!endpoint.setSocketOptions(socket)) {
                          endpoint.closeSocket(socket);
                      }
                  } else {
                      endpoint.destroySocket(socket);
                  }
              } catch (Throwable t) {
              }
          }
      } finally {
          stopLatch.countDown();
      }
      state = AcceptorState.ENDED;
  }
  ```

- Poller

  本质是一个 Selector，在单独线程里。Poller 在内部维护一个 Channel 数组，死循环不断检测 Channel 的数据就绪状态，一旦有 Channel 可读，就生成一个 SocketProcessor 任务对象扔给 Executor 去处理。

  ```java
  public class Poller implements Runnable {
      private Selector selector;
  
      public Poller() throws IOException {
          // 执行Selector.open()，执行本地方法ctl_create
          this.selector = Selector.open();
      }
      
      public void register(final NioSocketWrapper socketWrapper) {
          socketWrapper.interestOps(SelectionKey.OP_READ);
          PollerEvent event = null;
          if (eventCache != null) {
              event = eventCache.pop();
          }
          if (event == null) {
              event = new PollerEvent(socketWrapper, OP_REGISTER);
          } else {
              event.reset(socketWrapper, OP_REGISTER);
          }
          addEvent(event);
      }
      
      @Override
      public void run() {
          while (true) {
  			...
  
              if (!close) {
                  // 执行socketChannel.register()
                  // if (interestOps == OP_REGISTER) sc.register(getSelector(), SelectionKey.OP_READ, socketWrapper);
                  hasEvents = events();
                  if (wakeupCounter.getAndSet(-1) > 0) {
                      // 执行selector.select()，执行本地方法 epoll_ctl 和 epoll_wait
                      keyCount = selector.selectNow();
                  } else {
                      keyCount = selector.select(selectorTimeout);
                  }
                  wakeupCounter.set(0);
              }
  
              // select有返回 ready keys，进行处理
              Iterator<SelectionKey> iterator = keyCount > 0 ? selector.selectedKeys().iterator() : null;
              while (iterator != null && iterator.hasNext()) {
                  SelectionKey sk = iterator.next();
                  iterator.remove();
                  NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                  if (socketWrapper != null) {
                      // 处理 ready key
                      processKey(sk, socketWrapper);
                  }
              }
              timeout(keyCount,hasEvents);
          }
          getStopLatch().countDown();
      }
  }
  ```

- Executor

  线程池，负责运行 SocketProcessor 任务类，SocketProcessor的 `run()` 方法会调用 Http11Processor 来读取和解析请求数据。Http11Processor 是应用层协议的封装，它会调用容器获得响应，再把响应通过 Channel 写出。

### Nio2Endpoint

异步 I/O 模型里，内核做了很多事情，它把数据准备好，并拷贝到用户空间，再通知应用程序去处理，也就是调用应用程序注册的回调函数。Java 在操作系统 异步 IO API 的基础上进行了封装，提供了Java NIO.2 API，而 Tomcat 的异步 I/O 模型就是基于 Java NIO.2 实现的。

### AprEndpoint

AprEndpoint 是通过 JNI 调用 APR 本地库而实现非阻塞 I/O 的。

APR（Apache Portable Runtime Libraries）是 Apache 可移植运行时库，它是用 C 语言实现的，其目的是向上层应用程序提供一个跨平台的操作系统接口库。Tomcat 可以用它来处理包括文件和网络 I/O，从而提升性能。

APR提升性能：

- APR本地库的性能更好

  在需要频繁与操作系统进行交互场景，比如 Socket 网络通信，特别是 Web 应用使用了 TLS 来加密传输，TLS 协议在握手过程中有多次网络交互，在这种情况下 Java 和 C 语言程序相比有一定的差距。

- 通过 DirectByteBuffer 避免了 JVM 堆与本地内存之间的内存拷贝。

  > 高性能网络通信组件，比如 Netty，都是通过 DirectByteBuffer 来收发网络数据的。由于本地内存难于管理，Netty 采用了本地内存池技术。

- 通过 `sendfile` 特性避免了内核与应用之间的内存拷贝以及用户态和内核态的切换。















