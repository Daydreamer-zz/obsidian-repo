# Kakfa

## 1.Kakfa概述

### 1.1 定义

Kafka是一个分布式的基于发布/订阅模式的消息队列，主要用于大数据实时处理领域。

### 1.2消息队列

#### 1.2.1 传统消息队列的应用场景

异步处理业务逻辑

![image.png](https://i.loli.net/2021/10/05/J9gt64HIEx1VCPQ.png)

![image.png](https://i.loli.net/2021/10/05/aq4JBgjQCeMfGw7.png)

使用消息队列的好处

- 解耦

  允许你独立的扩展或者修改两边的处理过程，只要确保他们遵守相同的接口约束。

- 可恢复性

  系统的一部分组件失效是，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，接入队列中的消息仍然可以在系统恢复后被处理。

- 缓冲

  有助于控制和优化数据经过系统的速度，解决生成消息和消费消息处理速度不一致的情况

- 灵活性&峰值处理能力

  在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理这类峰值访问的标准来投资资源随时待命无疑使巨大的浪费。使用消息队列能够是关键的组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃。

- 异步通信

  很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理进制，允许用户把一个消息放入队列，但是并不立即处理它。想往队列中放入多少消息就放多少，然后在需要的时候再去处理他们。

#### 1.2.2 消息队列的两种模式

- 点对点模式(一对一，消费者主动拉取数据，消息收到后消息清除)

  消息生产者生产消息发送到Queue中，然后消息消费者从Queue中取出并消费消息。消息被消费者消费以后，Queue中不再有存储，所以消息消费者不可能消费到已经被消费的消息。Queue支持存在多个消费者，但是对一个消息而言，只会有一个消费者可以消费。

  ![image.png](https://i.loli.net/2021/10/05/NO95VmDZsUoEcAG.png)

- 发布/订阅模式(**一对多**，消费者消费数据后不会清除消息)

  消息生产者(发布)将消息发布到topic中，同时又多个消息消费者(订阅)消息。和点对点方式不同，发布到topic的消息会被所有的订阅者消费。

  两种动作：消费者主动拉取数据、队列主动推送数据

  ![image.png](https://i.loli.net/2021/10/05/E5xHqkOmt1J9sde.png)

### 1.3 Kafka基础架构

![image.png](https://i.loli.net/2021/10/05/EyK8rliRJXWAjYs.png)

- Producer：消息生产者，就是向kafka broker发消息的客户端
- Consumer：消息消费者，向kafka broker取消息的客户端
- Consumer Group（CG）：消费者组，由多个consumer组成。**消费者组内每个消费者负责消费不同区域的数据，一个分区只能由一个组内的消费者消费；消费者组之间互不影响**。所有的消费者都属于每个消费者组，即**消费者组是逻辑上的一个订阅者**
- Broker：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
- Topic：可以理解为一个队列，**生产者和消费者面相的都是一个topic**
- Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker(即服务器上)，**一个topic可以分为多个partition**，每个partition是一个有序的队列
- Replica：副本，为保证集群中的某个节点发生故障时，**该节点的partition数据不会丢失，且kafka仍然能够继续工作**，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干的follower。
- leader：每个分区多个副本的”主“，生产者发送数据的对象，以及消费者消费数据的对象都是leader
- follower：每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的leader

## 2 kafka快速入门

### 2.1 kafka配置文件

```properties
#broker 的全局唯一编号，不能重复
broker.id=0

#删除 topic 功能使能
delete.topic.enable=true

#处理网络请求的线程数量
num.network.threads=3

#用来处理磁盘 IO 的线程数量
num.io.threads=8

#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400

#接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400

#请求套接字的缓冲区大小
socket.request.max.bytes=104857600

#kafka 运行日志存放的路径
log.dirs=/data/kafka

#topic 在当前 broker 上的分区个数
num.partitions=1

#用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1

#segment 文件保留的最长时间，超时将被删除，单位为小时，168h为一周
log.retention.hours=168

#配置连接 Zookeeper 集群地址
zookeeper.connect=192.168.10.3:2181,192.168.2.11:2181,192.168.2.12:2181
```

### 2.2 kafka集群启动

在每台节点上启动

```bash
/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
```

### 2.3 kafka命令行操作

- 查看当前服务器所有topic

  ```bash
  kafka-topics.sh --zookeeper 192.168.10.3:2181 --list
  ```

- 创建topic

  --topic topic名字

  --replication-factor 定义副本数

  --partitions 定义分区数

  ```bash
  kafka-topics.sh --zookeeper 192.168.10.3:2181 --create --replication-factor 3 --partitions 1 --topic first
  ```

- 删除topic

  需要 server.properties 中设置 delete.topic.enable=true 否则只是标记删除

  ```bash
  kafka-topics.sh --zookeeper 192.168.10.3:2181 --delete --topic first
  ```

- 发送消息

  ```bash
  kafka-console-producer.sh --broker-list 192.168.10.3:9092 --topic first
  ```

- 消费消息

  --from-beginning：会把主题中以往所有的数据都读取出来

  ```bash
  kafka-console-consumer.sh --bootstrap-server  192.168.10.3:9092 --topic first 
  ```

  ```bash
  kafka-console-consumer.sh --zookeeper 192.168.10.3:2181 --topic first
  ```

- 查看某个topic的详细

  ```bash
  kafka-topics.sh --zookeeper 192.168.10.3:2181 --describe --topic first
  ```

- 修改topic分区数

  ```bash
  kafka-topics.sh --zookeeper 192.168.10.3:2181 --alter --topic first --partitions 6
  ```


## 3.Kafka架构深入

### 3.1 Kafka工作流程及文件存储机制

#### 3.1.1 kafka工作流程

![image.png](https://i.loli.net/2021/10/05/cOL4tilnYpJyh6g.png)

kafka中的消息是以**topic**进行分类的，生产者生产消息，消费者消费消息，都是面相topic的。

topic是逻辑上的概念，而partition是物理上的概念，每个partition对应的一个log文件，该log文件中存储的就是producer生产的数据。Producer生产的数据会被不断的追加到该Log文件的尾端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自消费到了哪个offset，以便于出错恢复时，从上次的位置继续消费。

#### 3.1.2 kafka文件存储机制

![image.png](https://i.loli.net/2021/10/05/q6FGyAsDuvbR4ac.png)

由于生产者生产的消息会不断追加到log文件结尾，为防止log文件过大导致数据定位效率低下，kafka采取了**分片**和**索引**机制，将每个partition分为多个segment（一般1G为单位分片）。每个segment对应两个文件：".index"和".log"文件。这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号。例如first这个topic有三个分区，则其对应的文件夹为：first-0、first-1、first-2

```bash
total 8
-rw-r--r--. 1 kafka kafka 10485760 Oct  5 20:11 00000000000000000000.index
-rw-r--r--. 1 kafka kafka       69 Oct  5 20:14 00000000000000000000.log
-rw-r--r--. 1 kafka kafka 10485756 Oct  5 20:11 00000000000000000000.timeindex
-rw-r--r--. 1 kafka kafka        8 Oct  5 20:14 leader-epoch-checkpoint
```

index和log文件以当前segment的第一条消息的offset命名。下图为index文件和log文件的结构示意图。**”.index“文件存储大量的索引信息，”.log“文件存储大量的数据**，索引中的元数据指向对应数据文件中的message的物理偏移地址。

![image.png](https://i.loli.net/2021/10/05/RjXqPvNsFA8ZtD2.png)

### 3.2 Kafka生产者

#### 3.2.1 分区策略

1) 分区的原因

   - **方便在集群中扩展**，每个partition可以通过调整以适应它所在的机器，而一个topic又可以有多个partition组成，因此整个集群就可以适应任意大小的数据了；
   - **可以提高并发**，因为可以以partition为单位读写了。

2) 分区的原则

   我们需要将producer发送的数据封装成一个**ProducerRecord**对象

   - 指明partition的情况下，直接将指明的值直接作为partition值
   - 没有指明partition值但是有key的情况下，将key的hash值与topic的partition数进行取余得到partition值
   - 既没有partition值有没有key值得情况下，第一次调用时随机生成一个证书后（后面每次调用在这个整数上自增），将这个值与topic可用的partition总数取余得到partition的值，也就是常说的round-robin算法。

#### 3.2.2 数据的可靠性保证

为了保证producer发送的数据，能够可靠的发送到指定的topic，topic的每个partition收到producer发送的数据后，都需要向producer发送ack(acknowledgement确认收到)，如果producer收到ack，就会进行下一轮的发送，否则重新发送数据。

![image.png](https://i.loli.net/2021/10/05/dGMwphraLVcjzgf.png)

1) kafka副本同步策略

   集群中partition全部完成同步，才回发送ack，优点是：选举新的leader时，容忍n台节点的故障，需要n+1个副本，缺点是延迟高。

2) ISR

   **leader维护了一个动态的in-sync replica set(ISR)，意为和leader同步的follower集合。当ISR中的follower完成数据的同步之后，leader就会给follower发送ack。如果follower会长时间未向leader同步数据，则该follower将被踢出ISR，改时间阈值由*replica.lag.time.max.ms*参数设定。leader发生故障之后，就会从ISR中选举新的leader。**

3) ack应答机制

   对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所有没必要等ISR中的follower全部接收成功。所以kafka为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡，选择以下的配置：

   1) acks设置为0

      producer不等待broker的ack，这一操作提供了一个最低的延迟，broker一接收到还没有写入磁盘就已经返回，当broker故障时有可能会**丢失数据**

   2) acks设置为1

      producer等待broker的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将会**丢失数据**

   3) acks设置为-1

      producer等待broker的ack，partition的leader和follower全部落盘成功后才返回ack。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成**数据重复**。