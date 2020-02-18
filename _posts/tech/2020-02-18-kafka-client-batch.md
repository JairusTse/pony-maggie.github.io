---
layout: post
title: 带你了解下Kafka的客户端缓冲池技术
category: tech
tags: [kafka,客户端,缓冲池,batch]
---

* content
{:toc}


最近看kafka源码，着实被它的客户端缓冲池技术优雅到了。忍不住要写篇文章赞美一下（哈哈）。

**注：本文用到的源码来自kafka2.2.2版本。**


## 背景

当我们应用程序调用kafka客户端 producer发送消息的时候，在kafka客户端内部，会把属于同一个topic分区的消息先汇总起来，形成一个batch。真正发往kafka服务器的消息都是以batch为单位的。如下图所示：

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-11.png)

这么做的好处显而易见。客户端和服务端通过网络通信，这样批量发送可以减少网络带来的性能开销，提高吞吐量。

这个Batch的管理就非常值得探讨了。可能有人会说，这不简单吗？用的时候分配一个块内存，发送完了释放不就行了吗。


kafka是用java语言编写的（新版本大部分都是用java实现的了），用上面的方案就是使用的时候new一个空间然后赋值给一个引用，释放的时候把引用置为null等JVM GC处理就可以了。

看起来似乎也没啥问题。但是在并发量比较高的时候就会频繁的进行GC。我们都知道GC的时候有个`stop the world`，尽管最新的GC技术这个时间已经非常短，依然有可能成为生产环境的性能瓶颈。

kafka的设计者当然能考虑到这一层。下面我们就来学习下kafka是如何对batch进行管理的。

## 缓冲池技术原理解析
kafka客户端使用了缓冲池的概念，预先分配好真实的内存块，放在池子里。

每个batch其实都对应了缓冲池中的一个内存空间，发送完消息之后，batch不再使用了，就把内存块归还给缓冲池。

听起来是不是很耳熟啊？不错，数据库连接池，线程池等池化技术其实差不多都是这样的原理。通过池化技术降低创建和销毁带来的开销，提升执行效率。

![](http://pony-maggie.github.io/assets/images/2020/java/02/2-12.png)

**代码是最好的文档，**，下面我们就来撸下源码。

我们撸代码的步骤采用的是从上往下的原则，先带你看看缓冲池在哪里使用，然后再深入到缓存池内部深入分析。

**下面的代码做了一些删减，值保留了跟本文相关的部分便于分析。**

```
public class KafkaProducer<K, V> implements Producer<K, V> {

    private final Logger log;
    private static final AtomicInteger PRODUCER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);
    private static final String JMX_PREFIX = "kafka.producer";
    public static final String NETWORK_THREAD_PREFIX = "kafka-producer-network-thread";
    public static final String PRODUCER_METRIC_GROUP_NAME = "producer-metrics";
    
    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
    }
    
    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {

            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                    serializedValue, headers, interceptCallback, remainingWaitMs);
                    ...
                    
    }
```
当我们调用客户端的发送消息的时候，底层会调用`doSend`，然后里面使用一个记录累计器`RecordAccumulator`把消息`append`进来。我们继续往下看看，

```
public final class RecordAccumulator {

    private final Logger log;
    private volatile boolean closed;
    private final AtomicInteger flushesInProgress;
    private final AtomicInteger appendsInProgress;
    private final int batchSize;
    private final CompressionType compression;
    private final int lingerMs;
    private final long retryBackoffMs;
    private final int deliveryTimeoutMs;
    private final BufferPool free;
    private final Time time;
    private final ApiVersions apiVersions;
    private final ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches;
    private final IncompleteBatches incomplete;
    // The following variables are only accessed by the sender thread, so we don't need to protect them.
    private final Map<TopicPartition, Long> muted;
    private int drainIndex;
    private final TransactionManager transactionManager;
    private long nextBatchExpiryTimeMs = Long.MAX_VALUE; // the earliest time (absolute) a batch will expire.
    
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        buffer = free.allocate(size, maxTimeToBlock);
        
        synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");

                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    return appendResult;
                }

                MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
                ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

                dq.addLast(batch);
        ...
```
`RecordAccumulator`其实就是管理一个batch队列，我们看到append方法实现其实是调用`BufferPool`的free方法申请（`allocate`）了一块内存空间(`ByteBuffer`)， 然后把这个内存空空间包装成batch添加到队列后面。

当消息发送完成不在使用batch的时候，`RecordAccumulator`会调用`deallocate`方法归还内存，内部其实是调用`BufferPool`的`deallocate`方法。

```
public void deallocate(ProducerBatch batch) {
        incomplete.remove(batch);
        // Only deallocate the batch if it is not a split batch because split batch are allocated outside the
        // buffer pool.
        if (!batch.isSplitBatch())
            free.deallocate(batch.buffer(), batch.initialCapacity());
    }
```

很明显，`BufferPool`就是缓冲池管理的类，也是我们今天要讨论的重点。我们先来看看分配内存块的方法。

```
public class BufferPool {

    static final String WAIT_TIME_SENSOR_NAME = "bufferpool-wait-time";

    private final long totalMemory;
    private final int poolableSize;
    private final ReentrantLock lock;
    private final Deque<ByteBuffer> free;
    private final Deque<Condition> waiters;
    /** Total available memory is the sum of nonPooledAvailableMemory and the number of byte buffers in free * poolableSize.  */
    private long nonPooledAvailableMemory;
    private final Metrics metrics;
    private final Time time;
    private final Sensor waitTime;

    public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");

        ByteBuffer buffer = null;
        this.lock.lock();
        try {
            // check if we have a free buffer of the right size pooled
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst();

            // now check if the request is immediately satisfiable with the
            // memory on hand or if we need to block
            int freeListSize = freeSize() * this.poolableSize;
            if (this.nonPooledAvailableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request, but need to allocate the buffer
                freeUp(size);
                this.nonPooledAvailableMemory -= size;
            } else {
                // we are out of memory and will have to block
                int accumulated = 0;
                Condition moreMemory = this.lock.newCondition();
                try {
                    long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                    this.waiters.addLast(moreMemory);
                    // loop over and over until we have a buffer or have reserved
                    // enough memory to allocate one
                    while (accumulated < size) {
                        long startWaitNs = time.nanoseconds();
                        long timeNs;
                        boolean waitingTimeElapsed;
                        try {
                            waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                        } finally {
                            long endWaitNs = time.nanoseconds();
                            timeNs = Math.max(0L, endWaitNs - startWaitNs);
                            recordWaitTime(timeNs);
                        }

                        if (waitingTimeElapsed) {
                            throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                        }

                        remainingTimeToBlockNs -= timeNs;

                        // check if we can satisfy this request from the free list,
                        // otherwise allocate memory
                        if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                            // just grab a buffer from the free list
                            buffer = this.free.pollFirst();
                            accumulated = size;
                        } else {
                            // we'll need to allocate memory, but we may only get
                            // part of what we need on this iteration
                            freeUp(size - accumulated);
                            int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
                            this.nonPooledAvailableMemory -= got;
                            accumulated += got;
                        }
                        ...
```
首先整个方法是加锁操作的，所以支持并发分配内存。

逻辑是这样的，当申请的内存大小等于`poolableSize`，则从缓存池中获取。这个`poolableSize`可以理解成是缓冲池的页大小，作为缓冲池分配的基本单位。从缓存池获取其实就是从ByteBuffer队列取出一个元素返回。

如果申请的内存不等于特定的数值，则向非缓存池申请。同时会从缓冲池中取一些内存并入到非缓冲池中。这个`nonPooledAvailableMemory`指的就是非缓冲池的可用内存大小。非缓冲池分配内存，其实就是调用`ByteBuffer.allocat`分配真实的JVM内存。


![](http://pony-maggie.github.io/assets/images/2020/java/02/2-13.png)

缓存池的内存一般都很少回收。而非缓存池的内存是使用后丢弃，然后等待`GC`回收。

继续来看看batch释放的代码，

```
public void deallocate(ByteBuffer buffer, int size) {
        lock.lock();
        try {
            if (size == this.poolableSize && size == buffer.capacity()) {
                buffer.clear();
                this.free.add(buffer);
            } else {
                this.nonPooledAvailableMemory += size;
            }
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock();
        }
    }
```
很简单，也是分为两种情况。要么直接归还缓冲池，要么就是更新非缓冲池部分的可以内存。然后通知等待队列里的第一个元素。


