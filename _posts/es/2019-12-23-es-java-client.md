---
layout: post
title: Elasticsearch java API客户端介绍
category: es
tags: [java,API,客户端,ElasticSearch]
---

* content
{:toc}

基本上官方指南就已经向我们说明了一切。如下图所示：


![](http://pony-maggie.github.io/assets/images/2019/es/12/12.jpg)

从官方指南上，ES的java 客户端分为两个大类。分别是：

* [Java REST Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html)
* [Java API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html)

下面分别说下这两种有什么区别。

## Java API

在ES 7.0之前最常采用的API，基于TransportClient客户端。网上大部分ES 客户端的资料基本都是基于它的。这种方式在ES 7.x后已经不被官方推荐，且在8.0版本中完全移除它。

鉴于有很多人还在使用低版本的ES，所以这种方式在一段时间内应该还是不会消失。我们来看看它的基本使用示例。

首先我们在maven中引入依赖，

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>transport</artifactId>
    <version>7.1.1</version>
</dependency>
```

连接一个集群，

```java
Settings settings = Settings.builder()
        .put("cluster.name", "myClusterName").build();

TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host1"), 9300))
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host2"), 9300));
```

索引一个文档，
```java
IndexResponse response = client.prepareIndex("twitter", "_doc", "1")
        .setSource(jsonBuilder()
                    .startObject()
                        .field("user", "kimchy")
                        .field("postDate", new Date())
                        .field("message", "trying out Elasticsearch")
                    .endObject()
                  )
        .get();
```

## Java REST Client

这是官方推荐的客户端，分为 Low Level REST Client 和 High Level REST Client，区别在于前者是直接让你通过 http 和 es 的集群通信，它更加灵活，随之带来的问题是调用者需要关心的细节也很多。调用者需要对 ES 较为熟悉才可以用好这些API。

High Level REST Client则是对Low Level REST Client的封装，它隐藏了大部分ES的细节，使得调用者即使不了解ES的细节也能用好客户端API。

下面来看看High Level REST Client的使用示例。

maven引入依赖，

```xml
<dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>7.1.0</version>
        </dependency>
        
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>7.1.0</version>
        </dependency>
```

根据集群信息创建客户端实例，
```java
public RestHighLevelClient restClient() {
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY,
                new UsernamePasswordCredentials(userName, password));

        RestClientBuilder builder = RestClient.builder(new HttpHost(host, port))
                .setHttpClientConfigCallback(httpClientBuilder -> httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider));
        RestHighLevelClient client = new RestHighLevelClient(builder);
        return client;

    }
```



创建索引，
```java
public String createProfileDocument(ProfileDocument document) throws Exception {
        UUID uuid = UUID.randomUUID();
        document.setId(uuid.toString());
        IndexRequest indexRequest = new IndexRequest(INDEX, TYPE, document.getId())
            .source(convertProfileDocumentToMap(document));
        IndexResponse indexResponse = client.index(indexRequest, RequestOptions.DEFAULT);
        return indexResponse.getResult().name();
    }
```

## 性能对比

大家直接看下面这篇文章吧，有详细的对比数据。

[Benchmarking REST client and transport client](https://www.elastic.co/cn/blog/benchmarking-rest-client-transport-client)


如果你英语不好，我可以大概解释下。

文章中的实验从bulk index和search两个维度测试对比了二者之间的性能，这篇文章是用es 5.0的版本进行的测试，结果显示虽然rest client在性能上已经和transport client 相差不大了。而且ES官方还会不断的优化前者，所以你基本上不用担心性能上的瓶颈。


## 总结

大部分时候你都应该使用 high level的api进行ES操作，虽然自己使用http直接封装ES的客户端也是可以的。但是还是推荐使用high level的客户端API。一方面是它隐藏了ES的复杂操作，让你即使对ES不熟悉也能轻松的使用API进行读写数据。另一方面，大概率它比自己的封装更稳定。

另外，两种客户端走的协议和端口也不一样，TransportClient客户端使用的TCP协议，9300端口，而rest client使用的是http协议，走的是9200端口。

另外，spring boot官方有对ES封装的starter，可以和spring data集成使用。这种方式用起来肯定更方便，不过有个缺点就是更新太慢了，截止到我写这篇文章，spring data es的版本是3.2.x，只支持到ES 6.8.1的版本。

![](http://pony-maggie.github.io/assets/images/2019/es/12/11.jpg)

**我个人比较推荐的还是 High Level REST Client 这种方式。**



参考:

* [ES官方文档](https://www.elastic.co/guide/index.html)
* [Benchmarking REST client and transport client](https://www.elastic.co/cn/blog/benchmarking-rest-client-transport-client)
