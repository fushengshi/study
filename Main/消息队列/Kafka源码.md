# Kafka 源码

## 日志模块

### 日志段

日志段（LogSegment）是 Kafka 保存消息的最小载体，消息是保存在日志段中的。每个日志段由两个核心组件构成：日志和索引。每个日志段对象保存自己的起始位移 baseOffset，每个 LogSegment 对象实例一旦被创建，它的起始位移就是固定的。

每个日志段对象会在磁盘上创建一组文件，包括消息日志文件（.log）、位移索引文件（.index）、时间戳索引文件（.timeindex）以及已中止（Aborted）事务的索引文件（.txnindex）。

```java
public void append(long largestOffset, long largestTimestampMs, long shallowOffsetOfMaxTimestamp,
    MemoryRecords records) throws IOException {
		...
        ensureOffsetInRange(largestOffset);

        // 通过Java.NIO FileChannel将内存中的消息对象写入到操作系统的 Page Cache
        long appendedBytes = log.append(records);

        if (largestTimestampMs > maxTimestampSoFar()) {
            maxTimestampAndOffsetSoFar = new TimestampOffset(largestTimestampMs, shallowOffsetOfMaxTimestamp);
        }

        // 日志段每写入4KB数据就要写入一个索引项
        if (bytesSinceLastIndexEntry > indexIntervalBytes) {
            offsetIndex().append(largestOffset, physicalPosition);
            timeIndex().maybeAppend(maxTimestampSoFar(), shallowOffsetOfMaxTimestampSoFar());
            bytesSinceLastIndexEntry = 0;
        }
        bytesSinceLastIndexEntry += records.sizeInBytes();
    }
}
```

> Java.NIO `filechannel.write()` 将 ByteBuffer 中的数据写入 Page Cache，实际刷盘由操作系统完成。
>
> FileChannel 提供了一个 `force()` 方法，用于通知操作系统进行及时的刷盘。



### 日志

日志是日志段的容器，里面定义了管理日志段的操作。

- Log End Offset（LEO）

  表示日志下一条待插入消息的位移值。Log 对象中的 LEO 永远指向下一条待插入消息，LEO 上面没有消息。

  LEO 对象被更新的时机：

  - Log 对象初始化时

    当 Log 对象初始化时，我们必须要创建一个 LEO 对象，并对其进行初始化。

  - 写入新消息时

    向 Log 对象插入新消息时，LEO 值就像一个指针一样不停地向右移动。

  - Log 对象发生日志切分（Log Roll）时

    发生日志切分，说明 Log 对象切换了 Active Segment，LEO 中的起始位移值和段大小数据都要被更新。

    > 日志切分：创建一个全新的日志段对象，并且关闭当前写入的日志段对象。通常发生在当前日志段对象已满的时候。

  - 日志截断（Log Truncation）时

    日志中的部分消息被删除了可能导致 LEO 值发生变化，从而要更新 LEO 对象。

- Log Start Offset

  表示日志当前对外可见的最早一条消息的位移值。

  Log Start Offset 更新时机：

  - Log 对象初始化时

    Log 对象初始化时要给 Log Start Offset 赋值，一般是将第一个日志段的起始位移值赋值给它。

  - Follower副本同步时

    一旦 Leader 副本的 Log 对象的 Log Start Offset 值发生变化，为了维持和 Leader 副本的一致性，Follower 副本也需要尝试去更新该值。

  - 日志截断时

    一旦日志中的部分消息被删除，可能会导致 Log Start Offset 发生变化。

  - 删除日志段时

    和日志截断类似，凡是涉及消息删除的操作都有可能导致 Log Start Offset 值的变化。

  - 删除消息时

    在 Kafka 中，删除消息就是通过抬高 Log Start Offset 值来实现的。

- 高水位（High Watermark，HW）

  > 见附录。

加载日志段：

```scala
private def loadSegmentFiles(): Unit = {
  for (file <- dir.listFiles.sortBy(_.getName) if file.isFile) {
    if (isIndexFile(file)) { // 如果是索引文件
      val offset = offsetFromFile(file)
      val logFile = LogFileUtils.logFile(dir, offset)
      if (!logFile.exists) { // 确保存在对应的日志文件，否则记录一个警告，并删除该索引文件
        Files.deleteIfExists(file.toPath)
      }
    } else if (isLogFile(file)) { // 如果是日志文件

      val baseOffset = offsetFromFile(file)
      val timeIndexFileNewlyCreated = !LogFileUtils.timeIndexFile(dir, baseOffset).exists()
      // 创建对应的LogSegment对象实例，并加入segments中
      val segment = LogSegment.open(dir, baseOffset, config, time, true, 0, false, "")

      try segment.sanityCheck(timeIndexFileNewlyCreated)
      catch {
          ...
          recoverSegment(segment)
      }
      segments.add(segment)
    }
  }
}
```



### 索引

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240623000127256.png" alt="image-20240623000127256" style="zoom:80%;" />

> Kafka 中的索引底层的实现原理是内存映射文件，即 Java 中的 MappedByteBuffer。
>
> 很多操作系统（比如 Linux），这段映射的内存区域实际上就是内核的页缓存（Page Cache）。数据不需要重复拷贝到用户态空间，避免了很多不必要的时间、空间消耗。

在 AbstractIndex 中，MappedByteBuffer 就是名为 `mmap` 的变量。

#### 改进版二分查找

当给定一个 Offset 时，Kafka 采用的是二分查找来高效定位不大于 Offset 的物理位移，然后找到目标消息。

大多数操作系统使用页缓存来实现内存映射，而目前几乎所有的操作系统都使用 LRU（Least Recently Used）或类似于 LRU 的机制来管理页缓存。Kafka 写入索引文件的方式是在文件末尾追加写入，而几乎所有的索引查询都集中在索引的尾部。原版的二分查找算法并没有考虑到缓存的问题，因此很可能会导致一些不必要的缺页中断（Page Fault）。此时，Kafka 线程会被阻塞，等待对应的索引项从物理磁盘中读出并放入到页缓存中。

改进版的二分查找策略总体的思路是，代码将所有索引项分成两个部分：热区（Warm Area）和冷区（Cold Area），然后分别在这两个区域内执行二分查找算法。查询最热那部分数据所遍历的 Page 永远是固定的，因此大概率在页缓存中。

```java
private int indexSlotRangeFor(ByteBuffer idx, long target, IndexSearchType searchEntity,
                              SearchResultType searchResultType) {
    if (entries == 0)
        return -1;

    // 确认热区首个索引项位于哪个槽
    int firstHotEntry = Math.max(0, entries - 1 - warmEntries());
    if (compareIndexEntry(parseEntry(idx, firstHotEntry), target, searchEntity) < 0) {
        // 如果在热区，搜索热区
        return binarySearch(idx, target, searchEntity,
            searchResultType, firstHotEntry, entries - 1);
    }

    if (compareIndexEntry(parseEntry(idx, 0), target, searchEntity) > 0) {
        switch (searchResultType) {
            case LARGEST_LOWER_BOUND:
                return -1;
            case SMALLEST_UPPER_BOUND:
                return 0;
        }
    }
    // 如果在冷区，搜索冷区
    return binarySearch(idx, target, searchEntity, searchResultType, 0, firstHotEntry);
}
```



## 请求处理

Kafka 服务器端也就是 Broker，其实就是一个不断接收外部请求、处理请求，然后发送处理结果的 Java 进程。高效地保存排队中的请求，是确保 Broker 高处理性能的关键。

### 请求处理全流程

![image-20240810000156634](https://gitee.com/fushengshi/image/raw/master/image-20240810000156634.png)

- Clients 或其他 Broker 发送请求给 Acceptor 线程

  Acceptor 线程通过调用 `accept()` 方法，创建对应的 SocketChannel，然后将该 Channel 实例传给 `assignNewConnection()` 方法，等待 Processor 线程将该 Socket 连接请求，放入到它维护的待处理连接队列中。

  后续 Processor 线程的 `run()` 方法会不断地从该队列中取出这些 Socket 连接请求，然后创建对应的 Socket 连接。

- Processor 线程处理请求，并放入请求队列

  Processor 线程处理请求，从底层 I/O 获取到发送数据，将其转换成 Request 对象实例，并最终添加到请求队列（requestQueue）。

- I/O 线程处理请求

  KafkaRequestHandler 线程循环地从请求队列中获取 Request 实例，然后交由 KafkaApis 的 `handle()` 方法，执行真正的请求处理逻辑。

- KafkaRequestHandler 线程将 Response 放入 Processor 线程的 Response 队列

  由 KafkaRequestHandler 线程的 KafkaApis 类完成，调用了 RequestChannel 的 `sendResponse()` 方法。

- Processor 线程发送 Response 给 Request 发送方

  Processor 线程取出 Response 队列中的 Response，返还给 Request 发送方。底层使用 Selector 实现真正的发送逻辑。



### SocketServer

这个组件是 Kafka 网络通信层中最重要的子模块（SocketServer.scala）。Acceptor 线程、Processor 线程和 RequestChannel 等对象都是网络通信的重要组成部分。

#### Acceptor

经典的 **Reactor 模式** 有个 Dispatcher 的角色，接收外部请求并分发给下面的实际处理线程。在 Kafka 中，这个 Dispatcher 就是 Acceptor 线程。

- NioSelector

  Java NIO 库的 Selector 对象实例，后续所有网络通信组件实现 Java NIO 机制的基础。

- Processors

  网络 Processor 线程池，Acceptor 线程在初始化时，需要创建对应的网络 Processor 线程池。

Acceptor 类继承 Runnable，`run()` 方法是处理 Reactor 模式 中分发逻辑的主要实现方法。

```scala
override def run(): Unit = {
	// 注册OP_ACCEPT事件
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    try {
        while (shouldRun.get()) {
            try {
                acceptNewConnections()
                closeThrottledConnections()
            }
            catch {
                case e: ControlThrowable => throw e
                case e: Throwable => error("Error occurred", e)
            }
        }
    } finally {
        closeAll()
    }
}

private def acceptNewConnections(): Unit = {
    // 每500毫秒获取一次就绪I/O事件
    val ready = nioSelector.select(500)
    // 如果有I/O事件准备就绪
    if (ready > 0) {
        val keys = nioSelector.selectedKeys()
        val iter = keys.iterator()
        while (iter.hasNext && shouldRun.get()) {
            try {
                val key = iter.next
                iter.remove()
                if (key.isAcceptable) {
                    // 调用accept方法创建Socket连接
                    accept(key).foreach { socketChannel =>
                        var retriesLeft = synchronized(processors.length)
                        var processor: Processor = null
                        do {
                            retriesLeft -= 1
                            // 指定由哪个Processor线程进行处理
                            processor = synchronized {
                                currentProcessorIndex = currentProcessorIndex % processors.length
                                processors(currentProcessorIndex)
                            }
                            currentProcessorIndex += 1
                        } while (!assignNewConnection(socketChannel, processor, retriesLeft == 0))
                    }
                } else
                throw new IllegalStateException("Unrecognized key state for acceptor thread.")
            } catch {
                case e: Throwable => error("Error while accepting connection", e)
            }
        }
    }
}
```



#### Processor

Acceptor 是做入站连接处理的，Processor 代码则是真正创建连接以及分发请求的地方。

```scala
override def run(): Unit = {
    try {
        while (shouldRun.get()) {
            try {
                // 创建新连接
                // 底层就是调用Java NIO的SocketChannel.register(selector, SelectionKey.OP_READ)
                configureNewConnections()

                // 发送Response，并将Response放入到inflightResponses临时队列
                processNewResponses()
                
                // 真正执行I/O操作逻辑
                poll() 
                
                // 将接收到的Request放入Request队列 requestChannel.sendRequest(req)
                processCompletedReceives()
                // 为临时Response队列中的Response执行回调逻辑
                processCompletedSends()
                // 处理因发送失败而导致的连接断开
                processDisconnected()
                // 关闭超过配额限制部分的连接
                closeExcessConnections()
            } catch {
                case e: Throwable => processException("Processor got uncaught exception.", e)
            }
        }
    } finally {
        debug(s"Closing selector - processor $id")
        CoreUtils.swallow(closeAll(), this, Level.ERROR)
    }
}
```



#### RequestChannel

传输 Request/Response 的通道。每个 RequestChannel 对象实例创建时，会定义一个队列来保存 Broker 接收到的各类请求，这个队列被称为请求队列或 Request 队列。Kafka 使用 Java 提供的阻塞队列 ArrayBlockingQueue 实现这个请求队列，并利用它天然提供的线程安全性来保证多个线程能够并发安全高效地访问请求队列。

![image-20240810000034020](https://gitee.com/fushengshi/image/raw/master/image-20240810000034020.png)

```scala
class RequestChannel(val queueSize: Int, val metricNamePrefix : String) extends KafkaMetricsGroup {
    import RequestChannel._
    val metrics = new RequestChannel.Metrics
    // request队列
    private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
    // Processor线程池
    private val processors = new ConcurrentHashMap[Int, Processor]()

    val requestQueueSizeMetricName = metricNamePrefix.concat(RequestQueueSizeMetric)
    val responseQueueSizeMetricName = metricNamePrefix.concat(ResponseQueueSizeMetric)

	...
}
```

Kafka Broker 端所有网络线程都是在 RequestChannel 中维护的，RequestChannel 下的 Processor 线程池负责具体的请求处理逻辑。

```scala
def sendRequest(request: RequestChannel.Request): Unit = {
    requestQueue.put(request)
}

def sendResponse(response: RequestChannel.Response): Unit = {
    ...

    // 找出response对应的Processor线程，即request当初是由哪个Processor线程处理的
    val processor = processors.get(response.processor)
    // 将response对象放置到对应Processor线程的Response队列中
    if (processor != null) {
      processor.enqueueResponse(response)
    }
}
```

除了 Processor 的管理之外，RequestChannel 的另一个重要功能是处理 Request 和 Response，具体表现为收发 Request 和发送 Response。



### KafkaRequestHandlerPool

KafkaRequestHandlerPool 是真正处理 Kafka 请求的地方。

```scala
def run(): Unit = {
    threadRequestChannel.set(requestChannel)
    while (!stopped) {
        val startSelectTime = time.nanoseconds
		// 从请求队列中获取下一个待处理的请求
        val req = requestChannel.receiveRequest(300)
        val endTime = time.nanoseconds
        val idleTime = endTime - startSelectTime
        aggregateIdleMeter.mark(idleTime / totalHandlerThreads.get)

        req match {
            case RequestChannel.ShutdownRequest => completeShutdown()
            return

            case callback: RequestChannel.CallbackRequest =>
            val originalRequest = callback.originalRequest
            try {
               ...
            }
            // 普通请求
            case request: RequestChannel.Request =>
            try {
                request.requestDequeueTimeNanos = endTime
                threadCurrentRequest.set(request)
                // 由KafkaApis.handle方法执行相应处理逻辑
                apis.handle(request, requestLocal)
            } catch {
            } finally {
                threadCurrentRequest.remove()
                request.releaseBuffer()
            }

            case RequestChannel.WakeupRequest => 
            warn("Received a wakeup request outside of typical usage.")

            case null => // continue
        }
    }
    completeShutdown()
}
```



## KafkaApis

KafkaApis 是 Broker 端所有功能的入口，同时关联了超多的 Kafka 组件。`handle` 方法封装了所有 RPC 请求的具体处理逻辑。每当社区新增 RPC 协议时，增加对应的 `handle×××Request` 方法和 case分支都是首要的。

```scala
override def handle(request: RequestChannel.Request, requestLocal: RequestLocal): Unit = {
    try {
        // 根据请求头部信息中的apiKey字段判断属于哪类请求
        request.header.apiKey match {
            case ApiKeys.PRODUCE => handleProduceRequest(request, requestLocal)
            case ApiKeys.FETCH => handleFetchRequest(request)
           	...
            case _ => throw new IllegalStateException(s"")
        }
    } catch {
    } finally {
        replicaManager.tryCompleteActions()
        if (request.apiLocalCompleteTimeNanos < 0)
        request.apiLocalCompleteTimeNanos = time.nanoseconds
    }
}
```



# 附录

## 高水位机制

![image-20240810001201554](https://gitee.com/fushengshi/image/raw/master/image-20240810001201554.png)

Kafka 所有副本都有对应的高水位和 LEO 值。Kafka 使用 Leader 副本的高水位来定义所在分区的高水位，分区的高水位就是其 Leader 副本的高水位。只有在这个 Offset 以下的消息才能被消费者读到，高水位的具体值取决于主从副本数据同步的状态（小于等于 HW 值的所有消息认为是已备份（replicated）的）。

Leader 副本处理生产者请求的逻辑：

- 写入消息到本地磁盘

- 更新分区高水位值

  - 获取 Leader 副本所在 Broker 端保存的所有 Follower 副本 LEO 值 (LEO-1, LEO-2, … , LEO-n)。
  - 获取 Leader 副本高水位值 currentHW。
  - 更新 `currentHW = max(currentHW, min(LEO-1, LEO-2, … , LEO-n))`。

  > 在 Leader 副本所在的 Broker 上，还保存了其他 Follower 副本的 LEO 值。

Leader副本处理 Follower 副本拉取（FETCH）消息的逻辑：

- 读取磁盘（或页缓存）中的消息数据。

- 使用 Follower 副本发送请求中的位移值更新 Leader 副本所在的 Broker 上 Follower 副本 LEO 值。

- 更新分区高水位值

  具体步骤与处理生产者请求的步骤相同。

Follower副本从 Leader 副本拉取消息的处理逻辑：

- 写入消息到本地磁盘。
- 更新 LEO 值。
- 更新高水位值
  - 获取 Leader 副本发送的高水位值 currentHW。
  - 获取更新过的 LEO 值 currentLEO。
  - 更新高水位为 `min(currentHW, currentLEO)`。

### 故障恢复

- Follower 副本故障

  Follower 发生故障后会被临时踢出 ISR。

  Follower 恢复后，Follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 Leader 进行同步。

  Follower 追上 Leader 之后（LEO 大于等于该 Partition 的 HW），重新加入 ISR。

- Leader 副本故障

  Leader 发生故障之后，会从 ISR 中选出一个新的 Leader，为保证多个副本之间的数据一致性，其余的 Follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 Leader 同步数据。

  这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。

> ISR（In-Sync-Replica）处于同步状态的副本集合，是指副本数据和主副本数据相差在一定返回（时间范围或数量范围）之内的副本，Leader 副本肯定是一直在 ISR 中的。当 Leader 副本宕机，新的 Leader 副本将从 ISR 中被选出。

### Leader Epoch 机制

Kafka 0.11版本正式引入了 Leader Epoch 概念，来规避因高水位更新错配导致的各种不一致问题。

> 根本原因在于 HW 值被用于衡量副本备份的成功与否，但 HW 值的更新是异步延迟的，特别是需要额外的 FETCH 请求处理流程才能更新，故这中间发生的任何崩溃都可能导致 HW 值的过期。

- Epoch

  一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。

- 起始位移（Start Offset）

  Leader 副本在该 Epoch 值上写入的首条消息的位移。

工作机制：

- 当副本成为 Leader 时

  生产者有新消息发送过来，会生成新的 Leader Epoch 以及 LEO 添加到 leader-epoch-checkpoint 文件中。

- 当副本变成 Follower 时

  - 发送 LeaderEpochRequest 请求给 Leader 副本，该请求包括了 Follower 中最新的 epoch 版本（LastEpoch）。

  - Leader 返回给 Follower  的响应中包含了一个 LastOffset。

    如果 Follower LastEpoch = Leader LastEpoch，则 LastOffset = Leader LEO。

    如果 Follower LastEpoch != Leader LastEpoch，则 LastOffset = 大于 Follower LastEpoch 中最小 Epoch 的 Start Offset 值。

    > 假设 Follower LastEpoch = 1，此时 Leader 有 (1, 20) (2, 80) (3, 120)，则 LastOffset = 80。

  - Follower 拿到 LastOffset 之后，会对比当前 LEO 值是否大于 LastOffset，如果当前 LEO 大于 LastOffset，则从 LastOffset 截断日志。

  - Follower 开始发送 FETCH 请求给 Leader 保持消息同步。



## 控制器（Controller）

控制器（Controller）为集群中的所有主题分区选举领导者副本，承载着集群的全部元数据信息，并负责将这些元数据信息同步到其他Broker上。

> 在 KRaft 模式下，一部分 Kafka Broker 被指定为 Controller，另一部分则为 Broker。这些 Controller 的作用就是以前由 Zookeeper 提供的共识服务，并且所有的元数据都将存储在 Kafka 主题中并在内部进行管理。



## 分区 Leader 选举策略

每个分区都必须选举出 Leader 才能正常提供服务，对于分区而言，Leader 副本是非常重要的角色。

- offlinePartitionLeaderElection

  因为 Leader 副本下线而引发的分区 Leader 选举算法。

- reassignPartitionLeaderElection

  因为执行分区副本重分配操作而引发的分区 Leader 选举算法。

- preferredReplicaPartitionLeaderElection

  因为执行 Preferred副本 Leader 选举而引发的分区 Leader 选举算法。

- controlledShutdownPartitionLeaderElection

  因为正常关闭 Broker 而引发的分区 Leader 选举算法。

它们的逻辑几乎是相同的，大概的原理都是从 AR，或给定的副本列表中寻找存活状态的ISR副本。

- 代码首先会顺序搜索 AR 列表（分区的副本列表，Assigned Replicas），把第一个同时满足以下两个条件的副本作为新的Leader。

  1. 该副本是存活状态，即副本所在的 Broker 依然在运行中。

  2. 该副本在 ISR 列表中。

- 如果没有 ISR 中的副本，代码会检查是否开启了 Unclean Leader 选举。

  如果可以接受和原来的 Leader 节点的数据存在较大的延迟，会造成数据丢失的情况，可以通过 `unclean.leader.election.enable` 参数设置。如果开启，只要满足上面第一个条件即可，如果未开启，则本次 Leader 选举失败，没有新 Leader 被选出。



## TimingWheel

> DelayQueue 插入和删除队列元素的时间复杂度是O(logN)。对于 Kafka 这种非常容易积攒几十万个延时请求的场景来说，该数据结构的性能是瓶颈。

时间轮有简单时间轮（Simple Timing Wheel）和分层时间轮（Hierarchical Timing Wheel）两类。

![image-20240811164120682](https://gitee.com/fushengshi/image/raw/master/image-20240811164120682.png)

Kafka 采用的是分层时间轮，时间轮共有两个层级，分别是Level 0和Level 1。每个时间轮有8个 Bucket，每个 Bucket 下是一个双向循环链表，用来保存延迟请求。











