---
layout: post
title: ES系列之一文带你避开日期类型存在的坑
category: elasticsearch
tags: [elasticsearch,java,时区,日期]
---

* content
{:toc}


## 概述
时间相关的字段是ElasticsSearch（以下简称ES）最常用的字段了，几乎所有的索引应用场景都会有时间字段，一般用于基于时间范围的搜索，聚合等场景。但是由于时区的问题，相信很多小伙伴都踩到过时间字段的坑，笔者自己就踩过。

本文希望给你提供一个避坑指南。

## 了解时区的基本概念
因为本文不是专门讲时区的，你只需要了解一些基本的概念就可以了。

![](http://pony-maggie.github.io/assets/images/2020/es/03/2-1.png)

我们知道全球分为24个时区，包含23个整时区及180°经线左右两侧的2个半时区。东经的时间比西经要早，也就是如果格林威治时间是中午12时，则中央经线15°E的时区为下午1时。比如北京位于东8区，所以北京时间应该是晚上8点。


* 格林威治标准时间GMT或者UTC

GMT和UTC可以认为是一个东西，只是精度的差异。他们代表的是全球的一个时间参考点，全球都以格林威治的时间作为标准来设定时间。

在程序中我们经常能见到这样的字符串：
```
Thu Oct 16 07:13:48 GMT 2019
```
这说明这个时间是GMT时间。

* CST中国标准时间

China Standard Time，是中国的标准时间。CST = GMT(UTC) + 8。比如

```
Thu Aug 25 17:15:49 CST 2019
```
表示的就是CST时间。有时候我们也能见到类似下面这样的表示：
```
2020-03-15T11:45:43Z
```
其中Z表示的就是UTC时间。

## 坑一，日期字段映射问题
我们知道ES有个Dynamic Mapping的机制，当索引不存在或者索引中的某些字段没有设置mapping属性，index的时候ES会自动创建索引并且根据传入的字段内容自动推断字段的格式。比如，整型的数字会变成Long，“yyyy-dd-mm”等格式的字段会转成date )，不过有时候这个推断并不是我们想要的。

举个我自己在项目中遇到的例子。当时有个实体对象要写入ES中，我用了fastjson转换成json的字符串然后写入ES。在ES查看的时候发现写入的字段变成了Long型失去了日期的属性，导致不能根据此字段进行日期相关的条件搜索。下面模拟下整个过程。

首先定义一个实体对象，
```
@Data
@ToString
public class TestEntity {
    private String stringData;
    private Byte byteData;
    private Date timeData;
}
```

然后写入整个对象，

```
TestEntity entity = new TestEntity();
entity.setByteData((byte)2);
entity.setStringData("test");
entity.setTimeData(new Date());
IndexRequest request = new IndexRequest("test_index");
request.id(id);
request.source(JSON.toJSONString(), XContentType.JSON);
client.index(request, RequestOptions.DEFAULT);
```

写入成功后发现无法根据整个时间字段进行排序和筛选，在ES里查看索引的mapping发现，`timeData`字段居然被识别成了Long型。

![](http://pony-maggie.github.io/assets/images/2020/es/03/2-2.png)

原因是fastjson默认把Date类型转换成long型的时间戳了。到ES这边以为是一个普通的整型。

这个问题的解决方案有两种。

第一种是在fastjson序列化的时候不要使用默认行为，而是指定日期类型的格式，
```
@Data
@ToString
public class TestEntity {
    private String stringData;
    private Byte byteData;

    @JSONField(format="yyyy-MM-dd HH:mm:ss")
    private Date timeData;
}

```
这样写进ES就会被自动识别成日期类型。

另一种解决方案是，在ES的maping里明确的指定字段的属性。
```
PUT test_index
{
  "mappings": {
    "properties": {
      "TimeData": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
```
这里我们给`TimeData`设置了日期类型，并且可以识别三种不同的日期格式。其中最后一个`epoch_millis`就是毫秒单位的时间戳。

## 坑二，时区问题

这个坑最常见。比如很多时候我们是直接把mysql的数据读出然后写入到ES。mysql里的日期写入到ES后发现时间ES查询的时间跟实际看到的时间差了8个小时，究竟是怎么回事呢？

先来看看官方文档怎么说，

>Internally, dates are converted to UTC (if the time-zone is specified) and stored as a long number representing milliseconds-since-the-epoch.

>Queries on dates are internally converted to range queries on this long representation, and the result of aggregations and stored fields is converted back to a string depending on the date format that is associated with the field.

这两段的意思是说，在ES内部默认使用UTC时间并且是以毫秒时间戳的long型存储的。针对日期字段的查询其实对long型时间戳的范围查询。

我们举一个例子，很多时候我们会把mysql的数据同步的ES，方法很多，我这里以用logstash迁移数据举例。（关于logstash具体的配置方法不是本文的重点我就不表了）mysql的数据是这样的：

![](http://pony-maggie.github.io/assets/images/2020/es/03/2-3.png)

logstash的配置如下：（只给出部分配置）

```
input {
  jdbc {
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/test"
    jdbc_user => "root"
    jdbc_password => "11111111"
    use_column_value => false
    #记录最后一次运行的结果
    record_last_run => true
    #上面运行结果的保存位置
    last_run_metadata_path => "jdbc-position.txt"
    statement => "SELECT * FROM kafkalogin"
```
执行logstash进行迁移，然后我们在kibana里发现数据是这样的：

![](http://pony-maggie.github.io/assets/images/2020/es/03/2-4.png)

很奇怪，似乎相差的时间也不是8个小时，而是5个小时或者6个小时。

这种问题我们的解决方案也很简单。我们已经知道输出端（ES）的默认时区是UTC，只需要再在输入端（mysql）也明确时区即可。改下logstash的配置如下：
```
input {
  jdbc {
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
```
然后你就会发现两边的时间是一样的。

如果你的mysql里的时间不是UTC而是东八区的时间，可以用如下的配置：
```
input {
  jdbc {
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=Asia/Shanghai"
```

这样迁移的数据在ES里查看是相差8个小时的。

还有一种解决方案是你存储的时间字符串本身就带有时区信息，比如 “2016-07-15T12:58:17.136+0800”。


我们在ES进行查询或者聚合的时候，建议指定时区避免产生意想不到的结果。比如：

```
GET _search
{
    "query": {
        "range" : {
            "timestamp" : {
                "time_zone": "+01:00", 
                "gte": "2015-01-01 00:00:00", 
                "lte": "now" 
            }
        }
    }
}
```
加上这个时区信息，ES在搜索的时候时间起始就是`2014-12-31T23:00:00 UTC`。

此外在使用Java Client聚合查询日期的时候，也需要注意时区问题，最好是指定时区进行搜索或者聚合。