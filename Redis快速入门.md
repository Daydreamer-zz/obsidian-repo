# redis缓存

## 一、redis介绍
### 1.redis简介

- redis是一个开源的使用ANSI C语言编写的Key-Value内存数据库
- 读写性能强，支持多种数据类型
- 把数据存储在内存中的高速缓存

### 2.redis特点

- 速度快
- 支持多种数据结构(string    list    hash    set    storted set)
- 持久化
- 主从复制(集群)
- 支持过期时间
- 支持事务
- 消息订阅
- 官方不支持windows

### 3.redis和memcache的对比

| 项目     | Redis                  |       memcached        |
| -------- | ---------------------- | :--------------------: |
| 过期策略 | 支持                   |          支持          |
| 数据类型 | 五种数据类型           |      单一数据类型      |
| 持久化   | 支持                   |         不支持         |
| 主从复制 | 支持                   |         不支持         |
| 虚拟内存 | 支持                   |         不支持         |
| 性能     | 强，多线程写入效果明显 | 强，单线程写入效果明显 |

### 4.Redis应用场景

- 数据缓存

  提高访问性能，使用方式与memcache相同

- 会话缓存（session cache）

  保存web会话信息（判断用户是否是登录状态）

- 排行榜/计数器

  nginx+lua+Redis计数器进行IP自动封禁

- 消息队列

  构建实时的消息系统，聊天，群聊

## 二、安装配置

### 1.安装(编译安装)

```
tar xf redis-3.2.6.tar.gz && cd redis-3.2.6/ && make && cd src/ && make install PREFIX=/application/redis
cp redis-3.2.6/redis.conf /application/redis/
```

### 2.启动

```
/application/redis/bin/redis-server /application/redis/redis.conf
```

### 3.配置文件

```
bind 192.168.1.4
protected-mode yes            #redis的安全机制
requirepass 199747            #设置登录密码，需要上一项为yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes                 #允许后台启动，改为yes
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""                    #后面跟日志路径，放引号里面
databases 16
always-show-logo yes
save 900 1                    #在900s之内，有一次操作，就保存到硬盘
save 300 10                   #300s之内，有10次操作，就保存到硬盘
save 60 10000                 #60s之内，有1w次操作，就保存到硬盘 
stop-writes-on-bgsave-error yes
rdbcompression yes            #保存本地的数据文件是否开启压缩，默认yes
rdbchecksum yes
dbfilename dump.rdb           #保存在硬盘的数据文件(持久化)
dir ./
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly no                      #日志开关,aof持久化
appendfilename "appendonly.aof"
appendfsync everysec            #默认everysec每秒同步一次，no表示操作系统进行数据缓存同步到磁盘，linux约30s，always表示每次更新操作后调用fsync将书记写入到硬盘
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-log-slower-than 10000         #慢日志查询，超过多少微秒,才认定为是慢查询
slowlog-max-len 128                   #保存多少慢日志
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

### 4.连接redis-server

```
redis-cli -h 192.168.1.4
```

## 三、常用操作

### 1.认证密码

```
192.168.1.4:6379> auth 199747
OK
```

### 2.设置键值，并获取

```
192.168.1.4:6379> set shz 21
OK
192.168.1.4:6379> get shz
"21"
```

### 3.获取系统中所有key

```
192.168.1.4:6379> KEYS *
1) "oldboy"
2) "shz"
```

### 4.获取当前所有配置

```
192.168.1.4:6379> CONFIG GET *
.............
```

### 5.变更运行配置(只修改当前内存中的配置，重启失效)

```
192.168.1.4:6379> CONFIG SET loglevel 'notice'
OK
```

## 四、redis数据类型

### 1.redis数据存储

-  内存
- 硬盘 ：数据文件.rdb    日志文件.aof

### 2.持久化

- RDB持久化可以在指定的时间间隔内生成数据集的时间点快照
- AOF持久化记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集。AOF文件的命令全部以Redis协议的格式来保存，新命令会被追加到文件的末尾。Redis还可以在后台对AOF文件进行重写，使得AOF文件的体积不会超出保存数据集状态所需的实际大小
- Redis还可以同时使用AOF和RDB持久化。在这种情况下，当Redis重启时，它会优先使用AOF文件来还原数据集，因为AOF文件保存的数据集通常比RDB文件所保存的数据集更加完整。
- 你甚至可以关闭持久化功能，让数据只在服务器运行时存在

## 五、核心实践，操作

### 1.redis保存的数据类型

- 字符串（REDIS_STRING）
- 列表（REDIS_LIST）
- 有序集合（REDIS_ZSET）

- 哈希表（REDIS_HASH）
- 集合（REDIS_SET）

### 2.常规操作

- KEYS *查看KEY
- DEL删除给定的一个过多个KEY
- EXISTS检查是否存在
- EXPIRE设置生存时间
- TTL以秒为单位返回过期时间
- DUMP RETORE序列化与反序列化
- EXPIRE PTTL PERSIST以毫秒为单位
- RENAME 变更KEY名
- SORT键值排序，有序数字时报错
- TYPE返回键所存储的类型

### 3.数据类型详解

#### 字符串（string）

- SET name shz

- GET name

- 一个键最大能存储512M


- append可以将value追加到key原来值得末尾
- Mget mset同时设置一个或者多个键值对
- STRLEN返回字符串长度
- INCR DECR将值增或减1
- INRBY DECRBY 减去指定量
- DECRBY count 20

#### hash（哈希）

- redis hash是一个键值对的集合
- redis hash是一个string类型的field和value的映射表
- hash特别适合用于存储对象
- 每个hash可以存储2^32-1个键值对



- HSET HGET 设置返回单个值
- HMSET HMGET 设置和获取多个值
- HGETALL 返回key的所有键值
- HEXSITS HLEN
- HLEYS HVALS 获取所有字段或值
- HDEL 删除key中的一个或者多个指定值

```
192.168.1.4:6379> HSET user1 name shz
(integer) 1
192.168.1.4:6379> hset user1 age 21
(integer) 1
192.168.1.4:6379> HGET user1 name
"shz"
192.168.1.4:6379> HGET user1 age
"21"

192.168.1.4:6379> HGETALL user1
1) "name"
2) "shz"
3) "age"
4) "21"

192.168.1.4:6379> HMGET user1 name age
1) "shz"
2) "21"
```

#### 列表

- Redis列表是简单的字符串列表
- 按照插入顺序排序每个
- list可以以存储2^32-1个键值对



- LPUSH 将一个或多个值插入到列表头部
- RPUSH 将一个或多个值插入到列表尾部
- LPOP/RPOP 一处表头/尾的元素
- LLEN返回列表长度
- LRANGE  返回指定的元素
- LREM greet 2 morning 删除前两个morning
- LREM greet -1 morning 删除后一个morning
- LREM greet 0 hello 删除所有hello
- Lindex  返回列表key中下标为index的元素
- LSET key index value 将列表key下标位index的元素的值为value
- LINSERT 插入数据位于某元素之前或之后(LINSERT key BEFORE XXX value)

```
插入表格
192.168.1.4:6379> LPUSH list1 name age sex
(integer) 3

查看类型
192.168.1.4:6379> type list1
list

查看表格数据
192.168.1.4:6379> LRANGE list1 0 10
1) "sex"
2) "age"
3) "name"

往前面出入数据
192.168.1.4:6379> LPUSH list1 phone
(integer) 4
192.168.1.4:6379> LRANGE list1 0 19
1) "phone"
2) "sex"
3) "age"
4) "name"

取出数据(从上面删除),且有返回值
192.168.1.4:6379> RPOP list1
"name"
192.168.1.4:6379> LRANGE list1 0 10
1) "phone"
2) "sex"
3) "age"

往后面插入数据
192.168.1.4:6379> RPUSH list1 from
(integer) 4
192.168.1.4:6379> LRANGE list1 0 10
1) "phone"
2) "sex"
3) "age"
4) "from"

取出数据（从下面删除），且有返回值
192.168.1.4:6379> RPOP list1
"from"
192.168.1.4:6379> LRANGE list1 0 10
1) "phone"
2) "sex"
3) "age"

直接删除，没有返回值
192.168.1.4:6379> LREM list1 1 phone
(integer) 1
192.168.1.4:6379> LRANGE list1 0 10
1) "sex"
2) "age"

设定一个值(根据前面的数标判断)
192.168.1.4:6379> LSET list1 1 name
OK
192.168.1.4:6379> LRANGE list1 0 10
1) "sex"
2) "name"

往指定位置插入数据
192.168.1.4:6379> LINSERT list1 AFTER name from
(integer) 3
192.168.1.4:6379> LRANGE list1 0 10
1) "sex"
2) "name"
3) "from"

```

## 六、redis高级应用

### 1. 生产消费模型

### 2.消息模式

- 发布消息通常有两种模式：队列模式(queuing)和发布-订阅模式(publish-subscibe)。队列模式中，consumers可以同时从服务端读取消息，每个消息只被其中一个consumer读到
- 发布-订阅模式中消息被广播到所有的consumer中，topic中的消息将被分发到组中的一个成员中。同一组中的consumer可以在不同的程序中，也可以在不同的机器上

### 3.Redis发布订阅

- Redis发布订阅(pub/sun)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接受消息。
- Redis客户端可以订阅任意数量的频道

### 4.订阅发布实例

复制3个ssh会话

```
192.168.1.4:6379> PUBLISH mq1 "i love u guys"
(integer) 2
```

```
192.168.1.4:6379> SUBSCRIBE mq1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "mq1"
3) (integer) 1
1) "message"
2) "mq1"
3) "i love u guys"
```

### 5.Redis事务

- Redis事务可以一次执行多个命令

  事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序执行。事务在执行的过程中，不会被其他客户端发送的命令请求打断

  原子性：事务中的命令要么全部被执行，要么全部不执行

- 执行过程：

  开始事务

  命令入队

  执行事务

- 事务命令

  DISCARD ：取消事务，放弃执行事务块内的所有命令

  EXEC ：执行所有事务块内的命令

  MULTI：标记一个事务块的开始

  UNWATCH：取消WATCH命令对所有key的监视

  WATCH key ：监视一个或多个key，如果在事务执行之前这个key被其他命令所改动，那么事务将被打断



  模拟转账(multi标记一个事务，确保转账扣钱和增加是同时操作的)

  这个数据类型是 有序集合

  ```
  
  192.168.1.4:6379> ZADD salary 3000 shz 5000 ll
  (integer) 2
  
  开启队列
  192.168.1.4:6379> MULTI
  OK
  
  将下面命令添加到队列
  192.168.1.4:6379> ZINCRBY salary 1000 shz
  QUEUED
  192.168.1.4:6379> ZINCRBY salary -1000 ll
  QUEUED
  
  执行命令
   192.168.1.4:6379> EXEC
  1) "4000"
  2) "4000"
  
  查看数据
  192.168.1.4:6379> ZRANGE salary 0 -1 withscores
  1) "ll"
  2) "4000"
  3) "shz"
  4) "4000"
  ```

  ### 6.服务器命令

  - INFO   服务器信息
  - CLIENT LIST    客户端列表，当前登录的客户，通过redis-cli连接的
  - FLUSHALL    清空所有数据
  - FLUSHDB  清空当前库
  - MONITOR  监控实时指令
  - SHUTDOWN  关闭服务器redis-server
  - SAVE  保存数据
  - SLAVEOF host port  主从的配置
  - SLAVEOF NO ONE  关闭主从复制
  - SYNC  主从同步
  - ROLE    返回主从角色

### 7.慢日志查询

在配置文件中开启并配置

```
192.168.1.4:6379> CONFIG GET slow*
1) "slowlog-log-slower-than"
2) "10000"
3) "slowlog-max-len"
4) "128"
```

### 8.数据备份

- CONFIG GET dir   获取当前目录
- Save备份（无持久化策略时），生成时在Redis当前目录中
- 恢复时只需将dump.rdb放入redis当前目录

## 七、主从复制

### 1.主从同步方式

redis的主从同步有两种方式：全同步或者部分同步

主从刚刚连接的时候，会进行全同步；在全同步结束后，进行部分同步。当然，如果有需要，slave在任何时候都可以发起全同步。

redis的策略是，无论如何，首先尝试进行部分同步，如果不成功哦，要求进行全同步，并启动BGSAVE，BGSAVE结束后，传输rdb文件；如果成功，允许从机进行部分同步，并传输积压空间中的数据。

在从服务器上执行

### 2.redis主从同步复制原理

- 从服务器向主服务器发送SYNC命令
- 接收到SYNC命令的主服务器会调用BGSAVE命令，创建一个RDB文件，并使用缓冲区记录下接下来执行的所有写命令。
- 当主服务器执行完BGSAVE命令是，它会向从服务器发送RDB文件，而从服务器则会接受并载入这个文件。
- 主服务器将缓冲区存储的所有写命令发送给从服务器执行。

### 3.redis命令的传播

在主服务器完成同步之后，主服务器每执行一个命令，它都会将被执行的写命令发送给从服务器执行，这个操作被成为“命令传播”(command propagate)

命令传播是一个持续的过程：只要复制仍在继续，命令传播就会一直进行，使得主从服务器的状态可以一直保持一致。

### 4.redis复制的SYNC和PSYNC

在redis2.8版本之前，断线之后重连的从服务器总是要执行一次完整同步(full resynchronization) 操作。

从redis2.8开始，redis使用PSYNC命令代替了SYNC命令。PSYNC比起SYNC的最大改进在于PSYNC实现了部分同步(partial resync)，其特性为：在主从服务器断线并且重新连接的时候，只要条件允许，PSYNC可以让主服务器只向从服务器同步断线期间缺失的数据，而不用重新向从服务器同步整个数据库。

### 5.redis复制的一致性问题

在读写分离的情境下，客户端向主服务器发送写命令，主服务器在执行这个写命令之后，向客户端返回回复，并将这个命令传播给从服务器。

接到回复的客户端继续向从服务器发送读命令，并且因为网络状态的原因，客户端的GET命令比主服务器传播的SET命令更快到达了服务器。

因为从服务器键值还未被更新，所以客户端在从服务器读取到的将是一个错误的数据。

### 6.redis复制的安全性提升

主服务器只在有至少N个从服务器的情况下，才执行写操作从REDIS2.8开始，为了保证数据的安全性，可以通过配置，让主服务器只在有至少N个当前已连接从服务器的情况下，才执行写命令。

不过，因为redis使用异步复制，所以主服务器发送的写数据并不一定会被从服务器接受到，因此数据丢失的可能性仍然是存在的。

通过以下两个参数保证数据的安全

```ini
min-slaves-to-write <number of slaves>
min-slaces-max-lag  <numer of seconds>
```

### 7.redis主从复制实践

```
192.168.1.5:6379> SLAVEOF 192.168.1.4 6379
OK

查看
192.168.1.5:6379> INFO Replication
# Replication
role:slave
master_host:192.168.1.4
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:1
master_link_down_since_seconds:1543653264
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:5a217d0959762c50b2ad0e289124da1e35b01c6e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

```
从升级为master
192.168.1.5:6379> SLAVEOF no one
OK
192.168.1.5:6379> INFO Replication
# Replication
role:master
connected_slaves:0
master_replid:a3a481c40cda3452b297b3040a153ace36abd5c0
master_replid2:5a217d0959762c50b2ad0e289124da1e35b01c6e
master_repl_offset:0
second_repl_offset:1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

## 八、Redis 高可用(redis sentnel)

### 1.简介

- redis-sentinel时Redis官方推荐的好可用性的解决方案，当用Redis做master-slave的高可用方案时，假如master宕机了，Redis本身（包括它的很多客户端）都没有实现自动进行主备切换，儿Redis-sentinel本身也是一个独立运行的进程，他能监控多个master-slave集群，发现master宕机后进行自动切换。

### 2.功能

- 监控：sentinel会不断地检查你的主服务器和从服务器是否运作正常
- 提醒：当被监控的某个Redis服务器出现问题时，sentinel可以通过api向管理员或者其他应用程序发送通知
- 自动故障迁移：当一个主服务器不能正常工作时，sentinel会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效的主服务器额其他从服务器改为新的主服务器；当客户端试图连接失效的主服务时，集群也会向客户端返回新主服务器的地址，使得集群可以使用新主服务器代替失效服务器

### 3.配置和启动

配置文件

```
port 26379
daemonize no
pidfile /var/run/redis-sentinel.pid
logfile ""
dir /tmp
sentinel monitor mymaster 127.0.0.1 6379 1  #指定监控master的bind,1指需要多少台sentinel同意才能切换master
sentinel down-after-milliseconds mymaster 30000  #超过30000毫秒后认为主机宕机
sentinel parallel-syncs mymaster 1  #指定failover过程中，能够被sentinel并行配置的从节点数量
sentinel failover-timeout mymaster 180000 #当主从切换多久后认为主从切换失败
sentinel deny-scripts-reconfig yes
```

启动

```
redis-sentinel /application/redis/sentinel.conf
```

## 九、redis集群

### 1.简介

- redis集群是一个可以在多个Redis节点之间进行数据共享的设施
- Redis集群不支持那些需要同时处理多个键的Redis命令，因为执行这些命令需要在多个Redis节点之间移动数据，并且在高负载的情况下，这些命令将降低Redis集群的性能，并导致不可预测的行为
- Redis集群通过分区来提供一定程度的可用性：即使集群中有一部分节点失效或者无法进行通讯，集群也可以继续处理命令请求
- 将数据自动划分到多个节点的能力
- 当集群中的一部分节点失效或者无法进行通讯是，仍然可以继续处理命令请求的能力

### 2.redis集群数据共享

- Redis集群使用数据分片而非一致性哈希来实现：一个Redis集群包含16384个哈希槽，数据库中的每个键都属于这16384个哈希槽的其中一个，集群使用公式CRC(key)%16384来计算键key属于哪个槽，其中CRC16(key)语句用于计算键(key)语句用于计算键key的CRC16校验

如：

- 节点A负责处理0号至5500号哈希槽
- 节点B负责处理5501至11000号哈希槽
- 节点C负责处理11001至16384号哈希槽

### 3.集群的复制

- 为了使得集群在一部分节点下线或者无法与集群的大多数节点进行通讯的情况下，仍然可以正常运作，Redis集群节点使用了主从复制的功能：集群中的每个节点都有1个至N个复制品，其中一个复制品为主节点，而其余的N-1个复制品为从节点。
- 在之前举例的节点A、B、C的例子中，如果节点B下线了，那么集群将无法正常运行，以为集群找不到节点来处理5501到11000号的哈希槽
- 假如在创建集群的时候（或者至少在节点B下线之前），我们为主节点B添加了从节点B1，那么当主节点B下线的时候，集群就会将B1设置为新的主节点，并让它代替下线的主节点B，继续处理5501-11000号的哈希槽，这样集群就不会因为主节点B的下线而无法正常运作了
- 不过如果节点B和B1都下线的话，Redis集群还是会停止工作

### 4.运行机制

- 所有的Redis节点彼此互联，内部使用二进制协议优化传输速度和带宽
- 节点的失败是通过集群中超过半数的master几点检测失效时才失效
- 客户端与Redis节点直连，不需要任何中间proxy层，客户端不需要连接集群所有节点，连接集群的任何一个可用节点即可
- 把所有的物理节点映射到[0-16384]slot上，cluster负责维护node<>slot<>key

### 5.配置集群

准备6个实例的redis

每个redis.conf修改如下内容

```
cluster-enabled yes

cluster-config-file /usr/local/redis/etc/nodes-6379.conf

cluster-node-timeout 5000
```

创建集群

```bash
redis-cli --cluster create 172.18.0.2:6379 172.18.0.3:6379 172.18.0.4:6379 172.18.0.5:6379 172.18.0.6:6379 172.18.0.7:6379 --cluster-replicas 1
# create 表示创建一个集群
# --cluster-replicas 1  表示为集群中的每一个主节点指定一个从节点
```

​    **注意：**

如果配置项`cluster-enabled`的值不为yes，则执行时会报错[ERR] Node x.x.x.x:6379 is not configured as a cluster node.，这时候必须将`cluster-enabled`的值改为yes，然后重启redis-server进程之后才可以继续创建redis集群

### 6.查看集群中的节点

```bash
[root@node01 ~]# redis-cli -h 172.18.0.2 -c cluster nodes
622e9423f7b8f43c6b89b0c935a9b7b088d7f1d3 172.18.0.2:6379@16379 myself,master - 0 1644205322000 1 connected 0-5460
4a9c2b26cd630b32751201fb0d8cab6f6e37395f 172.18.0.3:6379@16379 master - 0 1644205322000 2 connected 5461-10922
041f3c5111efa96ebc93fd5edc9f1906888f9bda 172.18.0.6:6379@16379 slave 4a9c2b26cd630b32751201fb0d8cab6f6e37395f 0 1644205323226 2 connected
33e5ea17521cd1be8c8f9dca344c958c7dcd4afe 172.18.0.7:6379@16379 slave 0dee5839946feff7cb2ec4832d7a1dac2274fa49 0 1644205322710 3 connected
b60d9aa34941aeb1446ad906cd4c9a2d397b1ffa 172.18.0.5:6379@16379 slave 622e9423f7b8f43c6b89b0c935a9b7b088d7f1d3 0 1644205322000 1 connected
0dee5839946feff7cb2ec4832d7a1dac2274fa49 172.18.0.4:6379@16379 master - 0 1644205322200 3 connected 10923-16383
```

**字段从左到右依次为**

节点ID | IP地址和端口 | 节点角色标志 | 最后发送ping时间 | 最后收到pong时间 | 连接状态 | 节点负责处理的hash slot

集群可以自动识别出ip/slot的变化，并通过Gossip（最终一致性，分布式服务数据同步算法）协议广播给其他节点知道。Gossip也成”病毒感染算法“、”谣言传播算法“

### 7.验证集群

```bash
[root@node01 ~]# redis-cli -h 172.18.0.2 -c 
172.18.0.2:6379> set name redis
-> Redirected to slot [5798] located at 172.18.0.3:6379
OK
172.18.0.3:6379> 
[root@node01 ~]# redis-cli -h 172.18.0.2 -c 
172.18.0.2:6379> get name
-> Redirected to slot [5798] located at 172.18.0.3:6379
"redis"
```

### 8.检查节点状态

```bash
[root@node01 ~]# redis-cli --cluster check 172.18.0.2:6379
172.18.0.2:6379 (622e9423...) -> 0 keys | 5461 slots | 1 slaves.
172.18.0.3:6379 (4a9c2b26...) -> 1 keys | 5462 slots | 1 slaves.
172.18.0.4:6379 (0dee5839...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.18.0.2:6379)
M: 622e9423f7b8f43c6b89b0c935a9b7b088d7f1d3 172.18.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 4a9c2b26cd630b32751201fb0d8cab6f6e37395f 172.18.0.3:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 041f3c5111efa96ebc93fd5edc9f1906888f9bda 172.18.0.6:6379
   slots: (0 slots) slave
   replicates 4a9c2b26cd630b32751201fb0d8cab6f6e37395f
S: 33e5ea17521cd1be8c8f9dca344c958c7dcd4afe 172.18.0.7:6379
   slots: (0 slots) slave
   replicates 0dee5839946feff7cb2ec4832d7a1dac2274fa49
S: b60d9aa34941aeb1446ad906cd4c9a2d397b1ffa 172.18.0.5:6379
   slots: (0 slots) slave
   replicates 622e9423f7b8f43c6b89b0c935a9b7b088d7f1d3
M: 0dee5839946feff7cb2ec4832d7a1dac2274fa49 172.18.0.4:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### 9.查看集群信息

```bash
[root@node01 ~]# redis-cli -h 172.18.0.2 
172.18.0.2:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:4647
cluster_stats_messages_pong_sent:4576
cluster_stats_messages_sent:9223
cluster_stats_messages_ping_received:4571
cluster_stats_messages_pong_received:4647
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:9223
```

### 10.redis集群添加密码

登录每个节点执行

```bash
[root@node01 ~]# redis-cli -h 172.18.0.2 -c 
172.18.0.2:6379> CONFIG SET masterauth 123456 
OK
172.18.0.2:6379> CONFIG SET requirepass 123456
OK
```

### 11.redis cluster操作命令行

#### 11.1 集群cluster

```bash
CLUSTER INFO # 打印集群信息
CLUSTER NODES # 列出集群当前已知的所有节点(node)，以及这些节点的信息
```

#### 11.2 节点node

```bash
CLUSTER MEET <ip> <port>  # 将ip和port所指定的节点添加到集群中，让它成为集群的一份子
CLUSTER FORGET <node_id> # 从集群中移除node_id指定的节点
CLUSTER REPLICATE <node_id> # 将当前节点设置为node_id指定的节点的从节点
CLUSTER SAVECONFIG # 将节点的配置文件保存到磁盘
```

#### 11.3 槽 slot

```bash
CLUSTER ADDSLOTS <slot> [slot ...] # 将一个或者多个槽(slot)指派(assign)给当前节点
CLUSTER DELSLOTS <slot> [slot ...] # 移除一个或者多个槽对当前节点的指派
CLUSTER FLUSHSLOTS # 移除指派给当前节点的所有槽， 让当前节点变成一个没有指派任何槽的节点
CLUSTER SETSLOT <slot> NODE <node_id> # 将槽slot指派给node_id指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽，然后在进行指派
CLUSTER SETSLOT <slot> MIGRATING <node_id> # 将本节点的槽slot迁移到node_id指定的节点中
CLUSTER SETSLOT <slot> IMPORTING <node_id> # 从node_id指定的节点中导入槽slot到本节点
CLUSTER SETSLOT <slot> STABLE # 取消对槽slot的导入(import)或者(migrate)
```

#### 11.4 键key

```bash
CLUSTER KEYSLOT <key> # 计算键key应该被放置在哪个槽上
CLUSTER COUNTKEYSINSLOT <slot> # 返回槽slot目前包含的键值对数量
CLUSTER GETKEYINSLOT <slot> <count> # 返回count个slot中的键
```

### 12.redis集群管理

#### 12.1 集群添加节点

额外启动另外两个redis实例：172.18.0.8:6379、172.18.0.9:6379

添加一个主节点到集群

```bash
# add-node参数后第二个参数是原来已经在集群中的任意节点
redis-cli --cluster add-node 172.18.0.8:6379 172.18.0.2:6379
```

添加一个从节点到集群

```bash
# --cluster-slave参数表示加入一个从节点到集群
redis-cli --cluster add-node 172.18.0.9:6379 172.18.0.2:6379 --cluster-slave
```

#### 12.2 为新节点分配槽

```bash
[root@node01 ~]# redis-cli --cluster reshard  172.18.0.2:6379
....
....
....

# 根据提示选择要迁移的slot数量
How many slots do you want to move (from 1 to 16384)? 4096

# 哪个节点接收迁移出的slot
What is the receiving node ID? 95eb9b0978ab4d44fdcf58cba7f3c23e5c234232

# 选择slot来源
# all表示所有的master重新分配
# 或者数据要提取slot的master节点id，最后用done结束
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:
```

#### 12.3 删除一个主节点

在删除master之前首先需要使用reshard迁移被移除的master节点全部slot，然后再删除该节点（目前只能把被删除的的master的slot迁移到一个节点上）

```bash
redis-cli --cluster reshard 172.18.0.8:6379
```

```bash
redis-cli --cluster del-node 172.18.0.2:6379 95eb9b0978ab4d44fdcf58cba7f3c23e5c234232
```

#### 12.4 删除一个从节点

```bash
redis-cli --cluster del-node 172.18.0.2:6379 6674dcf9312c3a05cd5a8beb7cd6adb08c065e81
```

