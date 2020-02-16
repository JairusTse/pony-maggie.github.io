---
layout: post
title: 数据库连接池的原理没你想得这么复杂
category: tech
tags: [数据库,连接池,锁,wait,druid]
---

* content
{:toc}


## 背景介绍

数据库连接池和线程池等池技术存在的意义都是为了解决资源的重复利用问题。在计算机里，创建一个新的资源往往开销是非常大的。而池技术可以统一分配，管理某一类资源，它允许我们的程序可以重复的使用这个资源，只有在极端情况下（比如连接池满）才会创建新的资源。


数据库连接这种资源尤其昂贵，它的创建开销很大，大量的创建连接和释放操作对程序的影响非常明显。

数据库连接池正是针对这个问题提出来的。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-1.png)


## 实现原理

需要注意的是，我们下面提供的几种实现方式都是基于简单的原型，目的是带你了解连接池实现的一些基本原理。真实的数据库连接池技术需要考虑更多复杂的细节。

**所以下面这些代码都是不能在生产上直接使用的**。

实现的时候会用到`java.sql.Connection`，由于这个只是一个接口无法创建实例，为了演示方便，我继承这个接口写了一个简单的测试类，只是在`commit`方法里加了延时模拟提交。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-2.png)

### 实现方式1
很容易马上想到的一种方案，我们用一个map存放连接对象，需要的时候从map里拿来用就可以了。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-3.png)

这里需要主要，尽管我们使用了线程安全的`ConcurrentHashMap`来存放连接资源，`getConnection`方法依然要加上`synchronized`关键字来避免并发问题。这一点是最容易忽略的。

试想一下，假设在某个场景下，我们希望某个应用的多个线程共享连接资源。

假设有2个线程同时执行到了`pool.containsKey(key)`，然后都返回false，那这两个线程都会创建连接。虽然ConcurrentHashMap的put方法只会加入其中一个，但还是生成了1个多余的连接。

原因在于，尽管`ConcurrentHashMap`本身每个操作都是线程安全的，但是当这些操作组合在一起使用的时候，就无法保证原子性了，所以有可能带来并发的问题。

**这里友情提示下，面试经常会遇到这个考点哦。**

### 实现方式2

第二种实现方式是在1的基础上进行的优化。1的方案有个问题就是每次访问`getConnection`都要加锁，释放锁，效率比较低。

第二种方案是利用java并发包里的`Future`机制来解决并发场景创建多余连接的问题。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-4.png)

我们来捋一捋这个实现会不会有并发的问题。假设两个线程同时进入else分支，在代码的28行`ConcurrentHashMap`可以确保只有一个线程会执行，也就是只会加入一个task。其它的线程都不会加入成功。

所以只有一个线程`connectionFutureTask == null`，这个线程开始异步执行创建连接的任务，而其它的线程则会调用`FutureTask`的get方法直接获取结果。


### 实现方式3

1和2的实现方式还存在一个问题， 多个线程获取到的其实同一个连接。这种方案在某些场景下是不允许的。比如spring数据库的事务管理器对于每个事务的处理线程都要求独立的连接资源。

下面的方案基于链表结构，有比较完整的获取，释放的操作，不同的线程可以拿到独立的连接资源。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-5.png)

注意到这个方案我们在获取连接的时候引入了超时时间，如果该方法能够在一段时间内获取到结果，那么将结果立刻返回，反之，超时返回默认结果。

## druid连接池的实现原理
了解了实现连接池的大概思路，我们可以来继续学习下市面上比较成熟的连接池产品。这其中阿里巴巴开源的druid开源连接池就是一个代表。

Druid作为java领域最好的连接池技术之一，连接池本身只是它的一部分功能。除此之外，它还还要配套的监控功能。当然这个不是我们本文的重点。

先来看看在代码中如何使用Druid连接池，

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-6.png)

所以继续深入到是DataSource里的`getConnection`方法，

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-7.png)

`init`方法主要的功能是根据配置文件初始化连接池，它内部会生成一些真正的物理连接然后放入一个数组里。当然这个方法要保证只会被调用一次。

继续往下看，最终会调用到`getConnectionInternal`这个私有方法，

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-8.png)

红色圈出的部分是核心，根据传入的等待时间走不同的分支，我们来看看`takeLast`方法。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-9.png)

代码逻辑也比较清楚，poolCount是连接池的目前的可用连接数量。

如果为0，就通过`emptySignal`唤醒生产者线程创建新的连接，同时当前线程挂起等待`notEmpty`的信号。`notEmptyWaitCount`维护的就是正在等待的消费者数量。


如果不为0，就从数组中取出最后一个连接返回。有人可能会有疑问，这里返回的是`DruidConnectionHolder`，不是`Connection`啊？

其实看下前者的定义你就明白了，

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-10.png)

`DruidConnectionHolder`封装了`Connection`以及连接的datasource信息，还有多个statement等，方面进行统一管理。

-----------
参考
* 《java并发编程的艺术》
* https://www.cnblogs.com/cz123/p/7693064.html

