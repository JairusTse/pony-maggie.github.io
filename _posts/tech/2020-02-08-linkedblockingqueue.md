---
layout: post
title: 你真的了解LinkedBlockingQueue的put,add和offer的区别吗
category: tech
tags: [put,offer,阻塞队列,kafka,LinkedBlockingQueue]
---

* content
{:toc}


## 概述
LinkedBlockingQueue的put,add和offer这三个方法功能很相似，都是往队列尾部添加一个元素。既然都是同样的功能，为啥要有有三个方法呢？

这三个方法的区别在于：

* put方法添加元素，如果队列已满，会阻塞直到有空间可以放
* add方法在添加元素的时候，若超出了度列的长度会直接抛出异常
* offer方法添加元素，如果队列已满，直接返回false

索引这三种不同的方法在队列满时，插入失败会有不同的表现形式，我们可以在不同的应用场景中选择合适的方法。


## 用法示例
先看看`add`方法，

```
public class LinkedBlockingQueueTest {

    public static void main(String[] args) throws InterruptedException {
        LinkedBlockingQueue<String> fruitQueue = new LinkedBlockingQueue<>(2);
        fruitQueue.add("apple");
        fruitQueue.add("orange");
        fruitQueue.add("berry");
    }

```

当我们执行这个方法的时候，会报下面的异常，

```
Exception in thread "main" java.lang.IllegalStateException: Queue full
  at java.util.AbstractQueue.add(AbstractQueue.java:98)
  at com.pony.app.LinkedBlockingQueueTest.testAdd(LinkedBlockingQueueTest.java:23)
  at com.pony.app.LinkedBlockingQueueTest.main(LinkedBlockingQueueTest.java:16)
```

然后再来看看`put`用法，

```
public class LinkedBlockingQueueTest implements Runnable {

    static LinkedBlockingQueue<String> fruitQueue = new LinkedBlockingQueue<>(2);


    public static void main(String[] args) throws InterruptedException {
        new Thread(new LinkedBlockingQueueTest()).start();

        fruitQueue.put("apple");
        fruitQueue.put("orange");
        fruitQueue.put("berry");

        System.out.println(fruitQueue.toString());

    }

    @Override
    public void run() {

        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        fruitQueue.poll();
    }
}

```

运行这段代码，你会发现首先程序会卡住（队列阻塞）3秒左右，然后打印队列的`orange`和`berry`两个元素。

因为我在程序的启动的时候顺便启动了一个线程，这个线程会在3秒后从队列头部移除一个元素。

最后看看`offer`的用法，

```
public static void main(String[] args) throws InterruptedException {
        LinkedBlockingQueue<String> fruitQueue = new LinkedBlockingQueue<>(2);
        
        System.out.println(fruitQueue.offer("apple"));
        System.out.println(fruitQueue.offer("orange"));
        System.out.println(fruitQueue.offer("berry"));

    }
```

运行结果：
```
true
true
false
```


## 源码分析
先来看看`add`方法的实现，

```
public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```
所以add其实是包装了一下offer，没什么可以说的。

然后来看看`put`和`offer`的实现，两个放在一起说。

put方法源码，
```
public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

```

offer方法源码，
```
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

我们重点关注他们的区别，offer方法在插入的时候会等一个超时时间`timeout`，如果时间到了队列还是满的（`count.get() == capacity`），就会返回false。

而put方法是无限期等待，
```
while (count.get() == capacity) {
                notFull.await();
            }
```

所以我们在应用层使用的时候，如果队列满再插入会阻塞。


## 实际场景应用

在早期版本的kafka中，生产者端发送消息使用了阻塞队列，代码如下：
```
private def asyncSend(messages: Seq[KeyedMessage[K,V]]) {
    for (message <- messages) {
      val added = config.queueEnqueueTimeoutMs match {
        case 0  =>
          queue.offer(message)
        case _  =>
          try {
            if (config.queueEnqueueTimeoutMs < 0) {
              queue.put(message)
              true
            } else {
              queue.offer(message, config.queueEnqueueTimeoutMs, TimeUnit.MILLISECONDS)
            }
          }
          catch {
            case _: InterruptedException =>
              false
          }
      }
      if(!added) {
        producerTopicStats.getProducerTopicStats(message.topic).droppedMessageRate.mark()
        producerTopicStats.getProducerAllTopicsStats.droppedMessageRate.mark()
        throw new QueueFullException("Event queue is full of unsent messages, could not send event: " + message.toString)
      }else {
        trace("Added to send queue an event: " + message.toString)
        trace("Remaining queue size: " + queue.remainingCapacity)
      }
    }
  }
```

可以看到，`config.queueEnqueueTimeoutMs`是0的时候，使用的是`offer`方法，小于0的时候则使用`put`方法。


我们在使用kafka的时候，可以通过`queue.enqueue.timeout.ms`来决定使用哪种方式。比如某些应用场景下，比如监控，物联网等场景，允许丢失一些消息，可以把`queue.enqueue.timeout.ms`配置成0，这样就kafka底层就不会出现阻塞了。


**新版的kafka(我印象中是2.0.0版本开始？)用java重写了，不再使用阻塞队列，所以没有上面说的问题。**

