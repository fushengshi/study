# Zookeeper

Apache ZooKeeper 是一个针对分布式系统的、可靠的、可扩展的协调服务，它通常作为统一命名服务、统一配置管理、注册中心（分布式集群管理）、分布式锁服务、Leader 选举服务等角色出现。

ZooKeeper 本身也是一个分布式应用程序。

![image-20240610171437256](https://gitee.com/fushengshi/image/raw/master/image-20240610171437256.png)

- Leader

  主节点，领导者。

  负责整个 ZooKeeper 集群的写操作，保证集群内事务处理的顺序性。同时负责整个集群中所有 Follower 节点与 Observer 节点的数据同步。

- Follower

  从节点。

  可以接收 Client 读请求并向 Client 返回结果，并不处理写请求，而是转发到 Leader 节点完成写入操作。

  Follower 节点还会参与 Leader 节点的选举。

- Observer

  ZooKeeper 集群中特殊的从节点，不会参与 Leader 节点的选举，其他功能与 Follower 节点相同。

  目的是增加 ZooKeeper 集群读操作的吞吐量，如果单纯依靠增加 Follower 节点,  因为 ZooKeeper 写数据时需要 Leader 将写操作同步给半数以上的 Follower 节点，ZooKeeper 集群的写能力会大大降低。

ZooKeeper 逻辑上是按照树型结构进行数据存储的，其中的节点称为 ZNode。每个 ZNode 有一个名称标识，即树根到该节点的路径（用 “/” 分隔），ZooKeeper 树中的每个节点都可以拥有子节点，这与文件系统的目录树类似。

![image-20240610171908047](https://gitee.com/fushengshi/image/raw/master/image-20240610171908047.png)

ZNode 节点类型有如下四种：

- 持久节点

  持久节点创建后，会一直存在，不会因创建该节点的 Client 会话失效而删除。

- 持久顺序节点

  持久顺序节点的基本特性与持久节点一致，创建节点的过程中，ZooKeeper 会在其名字后自动追加一个单调增长的数字后缀，作为新的节点名。

- 临时节点 

  创建临时节点的 ZooKeeper Client 会话失效之后，其创建的临时节点会被 ZooKeeper 集群自动删除。

  与持久节点的另一点区别是，临时节点下面不能再创建子节点。

- 临时顺序节点

  基本特性与临时节点一致，创建节点的过程中，ZooKeeper 会在其名字后自动追加一个单调增长的数字后缀，作为新的节点名。

除了可以通过 ZooKeeper Client 对 ZNode 进行增删改查等基本操作，还可以注册 **Watcher（事件监听器）** 监听 ZNode 节点、其中的数据以及子节点的变化。一旦监听到变化，则相应的 Watcher 即被触发，相应的 ZooKeeper Client 会立即得到通知。Watcher 有如下特点：

- 主动推送

  Watcher 被触发时，由 ZooKeeper 集群主动将更新推送给客户端，而不需要客户端轮询。

- 一次性 

  数据变化时，Watcher 只会被触发一次。如果客户端想得到后续更新的通知，必须要在 Watcher 被触发后重新注册一个 Watcher。

- 可见性 

  如果一个客户端在读请求中附带 Watcher，Watcher 被触发的同时再次读取数据，客户端在得到 Watcher 消息之前肯定不可能看到更新后的数据。更新通知先于更新结果。

- 顺序性 

  多个更新触发了多个 Watcher ，Watcher 被触发的顺序与更新顺序一致。



## 分布式锁

Redis 实现分布式锁性能较高，ZooKeeper 实现分布式锁可靠性更高。实际项目中应该根据业务的具体需求来选择。

ZooKeeper 分布式锁是基于 **临时顺序节点** 和 **Watcher（事件监听器）** 实现的。

- 加锁
  1. 首先我们要有一个持久节点 `/locks`，客户端获取锁就是在 `locks` 下创建临时顺序节点。
  2. 假设客户端 1 创建了 `/locks/lock1` 节点，创建成功之后，会判断  `lock1` 是否是 `/locks` 下最小的子节点。
  3. 如果 `lock1` 是最小的子节点，则获取锁成功。否则，获取锁失败。
  4. 如果获取锁失败，则说明有其他的客户端已经成功获取锁。客户端 1 并不会不停地循环去尝试加锁，而是在前一个节点比如 `/locks/lock0` 上注册一个事件监听器。这个监听器的作用是当前一个节点释放锁之后通知客户端 1（避免无效自旋），这样客户端 1 就加锁成功了。

- 释放锁
  1. 成功获取锁的客户端在执行完业务流程之后，会将对应的子节点删除。
  2. 成功获取锁的客户端在出现故障之后，对应的子节点由于是临时顺序节点，也会被自动删除，避免了锁无法被释放。
  3. 事件监听器其实监听的就是这个子节点删除事件，子节点删除就意味着锁被释放。



## ZAB 协议

Zookeeper 基于 ZAB（Zookeeper Atomic Broadcast），实现了主备模式下的系统架构，保持集群中各个副本之间的数据一致性。

ZAB 协议定义了四个阶段：

- Leader election（选举阶段）

  节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 Leader。

- Discovery（发现阶段）

  在这个阶段，Followers 跟准 Leader 进行通信，同步 Followers 最近接收的事务提议。

- Synchronization（同步阶段）

  同步阶段主要是利用 Leader 前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后准 Leader 才会成为真正的 Leader。

- Broadcast（广播阶段）

  到了这个阶段，ZooKeeper 集群才能正式对外提供事务服务，并且 Leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。

集群副本有三种状态：

- Leader：领导者。
- Follower：接受提议的跟随者。
- Observer：可以认为是领导者的的 Copy，不参与投票。

### Leader 选举

每次写成功的消息，都有一个全局唯一的标识 zxid。是 64 bit 的正整数，高 32 为叫 epoch 表示选举纪元，低 32 位是自增的 id，每写一次加一。全局唯一的zxid能够给选举和同步数据区分出优先级，核心思想是**递增的zxid顺序，保证了能够有优先级最高的节点当主。**

默认基于 TCP 的 FastLeaderElection：

Leader 是检测是否过半 Follower 心跳回复了，Follower 检测 Leader 是否发送心跳了。一旦 Leader 检测失败，则 Leader 进入 Looking 状态，其他 Follower 过一段时间因收不到 Leader 心跳也会进入 Looking 状态，从而出发新的 Leader 选举。

每个进入 Looking 状态的节点，会先把投票箱清空，然后通过广播投票给自己，再把投票消息发给其它机器，同时也在接受其他节点的投票。**投票信息包括 选举轮数（electionEpoch）、被投票节点的 zxid，被投票节点的编号**等等。

其他 Looking 状态的节点收到后首先判断票是否有效：

- 如果比本地选举轮数小，丢弃。

- 如果比本地选举轮数大，证明自己投票过期了。

  清空本地投票信息，更新选举轮数和结果为收到的内容，通知其他所有节点新的投票方案。

- 如果和本地选举轮数相等，比较优先级（zxid）：

  - 如果收到的优先级小，则忽略该投票。

  - 如果收到的优先级大，则更新自己的投票为对方发过来投票方案，把投票发出去。
  - 如果收到的优先级相等，则更新对应节点的投票。

每收集到一个投票后，查看已经收到的投票结果记录列表，看是否有节点能够达到一半以上的投票数：

- 如果有达到，则终止投票，宣布选举结束，更新自身状态，然后进行发现和同步阶段。
- 如果没有达到，则继续收集投票。

投票终止后，服务器开始更新自身状态，若过半的票投给了自己，则将自己的服务器状态更新为 Leading，否则将自己的状态更新为 Following。

### 主从同步

- 当 Leader 收到写操作时，先本地生成事务为事务生成 zxid，然后发给所有 Follower 节点。

- 当 Follower 收到事务时，先把提议事务的日志写到本地磁盘，成功后返回给 Leader。

- Leader 收到**超过半数**反馈后对事务提交。再通知所有的 Follower 提交事务，Follower 收到后也提交事务，提交后就可以对客户端进行分发了。

### 崩溃恢复

崩溃恢复需要经历 Leader 选举与数据恢复这两个阶段。

- 确保已经在 Leader 服务器上提交（Commit）的事务最终被所有服务器提交

  因为 Zab 协议中只要超过半数节点写入数据并响应，Leader 就会回复客户端写操作成功，如果此时 Leader 节点崩溃，可能只有少数的 Follower 节点收到了 Commit 消息。在选举过程中，需要收到了 Commit 信息节点成功当选 Leader，并将自身的数据同步给集群中的其它节点。

- 丢弃只在 Leader 上提出但没有被提交的事务

  Leader 节点可能是在等待半数 Follower 的 Ack 响应期间崩溃，也可能是在发送 Commit 消息之前崩溃，甚至可能是在广播事务提案的过程中崩溃。但无论是何种情况，可以确定的是集群中没有任何一个 Follower 将数据提交。因此当一个新 Leader 产生时，它会抛弃记录在事务日志但没有提交的事务。

  从客户端的角度看，因为 Leader 还没有将事务提交，所以也就没有收到主节点的写操作成功响应，这条事务是失败的。



## 附录

### 应用

- Kafka

  ZooKeeper 主要为 Kafka 提供 Broker 和 Topic 的注册以及多个 Partition 的负载均衡等功能。在 Kafka 2.8 之后，引入了基于 Raft 协议的 KRaft 模式，不再依赖 Zookeeper，大大简化了 Kafka 的架构。

- Hbase

  ZooKeeper 为 Hbase 提供确保整个集群只有一个 Master 以及保存和提供 regionserver 状态信息（是否在线）等功能。

- Hadoop

  ZooKeeper 为 Namenode 提供高可用支持。









