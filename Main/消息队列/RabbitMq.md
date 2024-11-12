# RabbitMq

RabbitMQ 是采用 Erlang 语言实现 AMQP（Advanced Message Queuing Protocol，高级消息队列协议）的消息中间件，它最初起源于金融系统，用于在分布式系统中存储转发消息。

![image-20240825181800510](https://gitee.com/fushengshi/image/raw/master/image-20240825181800510.png)

RabbitMQ 整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息。

在 RabbitMQ 中，消息并不是直接被投递到 Queue（消息队列） 中的，中间还必须经过 Exchange（交换器）这一层，Exchange（交换器）会把消息分配到对应的 Queue（消息队列）中。

- Producer（生产者）

  生产消息的一方（邮件投递者）。

- Consumer（消费者）

  消费消息的一方（邮件收件人）。

- Queue（消息队列）

  RabbitMQ 中消息只能存储在队列中。RabbitMQ 的生产者生产消息并最终投递到队列中，消费者可以从队列中获取消息并消费。

  > 和 Kafka 这种消息中间件相反。Kafka 将消息存储在 Topic（主题）这个逻辑层面，相对应的队列逻辑只是 Topic 实际存储文件中的位移标识。

- Broker

  对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 服务实例。

- Connection

  Connection 是物理 TCP 连接。Connection 将应用与 RabbitMQ 连接在一起。Connection 会执行认证、IP解析、路由等底层网络任务。

- Channel

  Channel是物理TCP连接中的虚拟连接。当应用通过 Connection 与 RabbitMQ 建立连接后，所有的 AMQP 协议操作（例如创建队列、发送消息、接收消息等）都会通过 Connection 中的 Channel 完成。Channel 可以复用 Connection，即一个 Connection 下可以建立多个 Channel。Channel 不能脱离 Connection 独立存在，而必须存活在 Connection 中。当某个 Connection 断开时，该 Connection 下的所有 Channel 都会断开。

- Exchange（交换器）

  在 RabbitMQ 中，消息并不是直接被投递到 Queue（消息队列）中的，中间还必须经过 Exchange（交换器）这一层，Exchange（交换器）会把我们的消息分配到对应的 Queue（消息队列）中。

  Exchange Type（交换器类型）：

  - fanout

    把所有发送到该 Exchange 的消息路由到所有与它绑定的 Queue 中，不需要做任何判断操作。

  - direct

    把消息路由到那些 Bindingkey 与 RoutingKey 完全匹配的 Queue 中。

  - **topic**

    topic 类型的交换器在 direct 匹配规则上进行了扩展，也是将消息路由到 BindingKey 和 RoutingKey 相匹配的队列中。



## AMQP

RabbitMQ 就是 AMQP 协议的 Erlang 的实现（RabbitMQ 还支持 `STOMP2`、 `MQTT3` 等协议）AMQP 的模型架构 和 RabbitMQ 的模型架构是一样的，生产者将消息发送给交换器，交换器和队列绑定 。

AMQP 协议的三层：

- Module Layer

  协议最高层，主要定义了一些客户端调用的命令，客户端可以用这些命令实现自己的业务逻辑。

- Session Layer

  中间层，主要负责客户端命令发送给服务器，再将服务端应答返回客户端，提供可靠性同步机制和错误处理。

- TransportLayer

  最底层，主要传输二进制数据流，提供帧的处理、信道复用、错误检测和数据表示等。

AMQP 模型的三大组件：

- 交换器（Exchange）

  消息代理服务器中用于把消息路由到队列的组件。

- 队列（Queue）

  用来存储消息的数据结构，位于硬盘或内存中。

- 绑定（Binding）

  一套规则，告知交换器消息应该将消息投递给哪个队列。



## 顺序消费

- 拆分多个 Queue（消息队列），每个 Queue（消息队列）对应一个 Consumer（消费者）。

- Consumer（消费者）内部用内存队列做排队，然后分发给底层不同的 worker 来处理。



## 消息可靠性

### 生产者到 RabbitMQ

- 事务机制

  在生产者发送消息之前，开启事务，而后发送消息，如果消息发送至 RabbitMQ Server 失败后，进行事务回滚，重新发送。如果 RabbitMQ Server 接收到消息，则提交事务。

  同步操作，存在阻塞生产者的情况直到 RabbitMQ Server 应答，会很大程度上降低发送消息的性能。

- Confirm 机制

  `setConfirmCallback()` 可以将信道设置为 confirm 模式，所有消息会被指派一个消息唯一标识，当消息被发送到 RabbitMQ Server 后，Server 确认消息后生产者会回调设置的方法，从而实现生产者可以感知到消息是否正确无误的投递，从而实现发送方确认机制。

  异步操作，发送消息的吞吐量会得到很大提升。

> 事务机制和 Confirm 机制是互斥的，两者不能共存。

### 交换机到队列

确保生产者能将消息投递到交换机的前提下，RabbitMQ 同样提供了消息投递失败的策略配置来确保消息的可靠性。

`mandatory` 配置：

- true：失败后返回客户端。
- false：失败后自动删除两种策略。

`setReturnCallback()` 可实现当交换机路由到指定队列失败后回调方法，获取到退回的消息信息，进行相应的处理如记录日志或重传等等。

### RabbitMQ 持久化

- 持久化

  通过确保消息、交换机、队列的持久化操作可以保证消息的在 RabbitMQ Server 中不丢失。

- 高可用

  持久化之外还需要保证 RabbitMQ 的高可用性，否则 MQ 宕机或磁盘受损都无法确保消息的可靠性。

  > 详见高可用。

### 消费者消费

- 消费者应答机制

  设置手动确认消费者应答机制，只有消费者确认消费成功后才删除消息，从而避免消息的丢失。

- 死信队列

  当消息在一个队列中变成死信（dead message）之后，它能被重新发送到另一个交换器中。

> 大多数情况下消息丢失都是因为代码出现错误，那么这样无论进行多少次重发都是无法解决问题的，只会增加 CPU 的开销。更好的解决办法是通过记录日志的方式等待后续回溯时更好的发现问题并解决问题。
>
> 先将消息落库，而后生产者将消息发送给 MQ，使用数据库记录消息的消费情况，对于重试多次仍然无法消费成功的消息，后续通过定时任务调度的方式对这些无法消费成功的消息进行补偿。同样这样也带来了问题就是消息落库需要数据库磁盘 I/O 的开销，增大数据库压力同时降低了性能。



## 高可用

### 仲裁队列

仲裁队列（Quorum Queues）提供队列复制的能力，保障数据的高可用和安全性。使用仲裁队列可以在RabbitMQ节点间进行队列数据的复制，在一个节点宕机时，队列依旧可以正常运行。

- 镜像集群

  一种主从集群，普通的集群基础上，添加了主从备份功能，提高集群的数据可用性。有某个节点出现了故障，那么从节点还是有备份数据。镜像集群虽然支持主从，但主从同步并不是强一致的，某些情况下可能有数据丢失的风险。

  节点重新上线，以现存的主队列为准，从队列的镜像数据会丢失。

- 仲裁队列

  仲裁队列代替镜像集群，底层采用 **Raft协议** 确保主从的数据一致性。

  每条消息的持久化，当节点重新上线时不会丢数据，主副本会直接从从副本中断的地方开始复制消息。复制的过程是非阻塞的，所以整个队列不会因为新的副本加入而受到影响。

























