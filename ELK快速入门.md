# ELK日志分析集群搭建

## 1.简介

如果你没有听说过Elastic Stack，那你一定听说过ELK，实际上ELK是三款软件的简称，分别是Elasticsearch、 Logstash、Kibana组成，在发展的过程中，又有新成员Beats的加入，所以就形成了Elastic Stack。所以说，ELK是旧的称呼，Elastic Stack是新的名字。

![image.png](https://i.loli.net/2021/10/12/6kn1lFXIM7JKQCg.png)

## 2.ELK组成

全系的Elastic Stack技术栈包括：

![image.png](https://i.loli.net/2021/10/12/7UpkroKjFRhf43t.png)

- Elasticsearch

  Elasticsearch 基于java，是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

- Logstash

  Logstash 基于java，是一个开源的用于收集,分析和存储日志的工具。

- Kibana

  Kibana 基于nodejs，也是一个开源和免费的工具，Kibana可以为 Logstash 和 ElasticSearch 提供的日志分析友好的Web 界面，可以汇总、分析和搜索重要数据日志

- Beats

  Beats是elastic公司开源的一款采集系统监控数据的代理agent，是在被监控服务器上以客户端形式运行的数据收集器的统称，可以直接把数据发送给Elasticsearch或者通过Logstash发送给Elasticsearch，然后进行后续的数据分析活动。Beats由如下组成:

  - Packetbeat：是一个网络数据包分析器，用于监控、收集网络流量信息，Packetbeat嗅探服务器之间的流量，解析应用层协议，并关联到消息的处理，其支 持ICMP (v4 and v6)、DNS、HTTP、Mysql、PostgreSQL、Redis、MongoDB、Memcache等协议；
  - Filebeat：用于监控、收集服务器日志文件，其已取代 logstash forwarder；
  - Metricbeat：可定期获取外部系统的监控指标信息，其可以监控、收集 Apache、HAProxy、MongoDB MySQL、Nginx、PostgreSQL、Redis、System、Zookeeper等服务；

> Beats和Logstash其实都可以进行数据的采集，但是目前主流的是使用Beats进行数据采集，然后使用 Logstash进行数据的分割处理等，早期没有Beats的时候，使用的就是Logstash进行数据的采集。

## 3.Elasticsearch部分

### 3.1  简介

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

我们建立一个网站或应用程序，并要添加搜索功能，但是想要完成搜索工作的创建是非常困难的。我们希望搜索解决方案要运行速度快，我们希望能有一个零配置和一个完全免费的搜索模式，我们希望能够简单地使用JSON通过HTTP来索引数据，我们希望我们的搜索服务器始终可用，我们希望能够从一台开始并扩展到数百台，我们要实时搜索，我们要简单的多租户，我们希望建立一个云的解决方案。因此我们利用Elasticsearch来解决所有这些问题及可能出现的更多其它问题。

ElasticSearch是Elastic Stack的核心，同时Elasticsearch 是一个分布式、RESTful风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为Elastic Stack的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

### 3.2 安装Elasticsearch

#### 内核参数优化

```ini
# 设定系统最大打开的文件描述符数，建议修改为655350或者更高
fs.file-max=655350

# 用于限制一个进程可以拥有的虚拟内存大小，建议修改为262144或者更高
vm.max_map_count = 262144  

net.core.somaxconn = 32768
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_max_tw_buckets = 400000
```

#### 文件和进程打开限制配置

删除系统原有默认配置

```bash
rm -f /etc/security/limits.d/20-nproc.conf # 注意el8系统上没有这个
```

```bash
cat << EOF >> /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

#### 安装jdk

解压jdk二进制包

```bash
tar xf jdk-8u291-linux-x64.tar.gz -C /usr/local/
```

配置环境变量

```bash
export JAVA_HOME=/usr/local/jdk1.8.0_291
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib/tools.jar:${JAVA_HOME}/lib/dt.jar:${CLASSPATH}
export PATH=${JAVA_HOME}/bin:${PATH}
```

#### 安装Elasticsearch

下载Elasticsearch二进制包

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.8.19.tar.gz
tar xf elasticsearch-6.8.19.tar.gz -C /usr/local/
```

创建软链接

```bash
ln -s /usr/local/elasticsearch-6.8.19 /usr/local/elasticsearch
```

创建Elasticsearch systemd启动脚本

```bash
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target


[Service]
Environment=JAVA_HOME=/usr/local/jdk1.8.0_291
Environment=ES_HOME=/usr/local/elasticsearch
WorkingDirectory=/usr/local/elasticsearch
User=elastic
Group=elastic
ExecStart=/usr/local/elasticsearch/bin/elasticsearch
StandardOutput=journal
StandardError=inherit
LimitNOFILE=65536
LimitNPROC=2048
LimitAS=infinity
LimitFSIZE=infinity
TimeoutStopSec=0
KillSignal=SIGTERM
KillMode=process
SendSIGKILL=no
SuccessExitStatus=143
LimitMEMLOCK=infinity  #es锁住内存必须添加这项配置

[Install]
WantedBy=multi-user.target
```

#### 配置Elasticsearch

编辑Elasticsearch配置文件，/usr/local/elasticsearch/config/elasticsearch.yml

```yaml
cluster.name: my-es-cluster  # 集群的名称，名称相同的主机就是处于同一个集群
node.name: node01    #集群情况下，当前node的名字，每个node应该不一样
path.data: /data/es/data #数据目录
path.logs: /data/es/logs #日志目录
bootstrap.memory_lock: true #服务启动时即锁定足够大的内存，提高效率
network.host: 192.168.2.13 #监听的地址
http.port: 9200 #客户端访问端口
discovery.zen.ping.unicast.hosts: ["192.168.2.13", "192.168.2.14", "192.168.2.15"] #组播地址
http.cors.enabled: true  #允许跨域
http.cors.allow-origin: "*" #允许跨域的域名
```

创建Elasticsearch用户及目录

```bash
useradd elastic -s /sbin/nologin -M
```

```bash
mkdir -p /data/es/{data,logs} && chown -R elastic:elastic /data/es
```

#### 启动Elasticsearch

```bash
systemctl enable elasticsearch.service --now
```

#### Elasticsearch Master选举日志

1) Node添加到集群的node本机日志

   ```
   [2021-10-06T14:27:10,775][INFO ][o.e.c.s.ClusterService   ] [node02] detected_master {node01}{aBYrtnQ-Q6GOB1qkt11wng}{CEbECx4IRfaeGI6uXsRZmQ}{192.168.2.13}{192.168.2.13:9300}, added {{node01}{aBYrtnQ-Q6GOB1qkt11wng}{CEbECx4IRfaeGI6uXsRZmQ}{192.168.2.13}{192.168.2.13:9300},}, reason: zen-disco-receive(from master [master {node01}{aBYrtnQ-Q6GOB1qkt11wng}{CEbECx4IRfaeGI6uXsRZmQ}{192.168.2.13}{192.168.2.13:9300} committed version [7]])
   ```

2. 添加node时master的日志

   ```
   [2021-10-06T14:27:10,750][INFO ][o.e.c.s.ClusterService   ] [node01] added {{node02}{pgEUIABORRatnA_vTCwcDQ}{2G_CTGw-RciBy8grgluk0g}{192.168.2.14}{192.168.2.14:9300},}, reason: zen-disco-node-join[{node02}{pgEUIABORRatnA_vTCwcDQ}{2G_CTGw-RciBy8grgluk0g}{192.168.2.14}{192.168.2.14:9300}]
   ```

#### 查看集群健康状态

访问如下接口

```bash
curl -XGET 'http://192.168.2.13:9200/_cluster/health?pretty=true'
```

返回的"status"："green" 为正常， "yellow" 为不正常，"red"   为主分片丢失



#### JVM参数优化

JVM内存具体要根据node要存储的数据量来估算，为了保证性能，在内存和数据量有一个建议的比例：

像一般的日志文件，1G内存能存储48G-96G数据，官方要求JVM最大堆内存不能超过32G

其次就是主分片的数量，单个控制在30-50G

假设总数据量为1TB，3个node节点，1个副本，那么实际要存储的大小为2T，因为有一个副本的存在，2TB/3 = 700G，然后每个节点需要预留20%的空间，意味着每个node要存储大约850G的数据，按照内存和数据的比率计算：
850GB / 48 = 17GB，小于31G，因为31*48 = 1.4TB及每个node可以存储1.4TB数据，所以3个节点足够，850G / 30 = 29个主分片，因为要尽量控制主分片的大小为30G

### 3.3  ElasticSearch中的基本概念

####   索引

- 索引（index）是Elasticsearch对逻辑数据的逻辑存储，所以它可以分为更小的部分。
- 可以把索引看成关系型数据库的表，索引的结构是为快速有效的全文索引准备的，特别是它不存储原始值。
- Elasticsearch可以把索引存放在一台机器或者分散在多台服务器上，每个索引有一或多个分片（shard），每个分片可以有多个副本（replica）。

#### 文档

- 存储在Elasticsearch中的主要实体叫文档（document）。用关系型数据库来类比的话，一个文档相当于数据库表中的一行记录。
- Elasticsearch和MongoDB中的文档类似，都可以有不同的结构，但Elasticsearch的文档中，相同字段必须有相同类型。
- 文档由多个字段组成，每个字段可能多次出现在一个文档里，这样的字段叫多值字段（multivalued）。 每个字段的类型，可以是文本、数值、日期等。字段类型也可以是复杂类型，一个字段包含其他子文档或者数 组

#### 映射

所有文档写进索引之前都会先进行分析，如何将输入的文本分割为词条、哪些词条又会被过滤，这种行为叫做 映射（mapping）。一般由用户自己定义规则。

#### 文档类型

- 在Elasticsearch中，一个索引对象可以存储很多不同用途的对象。例如，一个博客应用程序可以保存文章和评 论。
- 每个文档可以有不同的结构。
- 不同的文档类型不能为相同的属性设置不同的类型。例如，在同一索引中的所有文档类型中，一个叫title的字段必须具有相同的类型。

### 3.4 RESTful API

在Elasticsearch中，提供了功能丰富的RESTful API的操作，包括基本的CRUD、创建索引、删除索引等操作。

#### 创建非结构化索引

在Lucene中，创建索引是需要定义字段名称以及字段的类型的，在Elasticsearch中提供了非结构化的索引，就是不需要创建索引结构，即可写入数据到索引中，实际上在Elasticsearch底层会进行结构化操作，此操作对用户是透明的。

#### 创建空索引

```json
PUT /test
{
    "settings": {
        "index": {
        "number_of_shards": "2", #分片数，不指定默认为5
        "number_of_replicas": "0" #副本数，不指定默认为1
        }
    }
}
```

#### 插入数据

> URL规则： POST /{索引}/{类型}/{id}

```json
POST /test/user/1001
#数据
{
  "id":1001,
  "name":"张三",
  "age":20,
  "sex":"男"
}
```

#### 更新数据

在Elasticsearch中，文档数据是不为修改的，但是可以通过覆盖的方式进行更新

```json
PUT /test/user/1001
{
  "id":1001,
  "name":"张三",
  "age":21,
  "sex":"女"
}
```

局部更新

```json
#注意：这里多了_update标识
POST /test/user/1001/_update
{
  "doc":{
    "age":23
  }
}
```

#### 删除索引

在Elasticsearch中，删除文档数据，只需要发起DELETE请求即可，不用额外的参数

```
DELETE 1 /test/user/1001
```

> 删除一个文档也不会立即从磁盘上移除，它只是被标记成已删除。Elasticsearch将会在你之后添加更多索引的时候才会在后台进行删除内容的清理。【相当于批量操作】

#### 搜索数据

根据id搜索数据

```json
GET /test/user/1001
#返回的数据如下
{
    "_index": "test",
    "_type": "user",
    "_id": "1001",
    "_version": 1,
    "_seq_no": 0,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "id": 1001,
        "name": "张三",
        "age": 20,
        "sex": "男"
    }
}
```

#### DSL搜索

Elasticsearch提供丰富且灵活的查询语言叫做DSL查询(Query DSL),它允许你构建更加复杂、强大的查询。 DSL(Domain Specific Language特定领域语言)以JSON请求体的形式出现。

```
POST /test/user/_search
#请求体
{
    "query" : {
        "match" : {   #match只是查询的一种
        	"age" : 20
        }
    }
}
```

实现：查询年龄大于30岁的男性用户。

现有数据：

![image.png](https://i.loli.net/2021/10/12/1GfXxIjVq8sOh4a.png)

```
POST /test/user/_search
#请求数据
{
    "query": {
        "bool": {
            "filter": {
                "range": {
                    "age": {
                        "gt": 30
                    }
                }
            },
            "must": {
                "match": {
                    "sex": "男"
                }
            }
        }
    }
}
```

返回结果为

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.6931472,
        "hits": [
            {
                "_index": "test",
                "_type": "user",
                "_id": "lM4tcHwBSNNMk5tuOjY2",
                "_score": 0.6931472,
                "_source": {
                    "id": 1002,
                    "name": "王五",
                    "age": 31,
                    "sex": "男"
                }
            }
        ]
    }
}
```

##### 全文搜索

```
POST /test/user/_search
#请求数据
{
    "query": {
        "match": {
        	"name": "张三 李四"
        }
    }
}
```

高亮显示，只需要在添加一个 highlight即可

```
POST /test/user/_search
#请求数据
{
    "query": {
        "match": {
            "name": "张三 李四"
        }
    },
    "highlight": {
        "fields": {
            "name": {}
        }
    }
}
```

##### 聚合

在Elasticsearch中，支持聚合操作，类似SQL中的group by操作。

```
POST /test/user/_search
{
    "aggs": {
        "all_interests": {
            "terms": {
                "field": "age"
            }
        }
    }
}
```

结果如下，通过年龄进行聚合

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 1.0,
        "hits": [
            {
                "_index": "test",
                "_type": "user",
                "_id": "1001",
                "_score": 1.0,
                "_source": {
                    "id": 1001,
                    "name": "张三",
                    "age": 20,
                    "sex": "男"
                }
            },
            {
                "_index": "test",
                "_type": "user",
                "_id": "lM4tcHwBSNNMk5tuOjY2",
                "_score": 1.0,
                "_source": {
                    "id": 1002,
                    "name": "王五",
                    "age": 31,
                    "sex": "男"
                }
            },
            {
                "_index": "test",
                "_type": "user",
                "_id": "lc4tcHwBSNNMk5tuiTbr",
                "_score": 1.0,
                "_source": {
                    "id": 1003,
                    "name": "小六",
                    "age": 38,
                    "sex": "女"
                }
            },
            {
                "_index": "test",
                "_type": "user",
                "_id": "1002",
                "_score": 1.0,
                "_source": {
                    "id": 1002,
                    "name": "李四",
                    "age": 21,
                    "sex": "女"
                }
            }
        ]
    },
    "aggregations": {
        "all_interests": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": 20,
                    "doc_count": 1
                },
                {
                    "key": 21,
                    "doc_count": 1
                },
                {
                    "key": 31,
                    "doc_count": 1
                },
                {
                    "key": 38,
                    "doc_count": 1
                }
            ]
        }
    }
}
```

### 3.5  ElasticSearch核心详解

####  文档

在Elasticsearch中，文档以JSON格式进行存储，可以是复杂的结构，如：

```json
{
    "_index": "haoke",
    "_type": "user",
    "_id": "1005",
    "_version": 1,
    "_score": 1,
    "_source": {
        "id": 1005,
        "name": "孙七",
        "age": 37,
        "sex": "女",
        "card": {
            "card_number": "123456789"
         }
    }
}
```

其中，card是一个复杂对象，嵌套的Card对象

####  元数据（metadata）

一个文档不只有数据。它还包含了元数据(metadata)——关于文档的信息。三个必须的元数据节点是：

| 节点     | 说明               |
| -------- | ------------------ |
| `_index` | 文档存储的地方     |
| `_type`  | 文档代表的对象的类 |
| `_id`    | 文档的唯一标识     |

#####  _index

索引(index)类似于关系型数据库里的“数据库”——它是我们存储和索引关联数据的地方。

> 提示：事实上，我们的数据被存储和索引在分片(shards)中，索引只是一个把一个或多个分片分组在一起的逻辑空间。然而，这只是一些内部细节——我们的程序完全不用关心分片。对于我们的程序而言，文档存储在索引(index)中。剩下的细节由Elasticsearch关心既可。

##### _type

在应用中，我们使用对象表示一些“事物”，例如一个用户、一篇博客、一个评论，或者一封邮件。每个对象都属于一个类(class)，这个类定义了属性或与对象关联的数据。user 类的对象可能包含姓名、性别、年龄和Email地址。 在关系型数据库中，我们经常将相同类的对象存储在一个表里，因为它们有着相同的结构。同理，在Elasticsearch 中，我们使用相同类型(type)的文档表示相同的“事物”，因为他们的数据结构也是相同的。

每个类型(type)都有自己的映射(mapping)或者结构定义，就像传统数据库表中的列一样。所有类型下的文档被存储在同一个索引下，但是类型的映射(mapping)会告诉Elasticsearch不同的文档如何被索引。

_type 的名字可以是大写或小写，不能包含下划线或逗号。

##### _id

id仅仅是一个字符串，它与_index 和_type 组合时，就可以在Elasticsearch中唯一标识一个文档。当创建一个文 档，你可以自定义_id ，也可以让Elasticsearch帮你自动生成（32位长度）

### 3.6  查询响应

#### pretty

可以在查询url后面添加pretty参数，使得返回的json更易查看。

![image.png](https://i.loli.net/2021/10/12/zCQF3yrsDjdbuxm.png)

#### 指定响应字段

在响应的数据中，如果我们不需要全部的字段，可以指定某些需要的字段进行返回。通过添加 _source

```
GET /test/user/1001?_source=id,name
{
    "_index": "test",
    "_type": "user",
    "_id": "1001",
    "_version": 1,
    "_seq_no": 0,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "张三",
        "id": 1001
    }
}
```

如不需要返回元数据，仅仅返回原始数据，可以这样：

```
GET /test/user/1001/_source

{
    "id": 1001,
    "name": "张三",
    "age": 20,
    "sex": "男"
}
```

还可以这样：

```
GET /test/user/1001/_source?_source=id,name

{
    "name": "张三",
    "id": 1001
}
```

#### 判断文档是否存在

如果我们只需要判断文档是否存在，而不是查询文档内容，那么可以这样：

```
HEAD /test/user/1001
```

### 3.7 批量操作

有些情况下可以通过批量操作以减少网络请求。如：批量查询、批量插入数据。

#### 批量查询

```
POST /test/user/_mget
{
    "ids": [
        "1001",
        "1002"
    ]
}
```

结果为

```json
{
    "docs": [
        {
            "_index": "test",
            "_type": "user",
            "_id": "1001",
            "_version": 1,
            "_seq_no": 0,
            "_primary_term": 1,
            "found": true,
            "_source": {
                "id": 1001,
                "name": "张三",
                "age": 20,
                "sex": "男"
            }
        },
        {
            "_index": "test",
            "_type": "user",
            "_id": "1002",
            "_version": 1,
            "_seq_no": 0,
            "_primary_term": 1,
            "found": true,
            "_source": {
                "id": 1002,
                "name": "李四",
                "age": 21,
                "sex": "女"
            }
        }
    ]
}
```

如果，某一条数据不存在，不影响整体响应，需要通过found的值进行判断是否查询到数据。

```bash
POST /test/user/_mget
{
    "ids": [
        "1001",
        "1003"
    ]
}
```

```json
{
    "docs": [
        {
            "_index": "test",
            "_type": "user",
            "_id": "1001",
            "_version": 1,
            "_seq_no": 0,
            "_primary_term": 1,
            "found": true,
            "_source": {
                "id": 1001,
                "name": "张三",
                "age": 20,
                "sex": "男"
            }
        },
        {
            "_index": "test",
            "_type": "user",
            "_id": "1003",
            "found": false  #这里是个false说明1003 id的数据不存在
        }
    ]
}
```

#### _bulk操作

在Elasticsearch中，支持批量的插入、修改、删除操作，都是通过_bulk的api完成的。

请求格式如下：（请求格式不同寻常）

```
{ action: { metadata }}
{ request body }
{ action: { metadata }}
{ request body }
...
```

批量插入

```json
POST /test/user/_bulk
{"create":{"_index":"test","_type":"user","_id":2001}}
{"id":2001,"name":"name1","age": 20,"sex": "男"}
{"create":{"_index":"test","_type":"user","_id":2002}}
{"id":2002,"name":"name2","age": 20,"sex": "男"}
{"create":{"_index":"test","_type":"user","_id":2003}}
{"id":2003,"name":"name3","age": 20,"sex": "男"}

```

> 确保最后以\n换行结束，即最后一行有个空行

批量删除

```
POST /test/user/_bulk
{"delete":{"_index":"test","_type":"user","_id":2001}}
{"delete":{"_index":"test","_type":"user","_id":2002}}
{"delete":{"_index":"test","_type":"user","_id":2003}}

```

> 确保最后以\n换行结束，即最后一行有个空行

### 3.8 分页

和SQL使用LIMIT 关键字返回只有一页的结果一样，Elasticsearch接受from 和size 参数：

- size: 结果数，默认10
- from: 跳过开始的结果数，默认0

如果你想每页显示5个结果，页码从1到3，那请求如下：

```
GET /test/user/_search?size=5
GET /test/user/_search?size=5&from=5
GET /test/user/_search?size=5&from=10
```

### 3.9 映射

前面我们创建的索引以及插入数据，都是由Elasticsearch进行自动判断类型，有些时候我们是需要进行明确字段类型的，否则，自动判断的类型和实际需求是不相符的。

自动判断的规则如下：

| JSON type                          | Field type  |
| ---------------------------------- | ----------- |
| Boolean:`true` or `false`          | `"boolean"` |
| Whole number: `123`                | `"long"`    |
| Floating point: `123.45`           | `"double"`  |
| String, valid date: `"2021-10-12"` | `"date"`    |
| String: `"foo bar"`                | `"string"`  |

Elasticsearch中支持的类型如下：

| 类型           | 表示的数据类型                     |
| -------------- | ---------------------------------- |
| String         | `string`, `text`, `keyword`        |
| Whole number   | `byte`, `short`, `integer`, `long` |
| Floating point | `float`, `double`                  |
| Boolean        | `boolean`                          |
| Date           | `date`                             |

- string类型在ElasticSearch 旧版本中使用较多，从ElasticSearch 5.x开始不再支持string，由text和 keyword类型替代。
- text 类型，当一个字段是要被全文搜索的，比如Email内容、产品描述，应该使用text类型。设置text类型 以后，字段内容会被分析，在生成倒排索引以前，字符串会被分析器分成一个一个词项。text类型的字段 不用于排序，很少用于聚合。
- keyword类型适用于索引结构化的字段，比如email地址、主机名、状态码和标签。如果字段需要进行过 滤(比如查找已发布博客中status属性为published的文章)、排序、聚合。keyword类型的字段只能通过精 确值搜索到。

#### 创建明确类型的索引

> 如果你要像之前旧版版本一样兼容自定义 type ,需要将 include_type_name=true 携带

```
PUT /test1?include_type_name=true

{
    "settings": {
        "index": {
            "number_of_shards": "5",
            "number_of_replicas": "1"
        }
    },
    "mappings": {
        "person": {
            "properties": {
                "name": {
                    "type": "text"
                },
                "age": {
                    "type": "integer"
                },
                "mail": {
                    "type": "keyword"
                },
                "hobby": {
                    "type": "text"
                }
            }
        }
    }
}
```

查看映射

```
GET /test1/_mapping
```

```json
{
    "test1": {
        "mappings": {
            "person": {
                "properties": {
                    "age": {
                        "type": "integer"
                    },
                    "hobby": {
                        "type": "text"
                    },
                    "mail": {
                        "type": "keyword"
                    },
                    "name": {
                        "type": "text"
                    }
                }
            }
        }
    }
}
```

插入测试数据

```
POST /test1/_bulk

{"index":{"_index":"test1","_type":"person"}}
{"name":"张三","age": 20,"mail": "111@qq.com","hobby":"羽毛球、乒乓球、足球"}
{"index":{"_index":"test1","_type":"person"}}
{"name":"李四","age": 21,"mail": "222@qq.com","hobby":"羽毛球、乒乓球、足球、篮球"}
{"index":{"_index":"test1","_type":"person"}}
{"name":"王五","age": 22,"mail": "333@qq.com","hobby":"羽毛球、篮球、游泳、听音乐"}
{"index":{"_index":"test1","_type":"person"}}
{"name":"赵六","age": 23,"mail": "444@qq.com","hobby":"跑步、游泳"}
{"index":{"_index":"test1","_type":"person"}}
{"name":"孙七","age": 24,"mail": "555@qq.com","hobby":"听音乐、看电影"}

```

### 3.10 结构化查询

#### term查询

term 主要用于精确匹配哪些值，比如数字，日期，布尔值或 not_analyzed 的字符串(未经分析的文本数据类型)：

```
POST /test1/person/_search
{
    "query": {
        "term": {
            "age": 20
        }
    }
}
```

#### terms查询

terms 跟 term 有点类似，但 terms 允许指定多个匹配条件。 如果某个字段指定了多个值，那么文档需要一起去 做匹配：

```
POST /test1/person/_search
{
    "query": {
        "terms": {
            "age": [
                20,
                21
            ]
        }
    }
}
```

#### range查询

range 过滤允许我们按照指定范围查找一批数据：

范围操作符包含：

- gt : 大于
- gte: 大于等于
- lt : 小于
- lte: 小于等于

```
POST /test1/person/_search

{
    "query": {
        "range": {
            "age": {
                "gte": 20,
                "lte": 22
            }
        }
    }
}
```

#### exists查询

exists 查询可以用于查找文档中是否包含指定字段或没有某个字段，类似于SQL语句中的IS_NULL 条件

```
POST /test1/person/_search

{
    "query": {
        "exists": {
        	"field": "card"
        }
    }
}
```

#### match查询

match 查询是一个标准查询，不管你需要全文本查询还是精确查询基本上都要用到它。

如果你使用 match 查询一个全文本字段，它会在真正查询之前用分析器先分析match 一下查询字符

```json
{
    "match": {
    	"tweet": "About Search"
    }
}
```

如果用match 下指定了一个确切值，在遇到数字，日期，布尔值或者not_analyzed 的字符串时，它将为你搜索你 给定的值：

```json
{ "match": { "age": 26 }}
{ "match": { "date": "2014-09-01" }}
{ "match": { "public": true }}
{ "match": { "tag": "full_text" }}
```

#### bool查询

bool 查询可以用来合并多个条件查询结果的布尔逻辑，它包含一下操作符：

- must : 多个查询条件的完全匹配,相当于 and 。
- must_not : 多个查询条件的相反匹配，相当于 not 。
- should : 至少有一个查询条件匹配, 相当于 or 。

这些参数可以分别继承一个查询条件或者一个查询条件的数组：

```json
{
    "bool": {
        "must": {
            "term": {
                "folder": "inbox"
            }
        },
        "must_not": {
            "term": {
                "tag": "spam"
            }
        },
        "should": [
            {
                "term": {
                    "starred": true
                }
            },
            {
                "term": {
                    "unread": true
                }
            }
        ]
    }
}
```

### 3.11 过滤查询

前面讲过结构化查询，Elasticsearch也支持过滤查询，如term、range、match等。

示例：查询年龄为20岁的用户。

```
POST /test1/person/_search

{
    "query": {
        "bool": {
            "filter": {
                "term": {
                    "age": 20
                }
            }
        }
    }
}
```

### 3.12 中文分词

#### 分词概念

分词就是指将一个文本转化成一系列单词的过程，也叫文本分析，在Elasticsearch中称之为Analysis。

举例：我是中国人 --> 我/是/中国人

#### 分词api

指定分词器进行分词

```
POST /_analyze

{
    "analyzer" : "standard",
    "text": "Hello World"
}
```

在结果中不仅可以看出分词的结果，还返回了该词在文本中的位置

```json
{
    "tokens": [
        {
            "token": "hello",
            "start_offset": 0,
            "end_offset": 5,
            "type": "<ALPHANUM>",
            "position": 0
        },
        {
            "token": "world",
            "start_offset": 6,
            "end_offset": 11,
            "type": "<ALPHANUM>",
            "position": 1
        }
    ]
}
```



指定索引分词

```
POST /test1/_analyze

{
    "analyzer" : "standard",
    "field": "hobby",
    "text": "Listen to Music"
}
```

#### 中文分词

中文分词的难点在于，在汉语中没有明显的词汇分界点，如在英语中，空格可以作为分隔符，如果分隔不正确就会造成歧义。如：

- 我/爱/炒肉丝
- 我/爱/炒/肉丝

常用中文分词器，IK、jieba、THULAC等，推荐使用IK分词器。

IK Analyzer是一个开源的，基于java语言开发的轻量级的中文分词工具包。从2006年12月推出1.0版开始，IKAnalyzer已经推出了3个大版本。最初，它是以开源项目Luence为应用主体的，结合词典分词和文法分析算法的中文分词组件。新版本的IK Analyzer 3.0则发展为面向Java的公用分词组件，独立于Lucene项目，同时提供了对Lucene的默认优化实现。

采用了特有的“正向迭代最细粒度切分算法“，具有80万字/秒的高速处理能力 采用了多子处理器分析模式，支持：英文字母（IP地址、Email、URL）、数字（日期，常用中文数量词，罗马数字，科学计数法），中文词汇（姓名、地名处理）等分词处理。 优化的词典存储，更小的内存占用。

IK分词器 Elasticsearch插件地址：https://github.com/medcl/elasticsearch-analysis-ik

#### 安装IK分词器

下载对应es版本的分词器

```bash
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.8.19/elasticsearch-analysis-ik-6.8.19.zip
```

在elasticsearch的plugins下建立ik目录

```bash
mkdir -p ${ES_HOME}/plugins/ik
```

在新建的ik目录解压ik分词器安装包

```bash
unzip elasticsearch-analysis-ik-6.8.19.zip -d ${ES_HOME}/plugins/ik/
chown -R elastic:elastic ${ES_HOME}
```

重启elasticsearch

```bash
systemctl restart elasticsearch.service
```

#### 测试中文分词

```
POST /_analyze

{
    "analyzer": "ik_max_word",
    "text": "我是中国人"
}
```

返回结果：

```json
{
    "tokens": [
        {
            "token": "我",
            "start_offset": 0,
            "end_offset": 1,
            "type": "CN_CHAR",
            "position": 0
        },
        {
            "token": "是",
            "start_offset": 1,
            "end_offset": 2,
            "type": "CN_CHAR",
            "position": 1
        },
        {
            "token": "中国人",
            "start_offset": 2,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 2
        },
        {
            "token": "中国",
            "start_offset": 2,
            "end_offset": 4,
            "type": "CN_WORD",
            "position": 3
        },
        {
            "token": "国人",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 4
        }
    ]
}
```

### 3.13 ElasticSearch集群

#### 集群节点

ELasticsearch的集群是由多个节点组成的，通过cluster.name设置集群名称，并且用于区分其它的集群，每个节点通过node.name指定节点的名称。

在Elasticsearch中，节点的类型主要有4种：

- master节点
  - 配置文件中node.master属性为true(默认为true)，就有资格被选为master节点。master节点用于控制整个集群的操作。比如创建或删除索引，管理其它非master节点等。
- data节点
  - 配置文件中node.data属性为true(默认为true)，就有资格被设置成data节点。data节点主要用于执行数据相关的操作。比如文档的CRUD。
- 客户端节点
  - 配置文件中node.master属性和node.data属性均为false。
  - 该节点不能作为master节点，也不能作为data节点。
  - 可以作为客户端节点，用于响应用户的请求，把请求转发到其他节点
- 部落节点
  - 当一个节点配置tribe.*的时候，它是一个特殊的客户端，它可以连接多个集群，在所有连接的集群上执行 搜索和其他操作。

#### 搭建集群

集群搭建参考 3.2

查看集群：

![image.png](https://i.loli.net/2021/10/12/r132XaqFiK5MP9y.png)

创建索引：

![image.png](https://i.loli.net/2021/10/12/S9pvZ5Ew7LKCzqx.png)



查看分片：

![image.png](https://i.loli.net/2021/10/12/TcuwVNh4KDrOZUo.png)

查询集群状态：/_cluster/health 响应：

```json
{
    "cluster_name": "my-es-cluster",
    "status": "green",
    "timed_out": false,
    "number_of_nodes": 3,
    "number_of_data_nodes": 3,
    "active_primary_shards": 5,
    "active_shards": 10,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100.0
}
```

集群状态三种颜色

| 颜色     | 意义                                         |
| -------- | -------------------------------------------- |
| `green`  | 所有的主分片和副本分片都可用                 |
| `yellow` | 所有的主分片可用，但不是所有的副本分片都可用 |
| `red`    | 不是所有的主分片和副本分片都可用             |

#### 集群分片和副本

为了将数据添加到Elasticsearch，我们需要索引(index)——一个存储关联数据的地方。实际上，索引只是一个用来指向一个或多个分片(shards)的“逻辑命名空间(logical namespace)”.

- 一个分片(shard)是一个最小级别“工作单元(worker unit)”,它只是保存了索引中所有数据的一部分。
- 我们需要知道是分片就是一个Lucene实例，并且它本身就是一个完整的搜索引擎。应用程序不会和它直接通信。
- 分片可以是主分片(primary shard)或者是复制分片(replica shard)。
- 索引中的每个文档属于一个单独的主分片，所以主分片的数量决定了索引最多能存储多少数据。
- 复制分片只是主分片的一个副本，它可以防止硬件故障导致的数据丢失，同时可以提供读请求，比如搜索或者从别的shard取回文档。
- 当索引创建完成的时候，主分片的数量就固定了，但是复制分片的数量可以随时调整。

#### 集群故障转移

##### 测试data节点故障

将其中一个角色为data的节点关闭，这里演示关闭node02节点

当前集群状态为黄色，表示主节点可用，副本节点不完全可用，过一段时间观察，发现节点列表中看不到node02，副本节点分配到了node01和node03，集群状态恢复到绿色。

![image.png](https://i.loli.net/2021/10/12/GSAkHnFZm3hDxo9.png)

![image.png](https://i.loli.net/2021/10/12/zw73VxjW6MdATly.png)

重新启动node02节点后可以看到，node02恢复后，重新加入了集群，并且重新分配了节点信息。

![image.png](https://i.loli.net/2021/10/12/kGfvnOlRKBrtyAH.png)

##### 测试master节点故障

关闭node01节点，也就是现在的master节点

从结果中可以看出，集群对master进行了重新选举，选择node03为master。并且集群状态变成黄色。 等待一段时间后，集群状态从黄色变为了绿色：

![image.png](https://i.loli.net/2021/10/12/VkfpOwMryuKeXvW.png)

![image.png](https://i.loli.net/2021/10/12/bWgG2CxvTqAFcpI.png)

重新启动node01节点

重启之后，发现node01可以正常加入到集群中，集群状态依然为绿色：

![image.png](https://i.loli.net/2021/10/12/CztUdeoVnxDlpvW.png)

##### **特别说明**

如果在配置文件中discovery.zen.minimum_master_nodes设置的不是N/2+1时，会出现脑裂问题，之前宕机 的主节点恢复后不会加入到集群。

**举例：**

- 如果有10个节点，都是data node，也是master的候选节点。则quorum=10/2+1=6
- 如果有3个master候选节点，100个数据节点。则quorum=3/2+1=2
- 如果有2个节点，都是data node，也是master的候选节点。则quorum=2/2+1=2（有问题）

如果其中一个节点挂了，那么master的候选节点只有一个，无法满足quorum数量。即无法选举出master。此时只能将quorum设置成1，但是设置为1有可能出现脑裂。

总结：一般es集群的节点至少要有3个，quorum设置为2

#### 分布式文档

##### 路由

一个集群保存文档时，会采用计算的方式来确定存储到哪个节点，计算公式如下：

```mathematica
shard = hash(routing) % number_of_primary_shards

# hash   算法保证数据均匀分散在分片中
# routing  是一个关键参数，默认是文档的id，也可以自定义
# number_of_primary_shards  主分片个数

# 注意：该算法与主分片相关，一旦确定后便不能更改主分片了
# 因为一旦修改主分片后，share的计算就完全不一样了
```

其中：

- routing值是一个任意字符串，它默认是_id但也可以自定义
- 这个routing字符串通过哈希函数生成一个数字，然后除以主切片的数量得到一个余数（remainder），余数的范围永远是0到`number_of_primary_shards - 1`，这个数字就是特定文档所在的分片，这也是为什么创建了主分片后，不能修改的原因。

##### 文档的写操作

新建、索引和删除请求都是写操作，它们必须在主分片上成功完成才能复制分片上。

![image.png](https://i.loli.net/2021/10/12/5HrlQxuz8MEn3OF.png)

主分片和复制分片上成功新建、索引和删除一个文档必要的顺序步骤为：

1) 客户端给Node1分送新建、索引和删除请求
2) 节点使用文档的_id确认文档属于分片0，它应该转发请求到Node3，因为分片0位于这个节点上
3) Node3在主分片上执行请求，如果成功，它转发请求到响应的位于node1和node2的额复制节点上，当所有的复制节点报告成功，node3报告成功到请求的节点，请求的节点再报告给客户端。

客户端接收到成功响应的时候，文档的行已经被应用于主分片和所有的复制分片，修改即为生效。

##### 搜索文档

文档能够从主分片或任意一个复制分片被检索。

![image.png](https://i.loli.net/2021/10/12/Jrqt1m3TKZgabjH.png)

在主分片或者复制分片上检索一个文档的步骤：

1) 客户端给Node1发送get请求
2) 节点使用文档的_id确定文档属于分片0，分片0对应的复制分片或者主分片在三个节点都有，此时，它转发请求到Node2
3) Node2返回文档给Node1然后返回给客户端。对于读请求，为了平衡负载，请求节点会为每个请求选择不同的分片----它会循环所有分片副本。可能的情况是，一个被索引的文档已经存在于主分片缺还没来得及同步到复制分片上。这时复制分片会报告文档未找到，主分片会成功返回文档。一旦索引请求成功返回给用户，文档则在主分片和复制分片都是可用的。

#### 全文搜索

对于全文搜索而言，文档可能分散在各个节点上，那么在分布式的情况下，如何搜索文档呢？

搜索，分为两个阶段：

- 搜索（query）
- 取回（fetch）

##### 搜索(query)

![image.png](https://i.loli.net/2021/10/12/euMrDP6fsk9TzUt.png)

查询阶段分为如下三步：

1) 客户端发送一个search(搜索)的请求给Node3，Node3创建一个长度为from+size的空优先级队列
2) Node3转发这个搜索请求到索引中的每个分片的主分片和副本分片。每个分片在本地执行这个查询并将结果返回到一个大小为from+size的有序本地优先队列里去。
3) 每个分片返回doceument的ID和它的优先队列里所有的document的排序给协调节点Node3，Node3把这些值合并到自己的优先队列里产生全局排序结果。

##### 取回(fetch)

![image.png](https://i.loli.net/2021/10/12/tXL6VOiopqfNSMw.png)

取回阶段由以下步骤构成：

1) 协调节点辨别出哪个document需要取回，并且向相关分片发出GET请求
2) 每个分片加载document并且根据需要丰富(enrich)它们，然后再将document返回协调节点
3) 一旦所有的document都被取回，协调节点会将结果返回给客户端。

### 3.14 elasticsearch集群认证

#### 启用elastic TLS

生成es证书，集群其他节点需要推送过去

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert -out /etc/elasticsearch/elastic-certificates.p12 -pass ""
chown elasticsearch:elasticsearch /etc/elasticsearch/elastic-certificates.p12
chmod 600 /etc/elasticsearch/elastic-certificates.p12
```

配置`/etc/elasticsearch/elasticsearch.yml`，加入如下内容，集群其他节点也应该配置

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

重启所有es节点

```bash
systemctl restart elasticsearch.service
```

#### 配置elastic集群密码

```bash
# auto为自动生成，需要手动记录密码信息
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto

# interactive为交互式手动生成每个用户的密码
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

## 4.Logstash部分

### 4.1 介绍

Logstash是一个开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到最喜欢的存储库中（我们的存储库当然是ElasticSearch）

![image.png](https://i.loli.net/2021/10/12/D6kexCa4UldmqWH.png)

我们回到我们ElasticStack的架构图，可以看到Logstash是充当数据处理的需求的，当我们的数据需要处理的时候，会将它发送到Logstash进行处理，否则直接送到ElasticSearch中

![image.png](https://i.loli.net/2021/10/12/7UpkroKjFRhf43t.png)

### 4.2 用途

Logstash可以处理各种各样的输入，从文档，图表中=，数据库中，然后处理完后，发送到指定位置

![image.png](https://i.loli.net/2021/10/12/9PuaoTNpYS3DL8H.png)



### 4.3 安装部署

可以安装在任意需要收集日志的节点

下载二进制包

```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.8.19.tar.gz
```

解压二进制安装包

```bash
tar xf logstash-6.8.19.tar.gz -C /usr/local/
```

### 4.3 测试使用

```bash
bin/logstash -e 'input{ stdin{} } output{ stdout { codec => rubydebug } }'
```

键入什么输出什么说明成功

测试输出到文件

```bash
bin/logstash -e 'input{ stdin{} } output{ file { path => "/tmp/log-%{+YYYY.MM.dd}.log" gzip => true }}'
```

测试输出到elasticsearch

```bash
bin/logstash -e 'input{ stdin{} } output{ elasticsearch{ hosts => ["192.168.2.4:9200"] index => "logstash-test-%{+YYYY.MM.dd}" }}'
```

### 4.4 配置文件

logstash的配置文件分为三部分：

```
input { #输入
stdin { ... } #标准输入
}
filter { #过滤，对数据进行分割、截取等处理
...
}
output { #输出
stdout { ... } #标准输出
}
```

#### 输入

- 采集各种样式、大小和来源的数据，数据往往以各种各样的形式，或分散或集中地存在于很多系统中。
- Logstash 支持各种输入选择 ，可以在同一时间从众多常用来源捕捉事件。能够以连续的流式传输方式，轻松地从您的日志、指标、Web 应用、数据存储以及各种 AWS 服务采集数据。

![image.png](https://i.loli.net/2021/10/12/ExdmAGDYg7y4i5a.png)

#### 过滤

- 实时解析和转换数据
- 数据从源传输到存储库的过程中，Logstash 过滤器能够解析各个事件，识别已命名的字段以构建结构，并将它们转换成通用格式，以便更轻松、更快速地分析和实现商业价值。

![image.png](https://i.loli.net/2021/10/12/B91sycP3whrQfST.png)

#### 输出

Logstash 提供众多输出选择，您可以将数据发送到您要指定的地方，并且能够灵活地解锁众多下游用例。

![image.png](https://i.loli.net/2021/10/12/FLfypnoAdIQvrT6.png)

### 4.5 读取自定义日志

```ruby
input{
    file{
        path => "/var/log/messages"
        type => "system-log"
        start_position => "beginning"
    }
}
filter{
    
}
output{
    elasticsearch{
        hosts => ["192.168.2.10:9200"]
        index => "system-log-%{+YYYY.MM}"
    }
}
```

### 4.6 logstash的高级应用

#### if判断输出

logstash的if判断写入不同的标签(从而实现一个配置文件收集不同的位置，并写入不同标签)

通过type定义多个，然后在用if判断输出到不同的index

如下实例

```ruby
input{
    file{
        path => "/var/log/message"
        type => "system-log"
        start_position => "beginning"
    }
    file{
        path => "/data/logs/elk-cluster.log"
        type => "es-logs"
    }
}
output{
    if [type] == "system-log" {
        elasticsearch {
            hosts => "[192.168.2.10:9200]"
            index => "system-log-%{+YYYY.MM}"
        }
    }
    if [type] == "es-logs" {
        elasticsearch {
            hosts => "[192.168.2.4:9200]"
            index => "es-logs-%{+YYYY.MM}"
        }
    }
}
```

注意：如果日志文件没有更新的话，logstash可能是无法收集，原因是/var/lib/logstash/plugins/inputs/file/.sincedb.xxxx 文件(是隐藏的)记录了日志文件的大和inode，也记录了logstash收集日志文件的位置

所以在head中你删除了此项目，凡是`.sincedb.xxxx`没删除，导致无法继续收集日志，也无法重新从文件开头收集 

#### ***filter插件(重点)***

数据从源传输到存储的过程中， Logstash 的 filter过滤器能够解析各个事件，识别已命名的字段结构，并将它们转换成通用格式，以便更轻松、更快速地分析和实现商业价值；

- 利用 Grok 从非结构化数据中派生出结构
- 利用 geoip 从 IP 地址分析出地理坐标
- 利用 useragent 从 请求中分析操作系统、设备类型

nginx日志匹配实例

```grok
input {
  http {
    port => 5656
  }
}

filter {
  grok {
    match => {
      # 对应的nginx标准日志格式
      # log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
      #                 '$status $body_bytes_sent "$http_referer" '
      #                 '"$http_user_agent" "$http_x_forwarded_for"';
      "message" => '%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:response_code} (?:%{NUMBER:bytes}|-) (?:%{QS:referrer}|-) %{QS:useragent} \"(?:%{IP:xff_clientip}|-)\"'
    }
  }
  
  mutate {
    # 替换字段中内容
    gsub => [
        "referrer", '"', "",
        "xff_clientip", '"', ""
    ]
  }
  
  # 提取日志中的时间作为@timestamp字段
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }
  
  # grok匹配的ip地址使用groip插件获取地理位置
  groip {
    source => "clientip"
  }
  
  # grok匹配的ua字段使用useragent插件解析出浏览器、设备名等信息
  useragent => {
    source => "useragent"
    target => "useragent"
  }
  
  
  mutate {
    # 去除无用字段
    remove_field => ["message", "ecs", "input", "tags"]
    
    # 转换字段的格式，方便后续在kibana中进行计算处理
    convert => ["bytes", "integer"]
    
    # 加入自定义字段，这里是index名称
    add_field => {"[@metadata][target_index]" => "nginx-access-%{+YYYY.MM.dd}"} 
  }
  
}

output {
  stdout {
    codec =>rubydebug
  }
}
```

mysql5.7慢日志匹配规则(需要在filebeat合并多行卫一个每个慢日志事件)

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    # 把换行替换卫空格
    gsub => ["message", "\n", " "]
  }
  
  grok {
    match => {
      "message" => "(?m)^# Time:.*\s+# User@Host: %{USER:User}\[%{USER-2:User}\] @ (?:(?<Clienthost>\S*) )?\[(?:%{IP:Client_IP})?\]  Id: %{NUMBER:Thread_id} # Query_time: %{NUMBER:Query_Time}  Lock_time: %{NUMBER:Lock_Time} Rows_sent: %{NUMBER:Rows_Sent}  Rows_examined: %{NUMBER:Rows_Examined} SET timestamp=%{NUMBER:timestamp}; \s*(?<Query>(?<Action>\w+)\s+.*)"
    }
  }
  
  date {
    match => ["timestamp", "UNIX", "YYYY-MM-dd HH:mm:ss"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }
  
  mutate {
    remove_field => ["message", "input", "timestamp"]
    convert => ["Thread_id", "integer"]
    convert => ["Rows_Sent", "integer"]
    convert => ["Rows_Examined", "integer"]
    convert => ["Query_Time", "float"]
    convert => ["Lock_Time", "float"]
    add_field => {"[@metadata][target_index]" => "mysql-logstash-%{+YYYY.MM.dd}"}
  }
}

output {
  stdout {
    stdout =>rubydebug
  }
}
```

#### 收集java应用日志

logstash默认将日志文件的一行当做一个事件，并分开发送，但是，在遇到如java日志多行为一个日志事件时：就需要用到codec插件

logstash的路径

一个事件--->input--->codec--->filter--->codec--->output

如果将识别多行日志的插件放到filter中，则所有的input日志都会生效，这显然不合理；所以需要将插件放到需要识别多行的`input codec`中  `codec => multiline`

编写方案：由于日志文件有时间戳，所以用"["区分，即匹配到一个"["就为一个事件

```
vim /etc/logstash/conf.d/codec.conf
```

```ruby
input{
    stdin{
        codec => multiline {
            pattern => "^\["   #正则匹配，开头为[的
            negate => true     #规定匹配到为真
            what => "previous"
        }
    }
}
output{
    stdout{
        codec => rubydebug
    }
}
```

经过测试，只要遇到中括号就把"["以上的行合并为一个事件，满足要求

所有把以上整合进file模块

```ruby
file{
    path => "/var/log/elasticsearch/mues.log"
    type => "es-log"
    start_position => "beginning"
    codec => multiline {
        pattern => "^\["   #正则匹配，开头为[的
        negate => true     #规定匹配到为真
        what => "previous
    }
}
```

#### 收集nginx json格式日志

修改nginx配置文件，使日志输出为json格式

```
vim /etc/nginx/nginx.conf
```

```
http{
    .....
    log_format access_log_json '{"user_ip":"$http_x_forward_for","lan_ip":"$remote_addr","log_time":"time_iso8601","user_req":"$request","http_code":"$status","body_bytes_sent":"$body_bytes_sent","req_time":"$request_time","user_ua":"$http_user_agent"}';
    access_log /var/log/nginx/access_json_log access_log_json;
}
```

在logstash中配置测试

```
vim /etc/logstash/conf.d/nginx_log.conf
```

```ruby
input{
    file{
        path => "/var/log/nginx/access_json.log"
        codec => "json"
    }
}
output{
    elasticsearch {
        hosts => "[192.168.2.4:9200]"
        index => "nginx_access_json-%{+YYYY.MM}"
    }
}
```

#### input插件syslog

修改rsysog配置文件，rsyslog可以将日志输出到文件，数据库或者其他应用

```
vim /etc/rsyslog.conf
```

```
$ModLoad imudp
$UDPServerRun 514

$ModLoad imtcp
$InputTCPServerRun 514

........
*.* @@192.168.2.4:514
```

修改logstash配置文件

```
vim /etc/logstash/conf.d/syslog.conf
```

```
input{
    syslog{
        type => "system-syslog"
        port => 514
    }
}
output{
    stdout{
        codec => rubydebug
    }
}
```

重启rsyslog服务

```
systemctl restart rsyslog
```

#### 写入redis再从redis读取

安装redis

```
mkdir -p /application
```

```
tar xf redis-5.0.0.tar.gz
```

```
cd redis-5.0.0
```

```
make
```

```
cd src/
```

```
make install PREFIX=/appliction/redis
```

复制默认配置文件到安装目录

修改配置文件

```
bind 192.168.2.4
......
daemonize yes
......
requirepass 199747
```

启动

```
/application/resis/bin/redis-server /application/redis/redis.conf
```

连接测试

```
/application/resis/bin/redis-cli -h 192.168.2.4
```

```
> AUTH 199747
```

配置logstash写入redis

```
vim /etc/logstash/conf.d/nginx.conf
```

```
input{
    file{
        path => "/var/log/nginx/access_json.log"
        codec => "json"
    }
}
output{
    redis{
        host => "192.168.2.4"
        port => 6379
        db => "1"
        password => "199747"
        data_type => "list"
        key => "nginx_access_json_log"
    }
}
```

logstash从redis读取后再写入elasticsearch

```
input{
    redis{
        data_type => "list"
        key => "nginx_access_json_log"
        db => "1"
        host => "192.168.2.4"
        port => 6379
        password => "199747"
    }
}
output{
    elasticsearch{
        hosts => "[192.168.2.10:9200]"
        index => "redis-nginx-%{+YYYY.MM.dd}"
    }
}
```

这样就实现了消息队列扩展：数据---->logstash---->MQ/Redis---->logstash---->Elasticsearch

## 5.Beats部分

### 5.1 Beats简介

通过查看ElasticStack可以发现，Beats主要用于采集数据

官网地址：https://www.elastic.co/cn/beats/

Beats平台其实是一个轻量性数据采集器，通过集合多种单一用途的采集器，从成百上千台机器中向Logstash或ElasticSearch中发送数据。

![image.png](https://i.loli.net/2021/10/12/r8ESfa1mdL6MeA3.png)

通过Beats包含以下的数据采集功能

- Filebeat：采集日志文件
- Metricbeat：采集指标
- Packetbeat：采集网络数据

![image.png](https://i.loli.net/2021/10/12/tO2I8o9grlsqTMD.png)

如果我们的数据不需要任何处理，那么就可以直接发送到ElasticSearch中

如果们的数据需要经过一些处理的话，那么就可以发送到Logstash中，然后处理完成后，在发送到ElasticSearch

最后在通过Kibana对我们的数据进行一系列的可视化展示

### 5.2 Filebeat

#### 介绍

Filebeat是一个轻量级的日志采集器

![image.png](https://i.loli.net/2021/10/12/9KHVStdlDYeaNsz.png)

#### why Filebeat?

当你面对成百上千、甚至成千上万的服务器、虚拟机和容器生成的日志时，请告别SSH吧！Filebeat将为你提供一种轻量型方法，用于转发和汇总日志与文件，让简单的事情不再繁华，关于Filebeat的记住以下两点：

- 轻量级日志采集器
- 输送至ElasticSearch或者Logstash，在Kibana中实现可视化

#### 架构

用于监控、收集服务器日志文件.

![image.png](https://i.loli.net/2021/10/12/4QTeJKWDxoiXubw.png)

流程如下：

- 首先是input输入，我们可以指定多个数据输入源，然后通过通配符进行日志文件的匹配
- 匹配到日志后，就会使用Harvester（收割机），将日志源源不断的读取到来
- 然后收割机收割到的日志，就传递到Spooler（卷轴），然后卷轴就在将他们传到对应的地方

#### 下载安装

下载二进制包

```bash
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.8.19-linux-x86_64.tar.gz
```

解压

```bash
tar xf filebeat-6.8.19-linux-x86_64.tar.gz -C /usr/local/
```

测试配置文件

```
vim filebeat.yml
```

```yaml
filebeat.inputs: # filebeat input输入
- type: stdin    # 标准输入
  enabled: true  # 启用标准输入
setup.template.settings: 
  index.number_of_shards: 3 # 指定下载数
output.console:  # 控制台输出
  pretty: true   # 启用美化功能
  enable: true
```

启动

```bash
./filebeat -c filebeat.yml -e
```

输入任意内容返回如下

```json
{
  "@timestamp": "2021-10-12T09:25:13.946Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "doc",
    "version": "6.8.19"
  },
  "log": {
    "file": {
      "path": ""
    }
  },
  "message": "test",
  "input": {
    "type": "stdin"
  },
  "prospector": {
    "type": "stdin"
  },
  "beat": {
    "name": "node01",
    "hostname": "node01",
    "version": "6.8.19"
  },
  "host": {
    "name": "node01"
  },
  "source": "",
  "offset": 0
}
```

#### 读取文件

修改配置文件

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /root/*.log
setup.template.settings:
  index.number_of_shards: 3
output.console:
  pretty: true
  enable: true
```

生成日志文件

```bash
echo "test" >> a.log
```

返回如下

```json
{
  "@timestamp": "2021-10-12T09:27:14.365Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "doc",
    "version": "6.8.19"
  },
  "host": {
    "name": "node01"
  },
  "beat": {
    "name": "node01",
    "hostname": "node01",
    "version": "6.8.19"
  },
  "message": "test",
  "source": "/root/a.log",
  "offset": 0,
  "log": {
    "file": {
      "path": "/root/a.log"
    }
  },
  "prospector": {
    "type": "log"
  },
  "input": {
    "type": "log"
  }
}
```

#### 自定义字段

当我们的元数据没办法支撑我们的业务时，我们还可以自定义添加一些字段

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /root/*.log
  tags: ["web", "test"]  #添加自定义tag，便于后续的处理
  fields:  #添加自定义字段
    from: test-web
  fields_under_root: true #true为添加到根节点，false为添加到子节点中
setup.template.settings:
  index.number_of_shards: 3
output.console:
  pretty: true
  enable: true
```

生成测试内容日志

```bash
echo "haha" >> a.log
```

返回内容

```json
{
  "@timestamp": "2021-10-12T09:29:03.590Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "doc",
    "version": "6.8.19"
  },
  "from": "test-web",
  "prospector": {
    "type": "log"
  },
  "log": {
    "file": {
      "path": "/root/a.log"
    }
  },
  "message": "haha",
  "source": "/root/a.log",
  "tags": [  #自定义的tag字段
    "web",
    "test"
  ],
  "offset": 5,
  "input": {
    "type": "log"
  },
  "host": {
    "name": "node01"
  },
  "beat": {
    "version": "6.8.19",
    "name": "node01",
    "hostname": "node01"
  }
}
```

#### 输出到Elasticsearch

配置文件

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /root/*.log
  tags: ["web", "test"]
  fields:
    from: test-web
  fields_under_root: false 
setup.template.settings:
  index.number_of_shards: 1
output.elasticsearch:
  hosts: ["192.168.2.10:9200"]
```

生成测试日志内容

```bash
echo "es test" >> a.log
```

新生成的索引

![image.png](https://i.loli.net/2021/10/12/Fm9Hu3ElPNeO2AB.png)

#### Filebeat工作原理

Filebeat主要由下面几个组件组成： harvester、prospector 、input

**harvester**：

- 负责读取单个文件的内容
- harvester逐行读取每个文件（一行一行读取），并把这些内容发送到输出
- 每个文件启动一个harvester，并且harvester负责打开和关闭这些文件，这就意味着harvester运行时文件描述符保持着打开的状态。
- 在harvester正在读取文件内容的时候，文件被删除或者重命名了，那么Filebeat就会续读这个文件，这就会造成一个问题，就是只要负责这个文件的harvester没用关闭，那么磁盘空间就不会被释放，默认情况下，Filebeat保存问价你打开直到close_inactive到达

**prospector**：

- prospector负责管理harvester并找到所有要读取的文件来源
- 如果输入类型为日志，则查找器将查找路径匹配的所有文件，并为每个文件启动一个harvester
- Filebeat目前支持两种prospector类型：log和stdin
- Filebeat如何保持文件的状态
  - Filebeat保存每个文件的状态并经常将状态刷新到磁盘上的注册文件中
  - 该状态用于记住harvester正在读取的最后偏移量，并确保发送所有日志行。
  - 如果输出（例如ElasticSearch或Logstash）无法访问，Filebeat会跟踪最后发送的行，并在输出再次可以用时继续读取文件。
  - 在Filebeat运行时，每个prospector内存中也会保存的文件状态信息，当重新启动Filebat时，将使用注册文件的数量来重建文件状态，Filebeat将每个harvester在从保存的最后偏移量继续读取
  - 文件状态记录在filebeat工作目录下的data/registry文件中

**input**：

- 一个input负责管理harvester，并找到所有要读取的源
- 如果input类型是log，则input查找驱动器上与已定义的glob路径匹配的所有文件，并为每个文件启动一个harvester
- 每个input都在自己的Go进程中运行

#### Module

前面要想实现日志数据的读取以及处理都是自己手动配置的，其实，在Filebeat中，有大量的Module，可以简化我们的配置，直接就可以使用，如下

```bash
[root@node01 filebeat-6.8.19-linux-x86_64]# ./filebeat modules list
Enabled:

Disabled:
apache2
auditd
elasticsearch
haproxy
icinga
iis
iptables
kafka
kibana
logstash
mongodb
mysql
nginx
osquery
postgresql
redis
suricata
system
traefik
```

##### 启用nginx模块

````bash
./filebeat modules enable nginx
````

修改nginx模块配置文件

```bash
vim modules.d/nginx.yml
```

```yaml
- module: nginx
  # Access logs
  access:
    enabled: true
    # 添加日志文件
    var.paths: ["/var/log/nginx/access.log*"]

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  # Error logs
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

##### 配置filebeat

```yaml
filebeat.inputs:
setup.template.settings:
  index.number_of_shards: 1
output.elasticsearch:
  hosts: ["192.168.2.10:9200"]
setup.kibana:
  host: "192.168.2.10:5601" # 如果要配置对应kibana中对应的filebeat的可视化模板，则必须配置
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
```

##### 生成es字段和kibana可视化模板

需要注意的是filebeat的主配置文件中必须配置es地址和kibana地址，确保es和kibana是可以被filebeat正常访问

```bash
./filebeat -c filebeat.yml setup
```

#### filebeath合并多行

filebeat在处理tomcat之类的java错误日志时，一个错误日志追踪包含多行，所以需要将多行日志进行多行合并

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
  - /usr/local/tomcat/logs/localhost_access*.txt
  json.keys_under_root: true
  json.overwrite_keys: true
  tags:
  - tomcat-access

- type: log
  enabled: true
  paths:
  - /usr/local/tomcat/logs/catalina.out
  tags:
  - tomcat-error
  multiline.pattern: '^\d{2}'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 10000
 
output.elasticsearch:
  hosts:
  - 192.168.10.21:9200
  indices:
  - index : "tomcat-access-%{[agent.version]}-%{+yyyy.MM.dd}"
    when.contains:
      tags: "tomcat-access"
  - index: "tomcat-error-%{[agent.version]}-%{+yyyy.MM.dd}"
    when.contains:
      tags: "tomcat-error"

setup.ilm.enabled: false
setup.template.name: "tomcat"
setup.template.pattern: "tomcat-*"
```

## 6.elastalert告警

### 6.1 安装elastalert

安装依赖包

```bash
yum install -y gcc-c++ make openssl-devel openssl python3-devel python3-pip
```

安装elastalert

```bash
pip3 install elastalert
```

### 6.2 配置elastalert

config.yaml

```yaml
# 用来加载rule的目录，默认是example_rules
rules_folder: example_rules

# 设置告警的频率，每1分钟执行一次告警
run_every:
  minutes: 1
  # seconds: 30

# 读取日志范围：读取最近1分钟的日志
buffer_time:
  minutes: 1

# 设置elasticsearch的地址及端口
es_host: 192.168.2.10
es_port: 9200

# elasticsearch的认证用户名+密码
#es_username: elastic
#es_password: xxxxxxxx

# 索引名称
writeback_index: elastalert_status
writeback_alias: elastalert_alerts

# 失败重试的时间限制
alert_time_limit:
  days: 2
```

### 6.3 elastalert钉钉告警插件

elastalert_modules/dingtalk_alert.py

```python
from os import times
import time
import hmac
import hashlib
import base64
import urllib.parse
import json
import requests
from elastalert.alerts import Alerter, DateTimeEncoder
from requests.exceptions import RequestException
from elastalert.util import EAException


class DingTalkAlerter(Alerter):
    
    required_options = frozenset(['dingtalk_webhook', 'dingtalk_msgtype'])

    def __init__(self, rule):
        super(DingTalkAlerter, self).__init__(rule)
        self.dingtalk_webhook_url = self.rule['dingtalk_webhook']
        self.dingtalk_msgtype = self.rule.get('dingtalk_msgtype', 'text')
        self.dingtalk_isAtAll = self.rule.get('dingtalk_isAtAll', False)
        self.dingtalk_title = self.rule.get('dingtalk_title', '')
        self.dingtalk_secret = self.rule.get('dingtalk_secret', '')

    def format_body(self, body):
        return body.encode('utf8')
    
    def alert(self, matches):
        params = ""
        if len(self.dingtalk_secret) > 10:
            timestamp, sign = self.get_secret()
            params = {
                "timestamp": timestamp,
                "sign": sign
            }
        headers = {
            "Content-Type": "application/json",
            "Accept": "application/json;charset=utf-8"
        }
        body = self.create_alert_body(matches)
        payload = {
            "msgtype": self.dingtalk_msgtype,
            "text": {
                "content": body
            },
            "at": {
                "isAtAll":False
            }
        }
        try:
            response = requests.post(self.dingtalk_webhook_url, 
                        data=json.dumps(payload, cls=DateTimeEncoder),
                        headers=headers, params=params)
            response.raise_for_status()
        except RequestException as e:
            raise EAException("Error request to Dingtalk: {0}".format(str(e)))

    def get_info(self):
        return {
            "type": "dingtalk",
            "dingtalk_webhook": self.dingtalk_webhook_url
        }
        pass

    def get_secret(self):
        timestamp = str(round(time.time() * 1000))
        secret = self.dingtalk_secret
        secret_enc = secret.encode('utf-8')
        string_to_sign = '{}\n{}'.format(timestamp, secret)
        string_to_sign_enc = string_to_sign.encode('utf-8')
        hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()
        sign = urllib.parse.quote_plus(base64.b64encode(hmac_code))
        return timestamp, sign
```



### 6.4 演示nginx 404告警规则

nginx_access_404.yaml

```yaml
# 告警名称
name: nginx_access_404

# 告警类型
type: frequency

# 告警匹配的索引名称
index: nginx-access*

#告警的条件，查询最近1分钟的日志，当10s内发生5次404错误则触发告警
num_events: 5
timeframe:
  seconds: 10
  #minutes: 1

filter:
- query:
    query_string:
      query: "response_code: 404"


# 告警方式：钉钉
alert_text_type: alert_text_only
alert:
- "elastalert.elastalert_modules.dingtalk_alert.DingTalkAlerter"
dingtalk_webhook: "xxx"
dingtalk_msgtype: "text" 
dingtalk_secret: "xxxx"

alert_text: |
  告警程序: ElasticSearch_Alert
  告警节点: {}
  调用方式: {}
  请求链接: {}
  触发条件: 10s 内 {} 状态码 超过 {} 次
alert_text_args:
  - host.name
  - method
  - request
  - response_code
  - num_hits
```

### 6.5 启动elastalert

```bash
python3 -m elastalert.elastalert --verbose --config config.yaml --rule nginx_access_404.yaml
```
