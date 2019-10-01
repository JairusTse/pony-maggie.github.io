---
layout: post
title: kafka系列之camel-kafka
category: tech
tags: [kafka,camel,topic,endpoint]
---

* content
{:toc}

## 概述

首先关于 camel 的基本概念和用法，以及 kafka 的基本概念和用法，这里就不啰嗦了。这篇文章假设你对二者都有基本的认识。

camel 本身是一个路由引擎，通过 camel 你可以定义路由规则，指定从哪里（源）接收消息，如何处理这些消息，以及发往哪里（目标）。camel-kafka 就是 camel 的其中一个组件，它从指定的 kafka topic 获取消息来源进行处理。

有些小伙伴可能有疑问了，kafka 本身不就是生产者-消费者模式吗？原生 kafka 发布消息，然后消费进行消息处理不就行了，为啥还用 camel-kafka 呢？

首先恭喜你是一个爱思考的小伙伴！这个问题的答案是这样，camel 本身提供的是高层次的抽象，你可以选择从 kafka 作为源接收数据，也可以使用其它组件，比如mq，文件等。camel 让你能使用相同的api和处理流程，处理不同协议和数据类型的系统。

所有总结下，（下面这句话很重要，读三遍）

**camel实现了客户端与服务端的解耦， 生产者和消费者的解耦。**

比如我们可以选择从kafka获取消息，然后发送到jms（activemq）。

```java
from("kafka:test?brokers=localhost:9092")
.to("jms:queue:test.mq.queue")
```

![](http://pony-maggie.github.io/assets/images/2019/tech/10/camel-kafka/1.png)

## 详解camel-kafka

camel对每个组件约定一个发送和接受的 endpoint uri，kafka 的uri格式是，
```
kafka:topic[?options]
```

option中有很多选项，比如最重要的 brokers 指定 kafka 服务的地址。然后是uri的参数，类似http uri的参数格式。下面是个示例：
```java
from("kafka:test?brokers=localhost:9200"
                        + "&maxPollRecords=5000"
                        + "&consumersCount=1"
                        + "&seekTo=beginning"
                        + "&groupId=kafkaGroup")
                        .routeId("FromKafka")
                    .to("stream:out");
```
每个参数的意思我就不一一解释了，在文章最后我会给出官方的链接，大家可以自己去查阅。


说了这么多，我们还是运行一个程序看看效果。这个程序来自 apache camel 官方example，完整的代码在文章的最后有链接。

首先，pom引入依赖，

```xml
  <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-kafka</artifactId>
            <version>2.24.1</version>
        </dependency>
```


我需要在本地启动一个 kafka 的server，具体过程网上很多，这里不啰嗦了。唯一要注意的是 kafka server 的版本最好跟 camel-kafka 引入的 kafka-client 版本一致，以免踩坑。

kafka环境安装好之后，创建两个topic，

```bash
bogon:kafka_2.11-2.2.0 ponyma$ ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 2 --topic TestLog
Created topic TestLog.
bogon:kafka_2.11-2.2.0 ponyma$ ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic AccessLog
Created topic AccessLog.
```

先来看看消费者部分的代码，

```java
public static void main(String[] args) throws Exception {

        System.out.println("About to run Kafka-camel integration2...");

        CamelContext camelContext = new DefaultCamelContext();

        // Add route to send messages to Kafka

        camelContext.addRoutes(new RouteBuilder() {
            public void configure() {
                PropertiesComponent pc = getContext().getComponent("properties", PropertiesComponent.class);
                pc.setLocation("classpath:application.properties");

                System.out.println("About to start route: Kafka Server -> Log ");

                from("kafka:{{consumer.topic}}?brokers={{kafka.host}}:{{kafka.port}}"
                        + "&maxPollRecords={{consumer.maxPollRecords}}"
                        + "&consumersCount={{consumer.consumersCount}}"
                        + "&seekTo={{consumer.seekTo}}"
                        + "&groupId={{consumer.group}}")
                        .routeId("FromKafka")
                    .to("stream:out");
            }
        });
        camelContext.start();

        // let it run for 5 minutes before shutting down
        Thread.sleep(5 * 60 * 1000);

        camelContext.stop();
    }
```

这个代码的核心就是camel的路由配置，也很简单，当前这个路由的意思是，从 kafka 某个 topic 读取数据，不做任何处理直接发送到标准输出。

再来看看生产者，
```java

        camelContext.addRoutes(new RouteBuilder() {
            public void configure() {
                PropertiesComponent pc = getContext().getComponent("properties", PropertiesComponent.class);
                pc.setLocation("classpath:application.properties");

                // setup kafka component with the brokers
                KafkaComponent kafka = new KafkaComponent();
                kafka.setBrokers("{{kafka.host}}:{{kafka.port}}");
                camelContext.addComponent("kafka", kafka);

                from("direct:kafkaStart").routeId("DirectToKafka")
                    .to("kafka:{{producer.topic}}").log(LoggingLevel.INFO, "${headers}");

                // Topic can be set in header as well.

                from("direct:kafkaStartNoTopic").routeId("kafkaStartNoTopic")
                    .to("kafka:dummy")
                    .log(LoggingLevel.INFO, "${headers}");

                // Use custom partitioner based on the key.
                /**
                 * partitioner指定分区发送
                 */

                from("direct:kafkaStartWithPartitioner").routeId("kafkaStartWithPartitioner")
                        .to("kafka:{{producer.topic}}?partitioner={{producer.partitioner}}")
                        .log("${headers}");


                // Takes input from the command line.

                from("stream:in").setHeader(KafkaConstants.PARTITION_KEY, simple("0"))
                        .setHeader(KafkaConstants.KEY, simple("1")).to("direct:kafkaStart");

            }

        });
```

* 第一个 from to 意思是监听 direct:kafkaStart ，发送到指定的 topic。
* 第二个 from to 也是监听某个 direct，但是没有发送的 kafka的topic上。
* 第三个 from to 是监听 direct:kafkaStartWithPartitioner，发送到特定 topic 的特定分区上。
* 第四个 from to 是监听我们在控制台的输入，发送到 direct:kafkaStart。

上面四个 from to 对应 下面四个发送的示例，通过日志打印我们可以看看数据是否被正确的进行路由了。

```java
headers.put(KafkaConstants.PARTITION_KEY, 0);
headers.put(KafkaConstants.KEY, "1");
producerTemplate.sendBodyAndHeaders("direct:kafkaStart", testKafkaMessage, headers);
```
这段代码的意思是，生产者发送数据到 direct:kafkaStart 这个endpoint上, headers指定了所有的消息都会发送到 kafka topic 的第一个分区。

```java
headers.put(KafkaConstants.KEY, "2");
headers.put(KafkaConstants.TOPIC, "TestLog");
producerTemplate.sendBodyAndHeaders("direct:kafkaStartNoTopic", testKafkaMessage, headers);
```
生产者发送数据到 direct:kafkaStartNoTopic 这个endpoint上，对应上面第二个 from to ，虽然没有指定发送目标的 kafka topic，但是我们在 header 里指定了 topic，所以跟第一个 from to 其实可以达到同样的效果。

后面两个就不贴出代码了，一个是发送到分区0，一个发送到分区1。分区的原则是 header 里指定的key，分区器是自定义的，在源码 stringPartitioner.java 中。这里不表。

先启动消费者端，然后启动生产者端，结果如下：

![](http://pony-maggie.github.io/assets/images/2019/tech/10/camel-kafka/2.png)

![](http://pony-maggie.github.io/assets/images/2019/tech/10/camel-kafka/3.png)

可以看到，运行的结果跟我们分析的是一致的。


-----

本文所用的示例源码地址：

https://camel.apache.org/components/latest/kafka-component.html

参考：

https://github.com/apache/camel/tree/master/examples/camel-example-kafka