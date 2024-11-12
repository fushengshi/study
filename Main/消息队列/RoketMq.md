# RocketMq

RocketMQ 是一个队列模型的消息中间件，具有高性能、高可靠、高实时、分布式的特点。

RocketMQ 中的消息模型就是按照主题模型所实现的。

> 主题模型的实现，每个消息中间件的底层设计不同，比如 Kafka 中的 分区 ，RocketMQ 中的 队列 ，RabbitMQ 中的 Exchange 。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240623144622387.png" alt="image-20240623144622387" style="zoom:80%;" />

- Producer

  生产者，支持分布式集群方式部署。

- Consumer

  消费者，支持以 push 推，pull 拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制。

- Broker

  消息队列服务器。主要负责消息的存储、投递和查询以及服务高可用保证。

  一个 Topic 分布在多个 Broker上，一个 Broker 可以配置多个 Topic ，是多对多的关系。

- NameServer

  注册中心 ，主要提供 Broker 管理 和 路由信息管理 。

  NameServer 就存放了 Broker 的路由表，消费者和生产者就从 NameServer 中获取路由表然后进行通信（生产者和消费者定期会向 NameServer 去查询相关的 Broker 的信息）。



## 顺序消费

RocketMQ 在主题上是无序的、它只有在队列层面才是保证有序 的。

将同一语义下的消息放入同一个队列。



## 重复消费

消费者实现**幂等**。



## 消息堆积

- 限流降级
- 增加多消费者实例水平扩展增加消费能力。增加消费者实例，同时还需要增加每个主题的队列数量。



## 分布式事务

在 RocketMQ 中使用的是 **事务消息** 和 **事务反查机制** 来解决分布式事务问题。

> RocketMQ 的事务消息也可以被认为是一个两阶段提交。

![image-20240610222218344](https://gitee.com/fushengshi/image/raw/master/image-20240610222218344.png)

- MQ 发送⽅在消息队列上开启⼀个事务，然后发送⼀个**半消息**给 MQ Server/Broker。

  事务提交之前，半消息对于 MQ 订阅⽅/消费者（⽐如第三⽅通知服务）不可⻅。

- 半消息发送成功的话，MQ 发送⽅就开始执⾏本地事务。

- MQ 发送⽅的本地事务执⾏成功的话，半消息变成正常消息，可以正常被消费。

  MQ 发送⽅的本地事务执⾏失败的话，会直接回滚。



## 存储机制

RocketMQ 采用的是混合型的存储结构，即为 Broker 单个实例下所有的队列共用一个日志数据文件来存储消息。

> Kafka采用分区分段结构。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240623181545045.png" alt="image-20240623181545045" style="zoom:80%;" />



RocketMQ 使用 ConsumeQueue 作为每个队列的索引文件提升读取消息的效率。可以直接根据队列的消息序号，计算出索引的全局位置，然后直接读取这条索引，再根据索引中记录的消息的全局位置找到消息。

- 生产者发送消息会指定 Topic、QueueId 和具体消息内容，在 Broker 直接全部顺序存储到了 CommitLog。

- 根据生产者指定的 Topic 和 QueueId 将这条消息本身在 CommitLog 的偏移（Offset）、消息本身大小、Tag 的 hash 值存入对应的 ConsumeQueue 索引文件中。

- 每个队列中都保存了每个消费者组的消费位置 ConsumeOffset，消费者拉取消息进行消费的时候只需要根据 ConsumeOffset 获取下一个未被消费的消息就行了。



## 刷盘机制

设置 Broker 的参数 FlushDiskType 调整刷盘策略（ASYNC_FLUSH 或者 SYNC_FLUSH）。

- 同步刷盘

  同步刷盘中需要等待一个刷盘成功的 ACK ，同步刷盘性能上会有较大影响。

- 异步刷盘

  异步刷盘往往是开启一个线程去异步地执行刷盘操作，降低了读写延迟 ，提高了 MQ 的性能和吞吐量。

  异步刷盘在 Broker 意外宕机的时候会丢失部分数据。



## 主从模式

![image-20240623211836572](https://gitee.com/fushengshi/image/raw/master/image-20240623211836572.png)

RocketMQ 4.5 以前的版本大多都是采用 Master-Slave 架构来部署，能在一定程度上保证数据的不丢失，也能保证一定的可用性。最大的问题就是当 Master Broker 挂了之后 ，没办法让 Slave Broker 自动 切换为新的 Master Broker，需要手动更改配置将 Slave Broker 设置为 Master Broker，以及重启机器。

RocketMQ 4.5 版本之后提供了 DLedger模式，使用 **Raft算法**，如果Master节点出现故障，可以自动选举出新的 Master 进行切换。




















































































