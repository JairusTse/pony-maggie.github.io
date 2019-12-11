---
layout: post
title: 使用kafka连接器迁移mysql数据到ElasticSearch
category: tech
tags: [kafka,连接器,mysql,ElasticSearch]
---

* content
{:toc}

## 概述

把 mysql 的数据迁移到 es 有很多方式，比如直接用 es 官方推荐的 logstash 工具，或者监听 mysql 的 binlog 进行同步，可以结合一些开源的工具比如阿里的 canal。

这里打算详细介绍另一个也是不错的同步方案，这个方案基于 kafka 的连接器。流程可以概括为：
1. mysql连接器监听数据变更，把变更数据发送到 kafka topic。
2. ES 监听器监听kafka topic 消费，写入 ES。

Kafka Connect有两个核心概念：Source和Sink。 Source负责导入数据到Kafka，Sink负责从Kafka导出数据，它们都被称为Connector，也就是连接器。在本例中，mysql的连接器是source，es的连接器是sink。

这些连接器本身已经开源，我们之间拿来用即可。不需要再造轮子。

## 过程详解

### 准备连接器工具

我下面所有的操作都是在自己的mac上进行的。

首先我们准备两个连接器，分别是 `kafka-connect-elasticsearch` 和 `kafka-connect-elasticsearch`， 你可以通过源码编译他们生成jar包，源码地址：

[kafka-connect-elasticsearch](https://github.com/confluentinc/kafka-connect-elasticsearch)

[kafka-connect-mysql](https://github.com/confluentinc/kafka-connect-jdbc)

我个人不是很推荐这种源码的编译方式，因为真的好麻烦。除非想研究源码。

我是直接下载 confluent 平台的工具包，里面有编译号的jar包可以直接拿来用，下载地址：

[confluent 工具包](https://www.confluent.io/download/compare/)

**我下载的是 confluent-5.3.1 版本**, 相关的jar包在 confluent-5.3.1/share/java 目录下

我们把编译好的或者下载的jar包拷贝到kafka的libs目录下。拷贝的时候要注意，除了 kafka-connect-elasticsearch-5.3.1.jar 和 kafka-connect-jdbc-5.3.1.jar，相关的依赖包也要一起拷贝过来，比如es这个jar包目录下的http相关的，jersey相关的等，否则会报各种 `java.lang.NoClassDefFoundError` 的错误。

另外mysql-connector-java-5.1.22.jar也要放进去。



### 数据库和ES环境准备

数据库和es我都是在本地启动的，这个过程具体就不说了，网上有很多参考的。

我创建了一个名为test的数据库，里面有一个名为login的表。

### 配置连接器

这部分是最关键的，我实际操作的时候这里也是最耗时的。

首先配置jdbc的连接器。

我们从confluent工具包里拷贝一个配置文件的模板(confluent-5.3.1/share目录下)，自带的只有sqllite的配置文件，拷贝一份到kafka的config目录下，改名为sink-quickstart-mysql.properties，文件内容如下：

```
# tasks to create:
name=mysql-login-connector
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
tasks.max=1
connection.url=jdbc:mysql://localhost:3306/test?user=root&password=11111111
mode=timestamp+incrementing
timestamp.column.name=login_time
incrementing.column.name=id
topic.prefix=mysql.
table.whitelist=login
```

connection.url指定要连接的数据库，这个根据自己的情况修改。mode指示我们想要如何查询数据。在本例中我选择incrementing递增模式和timestamp 时间戳模式混合的模式， 并设置incrementing.column.name递增列的列名和时间戳所在的列名。

** 混合模式还是比较推荐的，它能尽量的保证数据同步不丢失数据。**具体的原因大家可以查阅相关资料，这里就不详述了。


topic.prefix是众多表名之前的topic的前缀，table.whitelist是白名单，表示要监听的表，可以使组合多个表。两个组合在一起就是该表的变更topic，比如在这个示例中，最终的topic就是mysql.login。

connector.class是具体的连接器处理类，这个不用改。

其它的配置基本不用改。

接下来就是ES的配置了。同样也是拷贝 quickstart-elasticsearch.properties 文件到kafka的config目录下，然后修改，我自己的环境内容如下：

```
name=elasticsearch-sink
connector.class=io.confluent.connect.elasticsearch.ElasticsearchSinkConnector
tasks.max=1
topics=mysql.login
key.ignore=true
connection.url=http://localhost:9200
type.name=mysqldata
```

topics的名字和上面mysql设定的要保持一致，同时这个也是ES数据导入的索引。从里也可以看出，ES的连接器一个实例只能监听一张表。

type.name需要关注下，我使用的ES版本是7.1，我们知道在7.x的版本中已经只有一个固定的type(_doc)了，使用低版本的连接器在同步的时候会报错误，我这里使用的5.3.1版本已经兼容了。继续看下面的章节就知道了。

关于es连接器和es的兼容性问题，有兴趣的可以看看下面这个issue：

https://github.com/confluentinc/kafka-connect-elasticsearch/issues/314

### 启动测试

当然首先启动zk和kafka。

然后我们启动mysql的连接器，

```
./bin/connect-standalone.sh config/connect-standalone.properties config/source-quickstart-mysql.properties &
```

接着手动往login表插入几条记录，正常情况下这些变更已经发到kafka对应的topic上去了。为了验证，我们在控制台启动一个消费者从mysql.login主题读取数据：

```
./bin/kafka-console-consumer.sh --bootstrap-server=localhost:9092 --topic mysql.login --from-beginning
```

可以看到刚才插入的数据。

把数据从 MySQL 移动到 Kafka 里就算完成了，接下来把数据从 Kafka 写到 ElasticSearch 里。

首先启动ES和kibana，当然后者不是必须的，只是方便我们在IDE环境里测试ES。你也可以通过控制台给ES发送HTTP的指令。

先把之前启动的mysql连接器进程结束（因为会占用端口），再启动 ES 连接器，
```
./bin/connect-standalone.sh config/connect-standalone.properties config/quickstart-elasticsearch.properties &

```

如果正常的话，ES这边应该已经有数据了。打开kibana的开发工具，在console里执行

```
GET _cat/indices
```
这是获取节点上所有的索引，你应该能看到，
```
green open mysql.login                  1WqRjkbfTlmXj8eKBPvAtw 1 1      4    0    12kb   7.8kb
```
说明索引已经正常创建了。然后我们查询下，
```
GET mysql.login/_search?pretty=true
```
结果如下，
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "mysql.login",
        "_type" : "mysqldata",
        "_id" : "mysql.login+0+0",
        "_score" : 1.0,
        "_source" : {
          "id" : 1,
          "username" : "lucas1",
          "login_time" : 1575870785000
        }
      },
      {
        "_index" : "mysql.login",
        "_type" : "mysqldata",
        "_id" : "mysql.login+0+1",
        "_score" : 1.0,
        "_source" : {
          "id" : 2,
          "username" : "lucas2",
          "login_time" : 1575870813000
        }
      },
      {
        "_index" : "mysql.login",
        "_type" : "mysqldata",
        "_id" : "mysql.login+0+2",
        "_score" : 1.0,
        "_source" : {
          "id" : 3,
          "username" : "lucas3",
          "login_time" : 1575874031000
        }
      },
      {
        "_index" : "mysql.login",
        "_type" : "mysqldata",
        "_id" : "mysql.login+0+3",
        "_score" : 1.0,
        "_source" : {
          "id" : 4,
          "username" : "lucas4",
          "login_time" : 1575874757000
        }
      }
    ]
  }
}
```

-------

参考：

1.《kafka权威指南》

2. https://www.jianshu.com/p/46b6fa53cae4