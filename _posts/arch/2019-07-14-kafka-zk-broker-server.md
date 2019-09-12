---
layout: post
title: Kafka中的broker-list,bootstrap-server以及zookeeper
category: arch
tags: [Kafka,zookeeper,broker,bootstrap-server]
---

* content
{:toc}

我刚学kafka的时候，对这几个概念有时候会混淆，尤其是配置的时候经常搞不清楚它们的区别。这篇文章打算做一个梳理。

## broker-list

broker指的是kafka的服务端，可以是一个服务器也可以是一个集群。producer和consumer都相当于这个服务端的客户端。

broker-list指定集群中的一个或者多个服务器，一般我们再使用console producer的时候，这个参数是必备参数，另外一个必备的参数是topic，如下示例：

```
C:\kafka\kafka_2.12-1.1.1
λ .\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic test20190713
>this is a test
>
```

本地主机如果要模拟多个broker，方法是复制多个server.properties，然后修改里面的端口， broker.id等配置模拟多个broker集群。

比如模拟三个broker的情况，首先把config 目录下的 server.properties 复制两份，分别命名为 server-1.properties 和 server-2.properties，然后修改这两个文件的下列属性，

server-1.properties:

```
broker.id=1
listeners=PLAINTEXT://:9093
log.dirs=C:/kafka/broker1
```

server-2.properties:

```
broker.id=2
listeners=PLAINTEXT://:9094
log.dirs=C:/kafka/broker2
```

broker.id 用来唯一标识每一个 broker，每个broker都有一个唯一的id值用来区分彼此。Kafka在启动时会在zookeeper中/brokers/ids路径下创建一个与当前broker的id为名称的虚节点，Kafka的健康状态检查就依赖于此节点。

我们可以打开一个zk的客户端，通过ls命令来查看下这个路径下的内容：

```
λ .\bin\windows\zookeeper-shell.bat localhost:2181
Connecting to localhost:2181
Welcome to ZooKeeper!
JLine support is disabled
ls
WATCHER::

WatchedEvent state:SyncConnected type:None path:null

ls /brokers/ids
[0]
```

可以看到我们默认启动的这个broker.id为0的节点。

## bootstrap-servers vs zookeeper

bootstrap-servers指的是目标集群的服务器地址，这个和broker-list功能是一样的，只不过我们在console producer要求用后者。

以前我们使用console consumer测试消息收发时会这样写：

```
C:\kafka\kafka_2.12-1.1.1
λ .\bin\windows\kafka-console-consumer.bat --zookeeper localhost:2181 --topic test20190713
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
```

这样可以接收到生产者控制台发送的消息。

现在我们也可以这样写，

 ```
C:\kafka\kafka_2.12-1.1.1
λ .\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test20190713
 ```

你可以自己测试下，也是可以收到消息的。

前者是老版本的用法，0.8以前的kafka，消费的进度(offset)是写在zk中的，所以consumer需要知道zk的地址。后来的版本都统一由broker管理，所以就用bootstrap-server了。如下图所示：

![kafka1](http://pony-maggie.github.io/assets/images/2019/tech/kafka1.jpg)

![kafka2](http://pony-maggie.github.io/assets/images/2019/tech/kafka2.jpg)



bootstrap-server还可以自动发现其它的broker。









