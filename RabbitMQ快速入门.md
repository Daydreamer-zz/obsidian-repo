# RabbitMQ

## 1.RabbitMQ介绍

**RabbitMQ**是一个在**AMQP**(Advanced Message Queuning Protocol)基础上实现的，可服用的企业消息系统，它可以用于大型软件系统各个模块之间的高效通信，支持高并发，支持可扩展。他支持多种语言客户端，支持AJAX，持久化，用于分布式系统存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

RabbitMQ是使用Erlang编写的一个开源的消息队列，本身支持很多协议：AMQT、XMPP、SMTP、STOMP，也正是如此，使他变得十分重量级，更适合用于企业级得到开发。同时实现了一个broker架构，这意味着消息在发送给客户端时先在中心队列排队，对路由（Routing）、负载均衡或者数据持久化都有很好的支持。

## 2.RabbitMQ特点

- 可靠性
- 灵活的路由
- 扩展性
- 高可用性
- 多种协议
- 多语言客户端
- 管理界面
- 插件机制

## 3.AMQT介绍

**AMQT**，即Advanced Message Queuing Protocol ，一个提供统一消息服务的应用层标准高级消息队列协议，是应用协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端于消息中间件可传递消息，并不受客户端/中间件不同产品，不同的开发语言等条件的限制。

## 4.什么是消息队列

MQ全称Message Queue，消息队列。是医用应用程序对应用程序的通信方法。应用程序通过读写出入队列的消息来通信，而无需专用连接来链接它们。

消息传递是指程序之间通过在消息中发送数据进行通信，而不是通过直接调用彼此来通信。队列的使用除去了接收和发送应用程序同时执行的要求。

在项目中，将一些无需及时返回且耗时的操作提取出来，进行异步处理，而这种异步处理的方式大大的节省了服务器的请求响应时间，从而提高了系统的吞吐量。

## 5.RabbitMQ应用场景

对于一个大型的软件系统来说，他会有很多的组件或者说模块或者说子系统。那么这些模块之间如何通信？这和传统的IPC有很大区别。传统的IPC很多都是在单一系统上的，模块耦合性很大，不适合扩展；如果使用socket那么不同的模块的确可以部署在不同的机器上，但是还有很多问题需要解决，比如：

1) 信息的发送者和接受者如何维持这个连接，如果一方的连接中断，这期间的数据如何方式丢失。
2) 如何降低发送者和接受者的耦合度
3) 如何让Priority高的接受者先接收到数据
4) 如何做到load balance?有效均衡接受者的负载
5) 如何有效的将相关数据发送到相关的接受者，也就是说接受者subscribe不同的数据，如何做到有效filter
6) 如何保证可扩展，甚至将这个通信模块发到cluster
7) 如何保证接受者收到了完整、正确的数据

AMQT协议解决了以上问题，而RabbitMQ实现了AMQT

## 6.RabbitMQ概念

- **Broker**：简单来说就是消息队列服务器实体
- **Exchange**：消息交换机，它指定消息按什么规则，路由到哪个队列
- **Queue**：消息队列载体，每个消息都会被投入到一个或者多个队列
- **Binding**：绑定，它的作用就是把Exchange和Queue按照路由规则绑定起来
- **Routing Key**：路由关键字，exchange根据这个关键字进行消息投递
- **vhost**：虚拟主机，一个broker里可以开多个vhost，用作不同用户的权限分离
- **producer**：消息生产者，就是投递消息的程序
- **consumer**：消息消费者，就是接收消息的程序
- **channel**：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务

**RabbitMQ**从整体上来看是一个典型的生产者消费者模型，主要负责接收、存储和转发消息

## 7.RabbitMQ使用流程

AMQP模型中，消息在producer中产生，发送到MQ的exchange上，exchange根据配置的路由方式发送到相应的Queue上，Queue又将消息发送给consumer，消息从Queue到Consumer有push和pull两种方式。消息队列的使用流程大致如下：

1) 客户端连接到消息队列服务器，打开一个channel
2) 客户端声明一个exchange，并设置相关属性
3) 客户端声明一个queue，并设置相关属性
4) 客户端使用routing key，在exchange和queue之间建立好绑定关系
5) 客户端投递消息到exchange

exchange接收到消息后，就根据消息的key和已经设置的binding，进行消息路由，将消息投递到一个或多个队列里。exchange也有几个类型，完全根据key进行投递的叫做Direct交换机，例如：绑定时设置了routing key为"abc"，那么客户端提交的消息，只有设置了key为"abc"的才会投递到队列

## 8.RabbitMQ安装

### 8.1 安装Erlang环境

Erlang 24 依赖于 OpenSSL 1.1，它**在 CentOS 7 上不可用**。因此 Erlang 24 软件包**只适用于CentOS8及以上版本**

下载erlang rpm包

```bash
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v24.2.1/erlang-24.2.1-1.el8.x86_64.rpm
```

```bash
yum localinstall erlang-24.2.1-1.el8.x86_64.rpm -y
```

### 8.2 安装rabbitmq-server

下载rpm包

```bash
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.13/rabbitmq-server-3.9.13-1.el8.noarch.rpm
```

```bash
yum localinstall rabbitmq-server-3.9.13-1.el8.noarch.rpm -y
```

### 8.3 启动rabbitmq-server

```bash
systemctl start rabbitmq-server
```

### 8.4 启动rabbitmq web页面插件

```bash
rabbitmq-plugins enable rabbitmq_management
```

### 8.5 rabbitmq添加用户

添加用户

```bash
rabbitmqctl add_user admin 123456
```

设置用户角色

```bash
rabbitmqctl set_user_tags admin administrator
```

## 9.RabbitMQ常用命令

### 9.1 基本命令

- 启动监控管理器

  ```bash
  rabbitmq-plugins enable rabbitmq_management
  ```

- 关闭监控管理器

  ```bash
  rabbitmq-plugins disable rabbitmq_management
  ```

- 查看所有的队列

  ```bash
  rabbitmqctl list_queues
  ```

- 清除所有的队列

  ```bash
  rabbitmqctl reset
  ```

- 关闭应用

  ```bash
  rabbitmqctl stop_app
  ```

- 启动应用

  ```bash
  rabbitmqctl start_app
  ```

### 9.2 用户和权限设置

- 添加用户

  ```bash
  rabbitmqctl add_user <username> <password>
  ```

- 分配角色

  ```bash
  rabbitmqctl set_user_tags <username> administrator
  ```

- 新增虚拟主机

  ```bash
  rabbitmqctl add_vhost <vhost_name>
  ```

- 将虚拟主机授权给新用户，后面三个`.*`代表用户拥有配置、写、读全部权限

  ```bash
  rabbitmqctl set_permissions -p <vhost_name> <username> ".*" ".*" ".*"
  ```

### 9.3 角色说明

1) 超级管理员(**administrator**)

   可以登录管理控制台，可查看所有的信息，并且可以对用户，策略进行操作

2) 监控者(**monitoring**)

   可以登录管理控制台，同时可以查看rabbitmq节点相关信息（进程数、内存使用情况、磁盘使用情况等）

3) 策略制定者(**policymaker**)

   可以登录管理控制台，同时可以对policy进行管理。但无法查看节点的相关系信息

4) 普通管理者(**management**)

   仅可登录管理控制台，无法查看到节点信息，也无法对策略进行管理

5) 其他

   无法登录管理控制台，通常就是普通的生产者和消费者

## 10.RabbitMQ集群部署及配置

消息队列中间件RabbitMQ，一般采用集群的方式部署，主要提供消息的接收和发送，实现各微服务之间的消息异步。一下介绍RabbitMQ+HA的方式进行部署

### 10.1 集群原理

RabbitMQ是根据erlang的分布式特性(RabbitMQ底层是通过erlang架构实现的，所有rabbitmqctl会启动erlang节点，并基于erlang节点来使用erlang系统连接RabbitMQ节点，在连接的过程中需要正确的erlang cookie和节点名称，erlang节点通过交换erlang cookie以获取认证)来实现的，所有部署RabbitMQ分布式集群时要先安装erlang，并把其中一个服务的cookie复制到另外两个节点。

RabbitMQ集群中，各个RabbitMQ为对等节点，即每个节点均提供给客户端连接，进行消息的接收和发送。节点分为内存节点和磁盘节点，一般的，均应建立为磁盘节点，为了防止机器重启后的消息丢失。

RabbitMQ的Cluster集群模式一般分为两种，**普通模式**和**镜像模式**。消息队列通过RabbitMQ HA镜像队列进行消息队列的实体复制。

普通模式，以两个节点（rabbit01、rabbit02）为例来进行说明。对于Queue来说，消息实体只存在于其中一个节点rabbit01（或者rabbit02），rabbit01和rabbit02两个节点仅有相同的元数据，即队列的结构。当消息进入rabbit01节点的Queue后，consumer从rabbit02节点消费时，RabbitMQ会临时在rabbit01、rabbit02间进行消息传输，把A中的消息实体取出并经过B发送给consumer。所以consumer应尽量连接每一个节点，从中取消息。即对于同一个逻辑队列，要在多个节点建立物理Queue。否则无论consumer连rabbit01或rabbit02，出口总在rabbit01，会产生瓶颈。当raabbit01节点故障后，rabbit02节点无法获取到rabbit01节点中还未消费的消息实体。如果做了消息持久化，那么得等rabbit01节点恢复，然后才可被消费；如果没有做持久化的话，就会产生消息丢失的现象。

镜像模式下，将需要消费的队列变为镜像队列，存在于多个节点，这样就可以实现RabbitMQ的HA高可用性。作用就是消息实体会主动在镜像节点之间实现同步，而不是像普通模式那样，在consumer消费数据时临时读取，缺点就是，集群内部的同步通讯会占用大量带宽。所以在对外可靠性要求较高的场合中适用。

### 10.2 部署RabbitMQ Cluster(普通模式)

rabbitmq有3种模式，但集群模式是2种

详细如下：

- 单一模式：即单机情况下不做集群，就单独运行一个rabbitmq而已
- 普通模式
- 镜像模式

1) 拉取docker镜像

   ```bash
   docker pull rabbitmq:3.9.13-management-alpine
   ```

2) 创建docker网络

   ```bash
   docker network create rabbitmq
   ```

3) 创建持久化目录

   ```bash
   mkdir -p /data/rabbit0{1..3}  && chown -R 100:101 /data/rabbit0*
   ```

4) 启动各个容器

   rabbit01

   ```bash
   docker run -d --name rabbit01 --hostname rabbit01 --network  rabbitmq -v /data/rabbit01:/var/lib/rabbitmq -v /data/cookie/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie -p 15672:15672 -p 5672:5672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=199747 rabbitmq:3.9.13-management-alpine
   ```

   rabbit02

   ```bash
   docker run -d --name rabbit02 --hostname rabbit02 --network  rabbitmq -v /data/rabbit02:/var/lib/rabbitmq -v /data/cookie/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=199747 rabbitmq:3.9.13-management-alpine
   ```

   rabbit03

   ```bash
   docker run -d --name rabbit03 --hostname rabbit03 --network  rabbitmq -v /data/rabbit03:/var/lib/rabbitmq -v /data/cookie/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=199747 rabbitmq:3.9.13-management-alpine
   ```

5) 加入集群

   分别再rabbit02和rabbit03容器中执行

   ```bash
   rabbitmqctl stop_app
   rabbitmqctl join_cluster --ram rabbit@rabbit01
   rabbitmqctl start_app
   ```

6) web页面中查看

   ![image.png](https://s2.loli.net/2022/02/08/FTYXatvjuchrqV2.png)

### 10.3 部署RabbitMQ Cluster(镜像模式)

参考文档：https://www.rabbitmq.com/ha.html

首先镜像模式要依赖policy模块，这个模块的作用为：设置哪些exchanges或者queue的数据需要复制，同步，如何复制同步。

```bash
rabbitmqctl set_policy ha_all "^" '{"ha-mode": "all"}'
```

参数：

ha_all：策略名称，随便起名

^：匹配符，只有一个^表示匹配所有，^abc为匹配名称为abc的exchanges或者queue

ha-mode：为匹配类型，他分为3中模式：

- all：所有
- exctly：部分(需要配置ha-params参数，此参数Wieint类型，比如3，众多集群中的随机3台机器)
- nodes：指定(需要配置ha-params参数，此参数为数组类型，比如["rabbit@rabbit01",  "rabbit@rabbit02"]这样指定rabbit01和rabbit02两台机器)

## 11. 生产环境建议

### 11.1 vhost

在生产中，如果rabbitmq只为单个系统提供服务的时候，我们模式使用`/`是可以的。但是在为多个系统提供服务时，建议使用单独的vhost。

### 11.2 user

对于生产环境，建议删除默认用户(guest)，默认用户只能从localhost连接。

我们可以创建指定权限的单独用户为每个应用提供服务。对于开启权限用户来说，我们可以使用证书，和源ipd地址过滤，和身份验证。来加强安全性。

### 11.3 最大打开文件限制

在生产环境中我们可能需要调整一些系统的默认限制，以便处理大量的并发连接和队列。

需要调整ulimit即可，这里不在详细叙述。

### 11.4 内存

当rabbitmq检测到它使用的内存超过系统的40%，它将不会接收任何新的消息，这个值是由参数`vm_memory_high_watermark`来控制的，默认值是一个安全的值，修改该值需要注意，rabbitmq的至少需要128M，建议`vm_memory_high_watermark`的值为`0.4~0.66`，不要使用大于`0.7`的值

### 11.5 磁盘

磁盘默认的存储数据阈值是50MB，当低于该值的时候，将触发流量限制。50M只适用于开发环境，生产环境需要调高该值，不然容易引起由磁盘空间不足导致的节点故障，也可能导致数据丢失。

生产环境建议设置的值为：

- 建议的最小值{disk_free_limit, {mem_relative, 1.0}}

  它是基于mem_relative的值，例如在具有4G内存的主机上，那么该值的阈值就是4G，如果磁盘可用空间低于4G，所有的生产者的消息都将拒绝。在允许恢复发布之前，通常需要消费者将队列消息消费完。

- 建议的更安全值{disk_free_limit, {mem_relative, 1.5}}

  在具有4G内存的RabbitMQ节点上，如果可用磁盘空间低于6GB，则所有的新消息都将被组织，但是如果我们停止的时候rabbitmq需要存储4GB的数据到磁盘，再下一次启动的手，就只有2G空间了

- 建议的最大值{disk_free_limit, {mem_relative, 2.0}}

  这个是最安全的值，如果你的磁盘空间足够多的话，建议设置为该值。但该值容易触发警告，因为在具有4G内存的节点上，需要最低空间大于8GB磁盘空间，如果你的磁盘空间比较少的话，不建议设置为该值。

### 11.6 连接

少使用短连接，使用连接池或者长连接

### 11.7 TLS

建议尽可能使用TLS连接，使用TLS会对传输的数据加密，但是对系统的吞吐量产生很大影响。

### 11.8 更改默认端口

web页面端口，`management.listen.port`

AMQP协议端口，`listeners.tcp.default`
