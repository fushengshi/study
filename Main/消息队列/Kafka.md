# Kafka

Kafka 是一个分布式流式处理平台。

- 消息队列：发布和订阅消息流，这个功能类似于消息队列，这也是 Kafka 也被归类为消息队列的原因。

- 容错的持久方式存储记录消息流：Kafka 会把消息持久化到磁盘，有效避免了消息丢失的风险。

- 流式处理平台： 在消息发布的时候进行处理，Kafka 提供了一个完整的流式处理类库。



## 消息模型

发布订阅模型（Pub-Sub） 使用主题（Topic） 作为消息通信载体，类似于广播模式，发布者发布一条消息，该消息通过主题传递给所有的订阅者，在一条消息广播之后才订阅的用户则是收不到该条消息的。

> RocketMQ 的消息模型和 Kafka 基本是完全一样的。唯一的区别是 Kafka 中没有队列这个概念，与之对应的是 Partition（分区）。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240622182204114.png" alt="image-20240622182204114" style="zoom:80%;" />

- Producer（生产者）

  产生消息的一方。

- Consumer（消费者）

  消费消息的一方。

- Consumer Group（消费者组）

  消费者组是 Kafka 提供的可扩展且具有容错性的消费者机制。多个实例共同订阅若干个主题，实现共同消费。当某个实例挂掉的时候，其他实例会自动地承担起它负责消费的分区。

  **每个 Partition 只能由消费组中的一个消费者进行消费。**

- Broker（代理）

  可以看作是一个独立的 Kafka 服务器，多个 Kafka Broker 组成一个 Kafka Cluster。

- Topic（主题）

  Producer 将消息发送到特定的主题，Consumer 通过订阅特定的 Topic 来消费消息。

- Partition（分区）

  Partition 属于 Topic 的一部分。一个 Topic 可以有多个 Partition，同一 Topic 下的 Partition 可以分布在不同的 Broker 上，一个 Topic 可以横跨多个 Broker。

- offset（位移）

  每个主题分区下的每条消息都被赋予了一个唯一的 ID 数值，用于标识它在分区中的位置。这个 ID 数值称为位移或偏移量。消息被写入到分区日志，位移值不能被修改。

### 分区机制

![image-20240811132340123](https://gitee.com/fushengshi/image/raw/master/image-20240811132340123.png)

Kafka 有主题（Topic）的概念，它是承载真实数据的逻辑容器，在主题之下还分为若干个分区，主题下的每条消息只会保存在某一个分区中。

分区的作用就是提供负载均衡的能力，实现系统的高伸缩性（Scalability）。不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的，这样每个节点的机器都能独立地执行各自分区的读写请求处理。并且，可以通过添加新的节点机器来增加整体系统的吞吐量。

分区策略是决定生产者将消息发送到哪个分区的算法：

- 轮询策略（Round-robin 策略）：顺序分配。

- 随机策略：随意地将消息放置到任意一个分区上。

- 按消息键保序策略

  Kafka 允许为每条消息定义消息键，保证同一个Key的所有消息都进入到相同的分区里面。

  每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略。



## 顺序消费

Kafka 只能保证 Partition（分区）中的消息有序。消息在被追加到 Partition 的时候都会分配一个特定的偏移量（Offset）。Kafka 通过 Offset 来保证消息在分区内的顺序性。

发送消息的时候指定了 Partition ，所有消息都会被发送到指定的 Partition。并且，同一个 key 的消息可以保证只发送到同一个 partition，可以采用表或对象的 id 来作为 key 。

每个 consumer group 管理自己的位移信息，同时可以引入 checkpoint 机制定期持久化，简化应答机制的实现。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240622200509022.png" alt="image-20240622200509022" style="zoom:80%;" />

## 重复消费

**原因：**

- 服务端侧已经消费的数据没有成功提交 Offset（根本原因）。
- 由于服务端处理业务时间长或者网络链接等等原因让 Kafka 认为服务假死，触发了分区 Rebalance。

**解决方案：**

- 消费消息服务做幂等校验。



## 消息延迟

### 减少消息延迟

- 消费端

  - 优化消费代码提升性能

    可以考虑使用多线程的方式来增加处理能力，接收到消息之后，把消息丢到线程池中来异步地处理。

  - 增加消费者的数量

    在 Kafka 中，一个 Topic 可以配置多个 Partition（分区），一个分区只能被一个消费者消费。Topic 的分区数量决定了消费的并行度，增加多余的消费者没有用处，可以通过增加 Partition 提高消费者的处理能力。


### 监控消息延迟

- 使用消息队列提供的工具，监控消息的堆积。

- 通过生成监控消息的方式来监控消息的延迟情况。

  定义一种特殊的消息，然后启动一个监控程序，将这个消息定时地循环写入到消息队列中消息的内容可以是生成消息的时间戳，并且也会作为队列的消费者消费数据。

  业务处理程序消费到这个消息时直接丢弃掉，而监控程序在消费到这个消息时和这个消息的生成时间做比较 ，如果时间差达到某一个阈值就报警。



## 高可用

### 多副本机制

Kafka 为分区（Partition）引入了多副本（Replica）机制。分区（Partition）中的多个副本之间会有一个 Leader，其他副本称为 Follower。发送的消息会被发送到 Leader 副本，然后 Follower 副本从 Leader 副本中拉取（FETCH）消息进行同步。

- AR（Assigned Replicas）

  分区的副本列表。

- ISR（In-Sync Replicas）

  同步副本集合，是指副本数据和主副本数据相差在一定返回（时间范围或数量范围）之内的副本。

  Leader 副本肯定是一直在 ISR 中的。当 Leader 副本宕机，新的 Leader 副本将从 ISR 中被选出。

Kafka 通过给特定 Topic 指定多个 Partition，各个 Partition 可以分布在不同的 Broker 上，能提供比较好的并发能力（负载均衡）。Partition 可以指定对应的 Replica 数，提高了消息存储的安全性、容灾能力，不过增加了所需要的存储空间。

> 自Kafka 2.4版本开始，社区通过引入新的 Broker 端参数，允许 Follower 副本有限度地提供读服务。



### 高水位机制

![image-20240810001201554](https://gitee.com/fushengshi/image/raw/master/image-20240810001201554.png)

Kafka 所有副本都有对应的高水位和 LEO 值。Kafka 使用 Leader 副本的高水位来定义所在分区的高水位，分区的高水位就是其 Leader 副本的高水位。只有在这个 Offset 以下的消息才能被消费者读到，高水位的具体值取决于主从副本数据同步的状态（小于等于 HW 值的所有消息认为是已备份（replicated）的）。

Leader副本处理生产者请求的逻辑：

- 写入消息到本地磁盘

- 更新分区高水位值

  - 获取 Leader 副本所在 Broker 端保存的所有 Follower 副本 LEO 值 (LEO-1, LEO-2, … , LEO-n)。
  - 获取 Leader 副本高水位值 currentHW。
  - 更新 `currentHW = max(currentHW, min(LEO-1, LEO-2, … , LEO-n))`。

  > 在 Leader 副本所在的 Broker 上，还保存了其他 Follower 副本的 LEO 值。

Leader 副本处理 Follower 副本拉取（FETCH）消息的逻辑：

- 读取磁盘（或页缓存）中的消息数据。

- 使用 Follower 副本发送请求中的位移值更新 Leader 副本所在的 Broker 上 Follower 副本 LEO 值。

- 更新分区高水位值

  具体步骤与处理生产者请求的步骤相同。

Follower 副本从 Leader 副本拉取消息的处理逻辑：

- 写入消息到本地磁盘。
- 更新 LEO 值为 currentLEO。
- 更新高水位值
  - 获取 Leader 副本发送的高水位值 leaderHW。
  - 获取更新过的 LEO 值 currentLEO。
  - 更新高水位为 `min(leaderHW, currentLEO)`。

#### 故障恢复

- Follower 副本故障

  Follower 发生故障后会被临时踢出 ISR。

  Follower 恢复后，Follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 Leader 进行同步。

  Follower 追上 Leader 之后（LEO 大于等于该 Partition 的 HW），重新加入 ISR。

- Leader 副本故障

  Leader 发生故障之后，会从 ISR 中选出一个新的 Leader，为保证多个副本之间的数据一致性，其余的 Follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 Leader 同步数据。

  这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。

#### Leader Epoch 机制

Kafka 0.11版本正式引入了 Leader Epoch 概念，来规避因高水位更新错配导致的各种不一致问题。

> 根本原因在于 HW 值被用于衡量副本备份的成功与否，但 HW 值的更新是异步延迟的，特别是需要额外的FETCH请求处理流程才能更新，故这中间发生的任何崩溃都可能导致 HW 值的过期。

- Epoch：一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。
- 起始位移（Start Offset）：Leader 副本在该 Epoch 值上写入的首条消息的位移。

工作机制：

- 当副本成为 Leader 时

  生产者有新消息发送过来，会生成新的 Leader Epoch 以及 LEO 添加到 leader-epoch-checkpoint 文件中。

- 当副本变成 Follower 时

  - 发送 LeaderEpochRequest 请求给 Leader 副本，该请求包括了 Follower 中最新的 epoch 版本（Follower LastEpoch）。

  - Leader 返回给 Follower  的响应中包含了一个 LastOffset。

    如果 Follower LastEpoch = Leader LastEpoch，则 LastOffset = Leader LEO。

    如果 Follower LastEpoch != Leader LastEpoch，则 LastOffset = 大于 Follower LastEpoch 中最小 Epoch 的 Start Offset 值。

    > 假设 Follower LastEpoch = 1，此时 Leader 有 (1, 20) (2, 80) (3, 120)，则 LastOffset = 80。

  - Follower 拿到 LastOffset 之后，会对比当前 LEO 值是否大于 LastOffset，如果当前 LEO 大于 LastOffset，则从 LastOffset 截断日志。

  - Follower 开始发送 FETCH 请求给 Leader 保持消息同步。



### KRaft模式

Kafka 2.8 之后，引入了基于 Raft 协议的 KRaft（Kafka Raft）模式，不再依赖 Zookeeper。在 Zookeeper 模式下，需同时运维 ZK 和 Broker，而 KRaft 模式仅需 3 个节点即可构成最小生产集群，且通信协调基于 Raft 协议，增强了一致性。

![image-20240623015235918](https://gitee.com/fushengshi/image/raw/master/image-20240623015235918.png)



在此模式下，一部分Kafka Broker被指定为 Controller，另一部分则为 Broker。这些 Controller 的作用就是以前由 Zookeeper 提供的共识服务，并且所有的元数据都将存储在Kafka主题中并在内部进行管理。

#### 控制器 Leader 选举

在实际应用中要求 Controller Quorum 的节点数为奇数且大于等于3，最多可以容忍 (n / 2 - 1) 个节点失败。当然，只有一个节点能成为领导节点即 Active Controller，领导选举就依赖于内置的 **Raft 协议变种（KRaft）**实现。

KRaft 集群中，所有控制器代理都维护一个保持最新的内存元数据缓存，以便任何控制器都可以在需要时接管作为活动控制器（active controller），所有 Broker 都与控制器进行通信，其中活动控制器将处理与其它 Broker 通信对元数据的更改。

> 对比 Redis 哨兵 Leader 选举。

#### 分区副本主从切换

生产者和消费者只与 Leader 副本交互，其它副本只是 Leader 副本的拷贝，只是为了保证消息存储的安全性。当 Leader 副本发生故障时会进行故障转移。

由控制器执行。

- 代码首先会顺序搜索分区副本列表（AR），把第一个同时满足以下两个条件的副本作为新的 Leader。

  1. 该副本是存活状态，即副本所在的 Broker 依然在运行中。

  2. 该副本在 ISR 列表中。

- 如果没有 ISR 中的副本，代码会检查是否开启了 Unclean Leader 选举。

  如果可以接受和原来的Leader节点的数据存在较大的延迟，会造成数据丢失的情况，可以通过 `unclean.leader.election.enable` 参数设置。如果开启，只要满足上面第一个条件即可，如果未开启，则本次 Leader 选举失败，没有新 Leader 被选出。

> 对比 Redis 主从切换。



### Zookeeper模式

在 Kafka 2.8 之前，Kafka 依赖 Zookeeper 集群做元数据管理和集群的高可用（Zookeeper提供共识服务）。

- Broker 注册

  在 Zookeeper 上会有一个专门用来进行 Broker 服务器列表记录的节点。每个 Broker 在启动时都会到 Zookeeper 上进行注册，即到 `/brokers/ids` 下创建属于自己的节点，将自己的 IP 地址和端口等信息记录到该节点中去。

- Topic 注册

  在 Kafka 中，同一个 Topic 的消息会被分成多个分区并将其分布在多个 Broker 上，这些分区信息及与 Broker 的对应关系也都是由 Zookeeper 在维护。

- 负载均衡

  上面也说过了 Kafka 通过给特定 Topic 指定多个 Partition，而各个 Partition 可以分布在不同的 Broker 上，这样便能提供比较好的并发能力。 对于同一个 Topic 的不同 Partition，Kafka 会尽力将这些 Partition 分布到不同的 Broker 服务器上。

  当生产者产生消息后也会尽量投递到不同 Broker 的 Partition 里面。当 Consumer 消费的时候，Zookeeper 可以根据当前的 Partition 数量以及 Consumer 数量来实现动态负载均衡。

#### 控制器 Leader 选举

控制器是 Kafka 的核心组件，它的主要作用是在 Zookeeper 的帮助下管理和协调整个 Kafka 集群。集群中任意一个 Broker 都能充当控制器的角色，但在运行过程中，只能有一个 Broker 成为控制器。

- 集群中第一个启动的 Broker 会通过在 Zookeeper 中创建临时节点 `/controller` 来成为控制器。

  其它 Broker 启动时也会在 Zookeeper 中创建临时节点，但是发现节点已经存在，会收到一个异常得知控制器已经存在，就会在 Zookeeper 中创建 watch 对象，便于收到控制器变更的通知。

- 控制器由于网络原因与 Zookeeper 断开连接或者异常退出，其它 Broker 通过 watch 收到控制器变更的通知，就会去尝试创建临时节点 `/controller`。

  如果有一个 Broker 创建成功，其它 Broker 收到创建异常通知得知控制器已经存在，创建 watch 对象。

  为了解决 Controller 脑裂问题，ZooKeeper 中还有一个与 Controller 有关的持久节点 `/controller_epoch`，存放的是一个整形值的 epoch number（纪元编号），集群中每选举一次控制器，就会通过 Zookeeper 创建一个数值更大的 epoch number，如果 Broker 收到比这个 epoch 数值小的数据会忽略消息。

  > 控制器所在 Broker 挂掉或者 Full GC 停顿时间太长超过 Zookeeper `session timeout` 出现假死，Kafka集群选举出新的控制器，如果之前被取代的控制器又恢复正常了，依旧是控制器身份，这样集群就会出现两个控制器，这就是控制器脑裂问题。

#### 分区副本主从切换

同KRaft模式。



### Rebalance 机制

消费者的重平衡 Rebalance 是让一个消费组的所有消费者就如何消费订阅 Topic 的所有分区达成共识的过程，在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 的完成。

Rebalance 的触发条件：

- 消费组成员个数发生变化。

  例如有 Consumer 实例加入或离开该消费组。

  要为业务处理逻辑留下充足的时间 `heartbeat.interval.ms` 使 Consumer 不会因为处理这些消息的时间太长而引发 Rebalance，但也不能时间设置过长导致 Consumer 宕机但迟迟没有被踢出 Group。

- 订阅的 Topic 个数发生变化。

- 订阅 Topic 的分区数发生变化。

Rebalance过程：

- 所有成员都向 Group Coordinator 发送 JoinGroup 请求，请求加入消费组。所有成员都发送了 JoinGroup 请求，Coordinator 会从中选择一个 Consumer 担任 Leader 的角色，并把组成员信息以及订阅信息发给 Leader，Leader 负责消费分配方案的制定。
- Leader 开始消费分配方案的制定，即 Consumer 消费订阅 Topic 的哪些 Partition。完成分配 Leader 会将这个方案封装进 SyncGroup 请求中发给 Coordinator。Coordinator 接收到分配方案之后会把方案塞进 SyncGroup 的 Response 中发给各个 Consumer。

#### 消费组选主

在 Kafka 的消费端，会有一个消费者协调器以及消费组，组协调器（Group Coordinator）需要为消费组内的消费者选举出一个消费组的Leader。

如果消费组内还没有 Leader，那么第一个加入消费组的消费者即为消费组的 Leader，如果某一个时刻 Leader 消费者由于某些原因退出了消费组，那么就会重新选举 Leader。组协调器中消费者的信息是以 HashMap 的形式存储的，其中 key 为消费者的 member_id，而value是消费者相关的元数据信息，而 Leader 的取值为 HashMap 中的第一个键值对的 key（等同于随机）。

> 消费组的 Leader 和 Coordinator 没有关联。消费组的 Leader 负责 Rebalance 过程中消费分配方案的制定。



### Kafka 脑裂

- KRaft 模式

  - 当 Quorum 节点向 Leader 拉取超时时，会进行新的 Controller Leader 选举。

  - 当 Controller 和 Quorum 节点出现网络分区时，集群中的其余 Quorum 节点开始新的选举。

    当前版本的 KRaft 实现，被隔离的 Leader 节点无法进行自动降级，导致集群中出现多个 Leader 从而导致脑裂。

  解决方法为 Controller 没有收到大多数量 Quorum 节点的 Fetch 请求时，将会进行新的 Leader 选举，将当前 Controller 节点（被隔离的 Leader）进行降级。

  > 对比 Redis 脑裂。

- Zookeeper 模式

  Kafka 脑裂问题是由于网络或其他原因导致多个 Broker 认为自己是 Controller，从而导致元数据不一致和分区状态混乱的问题。

  Kafka 通过 epoch number（纪元编号）解决脑裂问题，epoch number 是一个单调递增的版本号。

  - 假设有三个 Broker，分别是 Broker0，Broker1 和 Broker2。Broker0 是 Controller，在 ZooKeeper 中创建了 /controller 节点，并设置 epoch number = 1。Broker1 和 Broker2 在 /controller 节点设置了 Watcher。
  - Broker0 出现了 Full GC 等原因，与ZooKeeper的会话超时。ZooKeeper 删除了 /controller 节点，并通知 Broker1 和 Broker2 进行新的 Controller 选举。
  - Broker1 和 Broker2 同时尝试在 ZooKeeper 中创建 /controller 节点，假设 Broker1 成功成为新的 Controller，设置 epoch number = 2，并向 Broker2 同步数据。
  - Broker0 Full GC 结束后，继续向 Broker1 和 Broker2 同步数据，Broker1 和 Broker2 接收到数据后，发现 epoch number 小于当前值，就会拒绝这些消息，并通知 Broker0 最新的 epoch number，然后 Broker0 发现自己已经不是 Controller，与新的 Controller 建立连接。



## 消息可靠性

### 生产者到 Broker

kafka 发送消息使用异步发送带有回调的方式进行发送消息，再设置参数为发送消息失败后重试。

- acks=all

  - `0`：生产者写入消息不管服务器的响应，可能消息还在网络缓冲区，服务器没有收到消息，会丢失消息。

  - `1`：至少有一个副本收到消息才认为成功，一个副本那肯定就是集群的 Leader 副本了，但是如果 Leader 副本所在的节点挂了，Follower 没有同步这条消息，消息会丢失。

  - `all`：所有 ISR 都写入成功才算成功，所有 ISR 里的副本全挂了，消息才会丢失。

- retries=N

  设置一个非常大的值，可以让生产者发送消息失败后**重试**。

### Broker 持久化

消息在 Kafka 中是存储在本地磁盘上的，为了减少消息存储时对磁盘的随机 I/O，一般会将消息先写入到操作系统的 Page Cache 中，然后再找合适的时机刷新到磁盘上（异步刷盘），存在丢失消息的可能。

以集群方式部署 Kafka 服务，通过部署多个副本备份数据，保证消息尽量不丢失。

- replication.factor=N

  设置一个比较大的值，保证至少有 2 个或者以上的副本。

- min.insync.replicas=N

  代表消息如何才能被认为是写入成功，设置大于 1 的数，保证至少写入 1 个或者以上的副本才算写入消息成功。

- unclean.leader.election.enable=false

  这个设置意味着没有完全同步的分区副本不能成为 Leader 副本，如果为 true，没有完全同步 Leader 的副本成为 Leader 之后，会有消息丢失的风险。

### 消费者消费

因为重平衡（Rebalance）发生的时候，消费者会去读取上一次提交的偏移量，自动提交默认是每 5 秒一次，这会导致重复消费或者丢失消息。

- enable.auto.commit=false

  关闭自动提交位移，改为业务处理成功手动提交。



## 高性能

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240622232841285.png" alt="image-20240622232841285" style="zoom:80%;" />

### 生产消息

- 批量发送

  Kafka 采用了批量发送消息的方式，通过将多条消息按照分区进行分组，然后每次发送一个消息集合，从而大大减少了网络传输的开销。

- 消息压缩

  Kafka 支持的三种压缩算法：gzip、snappy、lz4。

  同时对多条消息进行压缩，能大幅减少数据量，从而更大程度提高网络传输率。压缩消息不仅仅减少了网络 I/O，它还大大降低了磁盘 I/O。

- 高效序列化

  用户可以根据实际情况选用快速且紧凑的序列化方式（比如 ProtoBuf、Avro）来减少实际的网络传输量以及磁盘存储量，进一步提高吞吐量。

- 内存池复用

  Kafka 内存池机制，它和连接池、线程池的本质一样，都是为了提高复用，减少频繁的创建和释放。

### 存储消息

- I/O 多路复用

  Kafka 采用典型的 Reactor 网络通信模型。

  <img src="https://gitee.com/fushengshi/image/raw/master/image-20240622233838532.png" style="zoom: 67%;" />

- 磁盘顺序写

  Kafka 作为消息队列，本质上就是一个队列，是先进先出的，而且消息一旦生产了就不可变。这种有序性和不可变性使得 Kafka 完全可以顺序写日志文件，仅仅将消息追加到文件末尾即可。

- Page Cache

  Kafka 利用了操作系统本身的缓存技术，在读写磁盘日志文件时，其实操作的都是内存，然后由操作系统决定什么时候将 Page Cache 里的数据刷入磁盘。

  ![image-20240622235714192](https://gitee.com/fushengshi/image/raw/master/image-20240622235714192.png)

  Page Cache 缓存的是最近会被使用的磁盘数据，利用的是时间局部性原理。预读到 Page Cache 中的磁盘数据，又利用了空间局部性原理。

  > 详见 内存.md 局部性原理。

  Kafka 作为消息队列，消息先是顺序写入，而且立马又会被消费者读取到。页缓存是 Kafka 做到高吞吐的重要因素之一。

- 分区分段结构

  <img src="https://gitee.com/fushengshi/image/raw/master/image-20240623181217931.png" alt="image-20240623181217931" style="zoom:67%;" />

  磁盘顺序写加上页缓存很好地解决了日志文件的高性能读写问题。但是如果一个 Topic 只对应一个日志文件，则只能存放在一台 Broker 机器上。

  分区（Partition）的概念和作用，它是 Kafka 并发处理的最小粒度，很好地解决了存储的扩展性问题。随着分区数的增加，Kafka 的吞吐量得以进一步提升。

  每个 Partition 又被分成了多个 Segment，分段对应真正的日志文件。

  > RocketMQ 是把消息都存一个文件中，Kafka 是一个分区一个文件。
  >
  > 每个分区一个文件在迁移或者数据复制层面上来说更灵活。
  >
  > 但是分区多了的话，写入需要频繁的在多个文件之间来回切换，对于每个文件来说是顺序写入，但是从全局看其实是随机写入，读取的时候也一样其实是随机读。

### 消费消息

- 稀疏索引

  <img src="https://gitee.com/fushengshi/image/raw/master/image-20240623000127256.png" alt="image-20240623000127256" style="zoom:80%;" />

  Kafka 所面临的查询场景能按照 Offset 或者 timestamp 查到消息即可。只需要在内存中维护一个**从 Offset 到日志文件偏移量**的映射关系。

  稀疏索引可以认为是在磁盘空间、内存空间、查找性能等多方面的一个折中。当给定一个 Offset 时，Kafka 采用的是二分查找来高效定位不大于 Offset 的物理位移，然后找到目标消息。

- 零拷贝

  - `mmap`

    索引文件的读写用到了 mmap，将磁盘文件与进程虚拟地址做映射，并不会系统调用以及额外的内存复制开销，从而提高了文件读取效率。

    > 基于 JDK 的 MappedByteBuffer 的 `map()` 函数，将磁盘文件映射到内存中。

  - `sendfile`

    TransportLayer 是 Kafka 传输层的接口，用于日志文件读写。将消息从磁盘文件中读出来，然后通过网卡发给消费者，数据直接从磁盘文件复制到网卡设备。

    > 使用 FileChannel 的 `transferTo` 方法，底层通过系统调用函数 `sendfile()` 方法实现。

    > RocketMQ 用 mmap + write 的方式，并且通过预热来减少大文件 mmap 因为缺页中断产生的性能问题。kafka 发送的效率更高，少了一次页缓存到 SocketBuffer 中的拷贝。

- 批量拉取

  和生产者批量发送消息类似，消息者也是批量拉取消息的，每次拉取一个消息集合，大大减少网络传输的开销。



## 分布式事务

Kafka 事务消息则是用在一次事务中需要发送多个消息的情况，基本上是配合幂等机制来实现 **Exactly Once 语义**的。

事务保证了多个分区写入的原子性，实现对**多个 Topic 的多个 Partition 的原子性写入**。通过事务协调器统一管理不同客户端发送到不同分区的操作，因为事务管理器中记录了这个操作，当客户端不管是 commit 还是 abort 的时候都可以通过事务协调器来查找同一个事务发送的分区，从而进行 commit 或者 abort 操作。

- Kafka 的事务机制，涉及到 Transactional Producer 和 Transactional Consumer，两者配合使用实现 Exactly Once 语义。

- Kafka 的事务机制，在底层依赖于幂等生产者。

  - 生产过程幂等

    每一个生产者一个唯一的 ID，并且生产的每一条消息一个唯一 ID，消息队列的服务端会存储 `生产者ID: 最后一条消息ID` 的映射。当某一个生产者产生新的消息时，消息队列服务端会比对消息 ID 是否与存储的最后一条 ID 一致，如果一致就认为是重复的消息，服务端会自动丢弃。

  - 消费过程幂等

    - 通用层面

      在消息被生产的时候，使用发号器给它生成一个全局唯一的消息 ID，消息被处理之后把这个 ID 存储在数据库中，在处理下一条消息之前，先从数据库里面查询这个全局 ID 是否被消费过，如果被消费过就放弃消费。

      > 如果消息在处理之后还没有写入数据库，消费者宕机重启之后发现数据库中没有这条消息，会重复执行两次消费逻辑。如果对于消息重复特别严格，需要引入事务机制，保证消息处理和写入数据库必须同时成功或者同时失败。

    - 业务层面

      通过乐观锁的方式，增加一个版本号的字段，在生产消息时先查询版本号，并将版本号同消息一起发送给消息队列。消费端在拿到消息和版本号后，在执行更新 SQL 时带上版本号。

      ```sql
      update xxx set amount = amount + 20, version = version + 1 where id = 1 and version = 1;
      ```


- 为支持事务机制，Kafka 引入了两个新的组件：Transaction Coordinator 和 Transaction Log。

  - Transaction Coordinator

    运行在每个 Kafka Broker 上的一个模块，是 Kafka Broker 进程承载的新功能之一。

  - Transaction Log

    Kafka 的一个内部 Topic（类似 __consumer_offsets ，是一个内部 Topic）。

    Transaction Log 有多个分区，每个分区都有一个 Leader，该 Leader 对应 Kafka Broker 上的 Transaction Coordinator 负责这些分区的写操作。

    Kafka 将日志文件格式进行了扩展，日志中除了普通的消息，还有**事务控制消息**专门用来标志一个事务的结束，有两种类型 commit 和 abort，分别用来表征事务已经成功提交或已经被成功终止。

- 开启了事务的生产者，生产的消息正常写到目标 Topic 中，同时通过 Transaction Coordinator 使用两阶段提交协议，将事务状态写到事务控制消息。

  - 两阶段提交协议的第一阶段

    Transactional Coordinator 更新内存中的事务状态为 prepare commit，并将状态持久化到 Transaction Log 中。

  - 两阶段提交协议的第二阶段

    Coordinator 首先将事务控制消息写入到目标 Topic 的目标 Partition。

    Coordinator 在写入事务控制消息后，会更新事务状态为  committed  或 abort， 并将该状态持久化到 Transaction Log 中。


- 开启了事务的消费者（配置读隔离级别为 read-committed），在内部会使用存储在目标 Topic Partition 中的事务控制消息，过滤掉没有提交的消息，包括回滚的消息和尚未提交的消息，确保只读到已提交的消息。

> 幂等性
>
> 幂等性确保了单分区的消息的一次处理语义，它通过 PID 和消息序列号来实现，当客户端发送到 Broker 端的消息序列号小于等于 Broker 端缓存的 PID 对应的 seq 的时候，忽略这个消息并保持，保证了消息不会被重复消费。幂等性的设置保证了 Kafka 的客户端重试，消息肯定能到达，这样就满足了 exactly once 的语义。










