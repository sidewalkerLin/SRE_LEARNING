### 一、ELK概述

1、什么是ELK?

ELK是Elasticsearch、Logstash、Kibana三大开源框架首字母大写简称。由于后期出现的filebeat代替了logstash的日志收集功能，并且增加了一些新项目，所以从5.X版本后更名为了Elastic Stack。

2、ELK架构

![image-20231021163315299](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231021163315299.png)

### 二、Elasticsearch

1、ES集群选举原理

**ES选举是采用的算法与协议：**

- Elasticsearch7.0版本以前：基于Bully算法改造的算法。在这个机制中，每个节点都有一个唯一的ID，当主节点失效时，ID最小的节点会被选为新的主节点。
- Elasticsearch7.0版本及以后：基于raft协议改造的算法。在这个机制中，一个节点要成为主节点，需要得到集群中n/2以上节点的同意。

**Elasticsearch选举何时发生：**

- 集群启动初始化时
- 群的Master崩溃的时候
- 任何一个节点发现当前集群中的Master节点没有得到n/2 + 1节点认可的时候，触发选举

#### 2、ES文档路由

##### 2.1、ES文档路由原理

我们都知道ES索引是通过分片机制将索引分成多个主分片分布的存放在不通机器上的。那么当一个文档存储到ES集群中时，是存储到哪个主分片上呢？当客户端读取一个文档，是从哪个分片上去读取呢？

实际上文档到分片是有一个映射算法的，算法的目的是为了保证所有文档能均匀的分布在所有分片上。算法如下：

```bash
shard = hash(routing) % number_of_primary_shards
# hash函数会生成一个整数，然后对总的主分片数进行取模，得到的就是文档存储在哪个分片上
# routing 是一个关键参数，默认是文档id，也可以自定义。
# number_of_primary_shards 主分片数
# 注意：该算法与主分片数相关，一但确定后便不能更改主分片。
# 因为一旦修改主分片修改后，Share的计算就完全不一样了
```

**轮询算法也能保证文档均匀分布，但是如果使用轮询算法，那么在客户端读取或修改文档的时候，就需要对所有分片进行遍历，性能上会有比较大的损耗**。

##### 2.2、ES文档创建删除流程

- 客户端发起文档创建或删除的请求，客户端会轮询配置的ES节点以此来达到负载均衡的目的
- 根据轮询策略，此时如果将请求发到了node3，node3会通过文档ID的hash值对主分片数取模，来确定文档应该在哪个分片上，比如shard1
- node3去查询cluster state，确认shard1是在哪个ES节点上，假如是node2，然后node3会将这个创建或删除文档的请求转发给node2
- node2上的主分片收到文档创建或删除的请求后，就会执行这个操作，在文档创建或删除后将这个请求，通过查询cluster state，转发到对应node节点的shard1的副本分片上
- 在所有的副本分片收到请求，并执行创建或删除文档的操作后，会通知主分片
- 主分片收到副本创建或删除文档成功的结果后，通知node3创建或删除文档成功
- node3再将结果返回给client

##### 2.3、ES文档读取流程

- 客户端发起读取文档的请求，客户端会轮询配置的ES节点以此来达到负载均衡的目的
- 根据轮询策略，此时如果将请求发到了node3，node3会通过文档ID的hash值对主分片数取模，来确定文档应该在哪个分片上，比如shard1
- node3去查询cluster state，确认shard1的主副分片列表，然后采用轮询的机制去请求对应的分片，假如此时轮询到了node1上的副本分片R1,那么node3就会将读取文档的请求转发给node1
- node1上的R1将请求结果返回给node3
- node3再返回给client

##### 2.4、ES文档批量创建删除流程





##### 2.5、ES文档批量读取流程



#### 3、Elasticsearch APIs

##### 3.1、Cluster APIs



##### 3.2、CAT APIs



##### 3.3、Document APIs

提供文档的增删改查功能

**(1)单文档API**

- **创建文档**

创建文档可以通过PUT方法或POST方法进行创建。其中PUT方法必须指定文档id；而POST可指定，也可以不指定，不指定则会随机生成一条文档id

```bash
#不管是PUT方法还是POST方法，指定文档id的时候，如果id已经存在，便是做的更新操作
#PUT方法指定文档id
PUT /index_name/_doc/1
{
    "id":1001,
    "name":"张三",
    "age":12,
    "desc":"我的自我描述",
    "birthday":"2020-02-02"
}

#POST方法指定文档id
POST /index_name/_doc/3
{
    "id":1002,
    "name":"张三",
    "age":12,
    "desc":"我的自我描述",
    "birthday":"2020-02-02"
}

#POST方法不指定文档id，文档id随机生成
POST /index_name/_doc
{
    "id":1003,
    "name":"张三",
    "age":12,
    "desc":"我的自我描述",
    "birthday":"2020-02-02"
}
```

- **删除文档**

```bash
DELETE /twitter/_doc/1
```

- **更新文档**

更新文档也有2中方式，一种是使用PUT方法来进行全量更新，一种是使用POST方法来部分更新

```bash
#全量替换
PUT twitter/_doc/1
{
  "name":"jack",
  "age":35   #只修改了age
}

#部分更新
POST twitter/_update/1
{
    "doc" : {    #最外层的doc是固定的结构，我们只需要准备要更新的字段放在里面就行
        "name" : "mark"
    }
}
```

- **查询文档**

```bash
GET twitter/doc/1
```

**(2)多文档API**

- **批量查询文档**

GET /index_name/\_mget是在指定索引中进行查询，GET /\_mget是在全部索引中进行查询

```bash
#GET /index_name/_mget
GET /_mget
{
    "docs" : [
        {
            "_index" : "twitter",
            "_id" : "1"
        },
        {
            "_index" : "twitter",
            "_id" : "2"
        }
    ]
}
```

- **批量增删改文档**

```bash
POST _bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

##### 3.4、Index APIs

索引 API 用于管理各个索引、索引设置、别名、映射和索引模板

```bash
PUT /my-index-000001
{
  "settings": {
    "index": {
      "number_of_shards": 3,  
      "number_of_replicas": 2 
    }
  }
}
```

##### 3.5、Search APIs



```text

```

4、cat APIs

```text
/_cat
/_cat/health
/_cat/indices
/_cat/master
/_cat/nodes
```



put文档必须要指定文档 _id ；post可指定，也可不指定，不指定则会随机生成一个 _id。



2、搭建ES集群

配置信息：

| 主机地址   | 主机名     | 操作系统    | ES版本 |
| ---------- | ---------- | ----------- | ------ |
| 10.0.0.165 | es-master1 | Ubuntu22.04 | 7.17.5 |
| 10.0.0.166 | es-master2 | Ubuntu22.04 | 7.17.5 |
| 10.0.0.162 | es-master3 | Ubuntu22.04 | 7.17.5 |

(1)基础环境准备

```bash
#各节点设置主机名唯一
#centos系列关闭selinux和firewalld
```





























