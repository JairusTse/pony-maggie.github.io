---
layout: post
title: 从一个生产上的错误看kafka的消费再均衡问题
category: tech
tags: [kafka,再均衡,topic,分区]
---

* content
{:toc}

## 问题描述
项目在生产上的一段错误日志如下，

```bash
[commitSync] processed message to kafka failed, Just Ignore this commit, wait for next commit to make these messages processed.org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:786)
```

这是一段kafka的错误日志，大概的意思是说，
>kafka的服务端在超过了 max.poll.interval.ms 时间内没有收到某个消费者的心跳，认为该消费者已经“挂了”，所以进行了topic的分区所有权“再均衡”。


## 问题的分析

按照我的个人习惯，遇到类似这样的生产问题，解决之后我会思考下涉及的技术细节并做整理。

如果对问题涉及的技术细节非常的了解，对于定位问题是非常有帮助的。本文就带你深入了解下上面那个错误日志涉及的一些技术细节。

### kafka的topic分区

为了提高消息处理的高可用以及便于横向扩展，kafka引入了topic的分区概念。属于同一个消费者群组的消费者可以分担的消费同一个topic不同分区的消息。从而达到分流的作用，可以使消息处理更高效。

![](http://pony-maggie.github.io/assets/images/2019/tech/11/kafka/kafka-partition.png)

如上图示例所示，topic A有三个分区，同时我们有三个属于同一个群组的消费者，这样每个消费者可以负责消费一个分区。大家各自负责自己的分区，系统有条不紊的运行着。

**一般情况下，我们通过增加群组里的消费者数量来提高 kafka 的消费能力。不过要注意，不要让消费者的数量超过主题分区的数量，多余的消费者只会被闲置。**

### 心跳机制

kafka 的服务端需要一直监控有哪些消费者在消费，监控的机制是通过消费者不断的发送心跳包实现的。消费者发送心跳有两个途径，一个是轮询(poll，这里不是为了秀英文，注意联系上面的错误日志)，一个是消费后提交 offset 。

**这两种方式是两个独立的线程，互相不干扰。**

只要消费者以正常的时间间隔发送心跳，就被认为是活跃的，说明它还在读取分区里的消息，否则就被认为是已经“死亡”。
这个所谓的正常的时间间隔，就是不能超过 max.poll.interval.ms。


### kafka的分区再均衡

消费者通过向服务端发送心跳来维持它们和群组的从属关系以及它们对分区的所有权关系。如果服务端认为某个消费者已经“死亡”，就会触发一次再均衡。如下图所示，

![](http://pony-maggie.github.io/assets/images/2019/tech/11/kafka/kafka-rebalance.png)

前面说过，群组里的消费者共同读取主题的分区。

比如有一个新的消费者加入群组，它读取的是原本由其他消费者读取的消息。当一个消费者被关闭或发生崩溃时，它就离开群组，原本由它读取的分区将由群组里的其他消费者来读取。

**分区的所有权从一个消费者转移到另一个消费者，这样的行为被称为再均衡。**

再均衡有什么意义吗？

当然，有了再均衡，我们可以放心的添加或者移除某个消费者，而不用担心消息的丢失。

## 解决问题

了解了相关的技术细节后，我们可以顺藤摸瓜，慢慢排查问题。基于前面的分析，我给出几个排查的方向：

1. 看看某个消费者的服务是否已经挂了？
2. 如果服务正常运行，服务所在的节点是否存在内存或者CPU占满的情况，导致消费者无法及时的发送心跳等。我遇到的情况就是后者引起的。后来解决了内存占用满的问题后，kafka的错误就不存在了。
3. 根据自己实际的业务情况，考虑增加 max.poll.interval.ms 的值。


参考:
1. 《kafka权威指南》
