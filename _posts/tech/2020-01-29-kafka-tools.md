---
layout: post
title: kafka的一些常用工具
category: tech
tags: [kafka,offset,工具,topic, latest]
---

* content
{:toc}



## 环境

以下的操作都是基于kafka_2.11-2.2.0


## 工具

### 新建topic

```
bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 2 --topic spring-kafka-demo2
```

* replication-factor: 指定副本数量
* partitions：指定分区

###  查看topic列表

```
/bin/kafka-topics.sh --zookeeper localhost:2181 --list
```


### 删除某个topic

```
/bin/kafka-topics.sh --zookeeper localhost:2181 --delete -topic spring-kafka-demo
```



### 查看有哪些消费组

```
./bin/kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list
```

老版本是指定zk的地址，类似这样：
```
 kafka-consumer-groups.sh --zookeeper 127.0.0.1:2181 --list
```
 
新版本使用bootstrap，这个是区别。

### 查看某个消费组的详情

```
kafka-consumer-groups.sh --new-consumer --bootstrap-server 127.0.0.1:9092  --group test-consumer-group --describe
```
这个是查看组名为`test-consumer-group`的消费组的情况。

查询的结果类似下面这样，

![屏幕快照 2020-01-29 下午8.43.17.png](https://note.youdao.com/yws/res/29055/WEBRESOURCEb3728e52de4a08660a8c8bfa5ea4693b)

简单对每列进行说明：

* TOPIC：
　　消费者的topic名称　　
* PARTITION：
　　分区数的名称　　

* CURRENT-OFFSET：
　　consumer group最后一次提交的offset

* LOG-END-OFFSET：
　　最后提交的生产消息offset
* LAG
　　消费offset与生产offset之间的差值
* CONSUMER-ID
　　消费者的ID编号，我们知道消费者组里面可以有最少要有一个消费者，当然也可以有多个消费者。
* HOST
　　消费者的主机IP地址。
* CLIENT-ID
　　链接的ID编号。

关于offset补充一些知识点。

kafka有个常用的设置是 `auto.offset.reset`，它的意义是，

>该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下（因消费者长
时间失效，包含偏移量的记录已经过时井被删除）该作何处理。它的默认值是 latest ， 意
思是说，在偏移量无效的情况下，消费者将从最新的记录开始读取数据（在消费者启动之
后生成的记录）。另一个值是 earliest ，意思是说，在偏移量无效的情况下，消费者将从
起始位置读取分区的记录。

这个属性有以下几个值，

* earliest： 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
* latest： 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
* none： topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常


**我要强调的是，这个设置只有在我们的消费者（或者消费者群组）在分区内找不到有效的offset时才会生效。**

举个例子，

我们使用java kafka客户端来操作kafak。

比如我们在消费组`group1`有个消费者，消费了5条消息然后节点挂了。

然后我们重启这个消费节点，那么我来问你，这个消费者会从哪里开始消费？

如果你回答根据`auto.offset.reset`的配置来决定那就说明你没理解我上面所说的。

正确的答案是，消费者会继续从上次挂掉的offset（kafka broker保存）那里继续消费，根本不理会`auto.offset.reset`。


再举个例子，

生产者在某个topic生产了一些消息，然后我们启动一个消费组`group2`，里面有一个消费者。

如果这个时候kafka没有这个topic消息的offset信息，那么`auto.offset.reset`的值就决定从哪里消费。
