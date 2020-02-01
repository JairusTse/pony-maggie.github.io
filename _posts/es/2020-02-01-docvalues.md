---
layout: post
title: 一文带你彻底弄懂ES中的doc_values和fielddata
category: elasticsearch
tags: [elasticsearch,es,doc_values,fielddata]
---

* content
{:toc}

## 基本概念

这两个概念比较像，所以大部分时候会放在一起说。

这两个概念源于`Elasticsearch`（后面简称ES）除了强大的搜索功能外，还可以支持排序，聚合之类的操作。搜索需要用到倒排索引，而排序和聚合则需要使用 "正排索引"。说白了就是一句话，倒排索引的优势在于查找包含某个项的文档，而反过来确定哪些项在单个文档里并不高效。

doc_values和fielddata就是用来给文档建立正排索引的。他俩一个很显著的区别是，前者的工作地盘主要在磁盘，而后者的工作地盘在内存。

我整理了一个表格，从不同维度比较这哥俩。

|维度| doc_values | fielddata
---|---|--
创建时间 | index时创建 | 使用时动态创建
创建位置 | 磁盘 | 内存(jvm heap)
优点 | 不占用内存空间 | 不占用磁盘空间
缺点 | 索引速度稍低 | 文档很多时，动态创建开销比较大，而且占内存

索引速度稍低这个是相对于fielddata方案的，其实仔细想想也可以理解。那排序举例，相对于一个在磁盘排序，一个在内存排序。谁的速度快自然不用多说。

在ES 1.x版本的官方说法是，

>Doc values are now only about 10–25% slower than in-memory fielddata

虽然速度稍慢，doc_values的优势还是非常明显的。一个很显著的点就是他不会随着文档的增多引起`OOM`问题。正如前面说的，doc_values在磁盘创建排序和聚合所需的正排索引。这样我们就避免了在生产环境给ES设置一个很大的`HEAP_SIZE`，也使得JVM的GC更加高效，这个又为其它的操作带来了间接的好处。

而且，随着ES版本的升级，对于doc_values的优化越来越好，索引的速度已经很接近fielddata了，而且我们知道硬盘的访问速度也是越来越快（比如SSD）。所以 doc_values 现在可以满足大部分场景，也是ES官方重点维护的对象。

所以我想说的是，doc values相比field data还是有很多优势的。所以 ES2.x 之后，支持聚合的字段属性默认都使用doc_values，而不是fielddata。



## 示例

光说不练假把式。上例子吧。我下面的例子是基于ES 7.1版本。

先准备一些数据，

```
PUT users
{
    "mappings" : {
      "properties" : {
        "name" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "keyword"
        },
        "age" : {
          "type" : "integer"
        }
      }
    }
}
```
这里我们新建了一个名为users的index，设置了mapping。然后插入一些测试数据，

```
PUT users/_doc/1
{
  "name":"tom",
  "mobile": "15978866921",
  "age": 30
}

PUT users/_doc/2
{
  "name":"jerry",
  "mobile": "15978866920",
  "age": 35
}

PUT users/_doc/3
{
  "name":"jack",
  "mobile": "15978866922",
  "age": 20
}
```

接着我们搜索的时候，基于年龄排序，

```
POST users/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

结果是正常的，

```
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "users",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "name" : "jerry",
          "mobile" : "15978866920",
          "age" : 35
        },
        "sort" : [
          35
        ]
      },
      {
        "_index" : "users",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "name" : "tom",
          "mobile" : "15978866921",
          "age" : 30
        },
        "sort" : [
          30
        ]
      },
      {
        "_index" : "users",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : null,
        "_source" : {
          "name" : "jack",
          "mobile" : "15978866922",
          "age" : 20
        },
        "sort" : [
          20
        ]
      }
    ]
  }
}
```

对于非text字段类型，doc_values默认情况下是打开的。索引我们上面的操作都没有问题。现在我们把doc_values关掉再试试。

由于修改mapping需要重建索引，为了简单起见，我这里把index直接删掉重建。

关闭doc_values,

```
DELETE users
PUT users
{
    "mappings" : {
      "properties" : {
        "name" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "keyword"
        },
        "age" : {
          "type" : "integer",
          "doc_values": false
        }
      }
    }
}
```

重新插入数据，然后再次搜索排序，发现报错，

![](http://pony-maggie.github.io/assets/images/2020/es/02/2-1.jpg)

意思就是`age`字段不支持排序了，需要打开doc_values才行。

fielddata现在用的不多，我就不演示了。


## 原理剖析

简单分析下doc_values的原理。我们知道搜索的时候需要使用倒排索引，类似下图这样，

![](http://pony-maggie.github.io/assets/images/2020/es/02/2-2.jpg)

比如说，所以我们要查找包含`brown`的文档，先在词项列表中找到 brown，然后扫描所有列，可以快速找到包含 brown 的文档。


但是如果是要对搜索结果进行排序或者其它聚合操作，倒排索引这种方式就没真这么容易了，反而是类下面这种正排索引更方便。doc_values其实是Lucene在构建倒排索引时，会额外建立一个有序的正排索引（基于document => field value的映射列表）。

![](http://pony-maggie.github.io/assets/images/2020/es/02/2-3.jpg)

## 总结

根据经验，大部分情况下你都不需要主动打开 fielddata，doc_values能满足大部分需求。

另外，mapping中的字段我们确定以后不会基于这个字段做排序或者聚合，可以把它关掉，但是一定要非常明确你的这个操作，因为如果要重新打开doc_values，需要重建索引，这个在生产环境下还是要谨慎。
