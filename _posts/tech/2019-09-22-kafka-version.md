---
layout: post
title: kafka系列之彻底弄清楚各版本差异
category: tech
tags: [kafka,版本,客户端,producer]
---

* content
{:toc}

我自己用了 kafka 也挺久的了，关于kafka的版本规则，各个大版本的升级究竟做了哪些优化等，并没有特别的关注。

本文打算做一个比较详细的整理。

## 1、版本命名规则

1.x之后，kafka 全面启用三位数的命名规则。也就是说，以前的版本都是这样色的，

* 0.8.2.2
* 0.9.0.1
* 0.10.0.0

后来1·x之后，kafka 全面启用了三位数版本规则，如果下图所示，

![kafka1.png](http://note.youdao.com/yws/res/21709/WEBRESOURCE6d074349f9c5eb438c84ae8ba868fd24)

新的版本规则，即 “大版本-小版本-patch版本“ 比较符合主流。

我们现在看到的 kafka 版本通常是这样的，
* kafka_2.11-2.2.0

前面部分2.11其实是scala的版本（kafka是scala编写的），后面三位就是真正的 kafka 版本。


## 2、几个主要的里程碑

### 0.8.2版本
* 为了提高吞吐量，producer 都以异步批量的方式发送消息到 broker 节点。
* consumer 的消费偏移位置 offset 由原来的保存在 zookeeper 改为保存在 kafka 本身。

### 0.9版本
* 增加安全相关特性，客户端连接 kafka 可以使用ssl或者sasl进行验证。
* 增加 kafka connect 模块
* 新的 consumer api 

### 1.0.0版本
* 支持 java 9
* 增强 stream api
* 引入了线程协议，便于升级

### 2.0.0版本
* 最低支持 java8
* 弃用多处 scala 相关的依赖，java 成主流

### 2.2.0
* 默认的consumer group id 由 "" 改为 null。
* `bin\kafka-topic.sh` 支持指定 `--bootstrap-server`，代替原来的`--zookeeper`。


## 3、关于客户端版本

kafka 支持多个语言的客户端api，我只关注 java 客户端。maven 的工程我们一般这样引入 kafka 客户端，

```xml
<dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>0.10.2.0</version>
        </dependency>
```

这种会引入两个依赖jar，分别是
* kafka-clients-0.10.2.0.jar
* kafka_2.11-0.10.2.0.jar

前者是官方推荐的java客户端，后者是scala客户端。调用方式有所不同。如果确定不使用 scala api，也可以用下面这种方式只包含java版本的客户端。
```xml
<dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.10.2.0</version>
        </dependency>
```


一个原则是，**尽量保持客户端版本和服务器上运行的server版本一致**。

参考：

http://kafka.apache.org/documentation.html#upgrade_110_notable
