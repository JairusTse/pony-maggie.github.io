---
layout: post
title: springboot整合kafka自动提交的问题
category: tech
tags: [java,springboot,kafka,offset,自动提交]
---

* content
{:toc}


最近遇到一个springboot整合kafka设置手动提交不生效的问题，后来发现是自己的方法不对，走了一些弯路，这里记录一下。

## 环境准备

* spring boot 2.1.6.RELEASE
* 本地zk, 单节点kafka，版本是kafka_2.11-2.2.0

新建一个topic，topic名是 `spring-kafka-demo4`，如下：

```
bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 2 --topic spring-kafka-demo3
```

topic设置了两个分区。

## 问题描述

消费者工程的设置如下：

```
spring.kafka.consumer.group-id=test-consumer-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.consumer.auto-commit-interval=100
```

消费的逻辑使用springboot注解，如下：

```java
public class KafkaReceiver {

    @KafkaListener(clientIdPrefix = "consumer-1", topics = {"spring-kafka-demo4"})
    public void listen(ConsumerRecord<?, ?> record) {

        Optional<?> kafkaMessage = Optional.ofNullable(record.value());

        if (kafkaMessage.isPresent()) {
            Object message = kafkaMessage.get();
            log.info("receive ------------------ message =" + message);
        }

    }
}
```

我启动一个生产者生产了5条消息，然后启动消费者。从日志上看消费正常，然后我使用`kafka-consumer-groups.sh` 工具查看了一下消费的情况，如下：

![](http://pony-maggie.github.io/assets/images/2020/java/01/1-1.png)


从图上看不对呀，不是设置了不自动提交offset吗？ 我的消费者逻辑里也没有手动提交的代码，为啥看到的两个分区消费者提交了offset呢？

为了防止有些人不明白，我简单对每列进行说明：

* TOPIC：消费者的topic名称　　
* PARTITION：分区数的名称　　
* CURRENT-OFFSET：consumer group最后一次提交的offset
* LOG-END-OFFSET：最后提交的生产消息offset
* LAG：消费offset与生产offset之间的差值
* CONSUMER-ID：消费者的ID编号，我们知道消费者组里面可以有最少要有一个消费者，当然也可以有多个消费者。
* HOST：消费者的主机IP地址。
* CLIENT-ID：链接的ID编号。

后来查了一些资料后，发现还需要设置 `ack-mode = manual`才可以，完整的设置如下：

```
spring.kafka.consumer.group-id=test-consumer-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.consumer.auto-commit-interval=100
spring.kafka.listener.ack-mode=manual
```

然后我们再重新生产5条新的消息，然后同样的逻辑消费，再看下消费的情况：

![](http://pony-maggie.github.io/assets/images/2020/java/01/1-2.png)


这次就正常了，可以看到消费者在两个分区上都没有提交offset。

那么既然我们设置了手动提交，如何在代码中手动提交呢？其实也非常简单，

```java
 @KafkaListener(clientIdPrefix = "consumer-1", topics = {"spring-kafka-demo4"})
    public void listen(ConsumerRecord<?, ?> record, Acknowledgment acknowledgment) {

        Optional<?> kafkaMessage = Optional.ofNullable(record.value());

        if (kafkaMessage.isPresent()) {

            Object message = kafkaMessage.get();
            log.info("receive ------------------ message =" + message);
        }

        acknowledgment.acknowledge();

    }
```

## 源码分析

现在我们从源码层面分析下，为啥只设置`enable-auto-commit=false` 时，spring默认自动提交了offset。

我们在依赖包里加入断点分析下。

![](http://pony-maggie.github.io/assets/images/2020/java/01/1-3.png)


注意到 `isManualAck` ， `isAnyManualAck` 以及 `isManualImmediateAck`这三个变量，当配置文件配置 `ack-mode=manual`时，前两个变量是true，最后一个是false。

继续往下看，

![](http://pony-maggie.github.io/assets/images/2020/java/01/1-4.png)


processCommit方法里有个 `updatePendingOffsets` 方法，这个方法就是用来提交offset的。很明显当`isManualAck`是true，`isManualImmediateAck`是false时， 这个方法不会执行。



