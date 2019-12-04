---
layout: post
title: 这可能是讲雪花算法最全的文章
category: tech
tags: [snowflake,雪花算法,workerid,ip地址]
---

* content
{:toc}

## 雪花算法的起源

snowflake中文的意思是 雪花，雪片，所以翻译成雪花算法。它最早是twitter内部使用的分布式环境下的唯一ID生成算法。在2014年开源。开源的版本由scala编写，大家可以再找个地址找到这版本。

https://github.com/twitter-archive/snowflake/tags

雪花算法产生的背景当然是twitter高并发环境下对唯一ID生成的需求，得益于twitter内部牛逼的技术，雪花算法流传至今并被广泛使用。它至少有如下几个特点：

* 能满足高并发分布式系统环境下ID不重复
* 基于时间戳，可以保证基本有序递增（有些业务场景对这个又要求）
* 不依赖第三方的库或者中间件
* 生成效率极高

## 雪花算法原理

![](http://pony-maggie.github.io/assets/images/2019/tech/12/snowflake-1.png)


雪花算法的原理其实非常简单，我觉得这也是该算法能广为流传的原因之一吧。

算法产生的是一个long型 64 比特位的值，第一位未使用。接下来是41位的毫秒单位的时间戳，我们可以计算下：
```
2^41/1000*60*60*24*365 = 69
```
也就是这个时间戳可以使用69年不重复，这个对于大部分系统够用了。

很多人这里会搞错概念，以为这个时间戳是相对于一个我们业务中指定的时间（一般是系统上线时间），而不是1970年。这里一定要注意。

10位的数据机器位，所以可以部署在1024个节点。

12位的序列，在毫秒的时间戳内计数。 支持每个节点每毫秒产生4096个ID序号，所以最大可以支持单节点差不多四百万的并发量，这个妥妥的够用了。


## 雪花算法java实现

雪花算法因为原理简单清晰，所以实现的话基本大同小异。下面这个版本跟网上的很多差别也不大。唯一就是我加了一些注释方便你理解。

```java
public class SnowflakeIdWorker {
    /** 开始时间截 (这个用自己业务系统上线的时间) */
    private final long twepoch = 1575365018000L;

    /** 机器id所占的位数 */
    private final long workerIdBits = 10L;

    /** 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数) */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    
    /** 序列在id中占的位数 */
    private final long sequenceBits = 12L;

    /** 机器ID向左移12位 */
    private final long workerIdShift = sequenceBits;

    /** 时间截向左移22位(10+12) */
    private final long timestampLeftShift = sequenceBits + workerIdBits;

    /** 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095) */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    /** 工作机器ID(0~1024) */
    private long workerId;

    /** 毫秒内序列(0~4095) */
    private long sequence = 0L;

    /** 上次生成ID的时间截 */
    private long lastTimestamp = -1L;

    //==============================Constructors=====================================
    /**
     * 构造函数
     * @param workerId 工作ID (0~1024)
     */
    public SnowflakeIdWorker(long workerId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("workerId can't be greater than %d or less than 0", maxWorkerId));
        }
        this.workerId = workerId;
    }

    // ==============================Methods==========================================
    /**
     * 获得下一个ID (该方法是线程安全的)
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long timestamp = timeGen();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }

        //上次生成ID的时间截
        lastTimestamp = timestamp;

        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - twepoch) << timestampLeftShift) //
                | (workerId << workerIdShift) //
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     * @return 当前时间(毫秒)
     */
    protected long timeGen() {
        return System.currentTimeMillis();
    }
}

```


可能有些人对于下面这个生成最大值得方法不太理解，我这里解释下，

```java
private final long sequenceMask = -1L ^ (-1L << sequenceBits);
```

首先我们要知道负数在计算机里是以补码的形式表达的，而补码是负数的绝对值的原码，再取得反码，然后再加1得到。

好像有点乱是吧，举个例子吧。

-1取绝对值是1，1的二进制表示，也就是原码是：
```
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
```

然后取反操作，也就是1变0; 0变1，得到：
```
11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111110
```

然后加1，得到：
```
11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
```

OK，这就是-1在计算机中的表示了，然后我们来看看(-1L << sequenceBits)，这个很简单，直接左移12个比特位即可，得到：

```
11111111 11111111 11111111 11111111 11111111 11111111 11110000 00000000
```

上面两个异或，得到：

```
00000000 00000000 00000000 00000000 00000000 00000000 00001111 11111111
```
也就是4095。

上面第一部分说到雪花算法的性能比较高，接下来我们测试下性能：

```java
public static void main(String[] args) {
        SnowflakeIdWorker idWorker = new SnowflakeIdWorker(1);

        long start = System.currentTimeMillis();
        int count = 0;
        for (int i = 0; System.currentTimeMillis()-start<1000; i++,count=i) {
            idWorker.nextId();
        }
        long end = System.currentTimeMillis()-start;
        System.out.println(end);
        System.out.println(count);
    }
```
用上面这段代码，在我自己的评估笔记本上，美妙可以产生400w+的id。效率还是相当高的。


## 一些细节讨论

### 调整比特位分布

很多公司会根据 snowflake 算法，根据自己的业务做二次改造。举个例子。你们公司的业务评估不需要运行69年，可能10年就够了。但是集群的节点可能会超过1024个，这种情况下，你就可以把时间戳调整成39bit，然后workerid调整为12比特。同时，workerid也可以拆分下，比如根据业务拆分或者根据机房拆分等。类似如下：

![](http://pony-maggie.github.io/assets/images/2019/tech/12/snowflake-2.png)

### workerid一般如何生成

方案有很多。比如我们公司以前用过通过jvm启动参数的方式传过来，应用启动的时候获取一个启动参数，保证每个节点启动的时候传入不同的启动参数即可。启动参数一般是通过-D选项传入，示例：
```
-Dname=value
```

然后我们在代码中可以通过
```java
System.getProperty("name");
```
获取，或者通过 `@value`注解也能拿到。

还有问题，现在很多部署都是基于k8s的容器化部署，这种方案往往是基于同一个yaml文件一键部署多个容器。所以没法通过上面的方法每个节点传入不同的启动参数。

这个问题可以通过在代码中根据一些规则计算workerid，比如根据节点的IP地址等。下面给出一个方案：

```java
private static long makeWorkerId() {
        try {
            String hostAddress = Inet4Address.getLocalHost().getHostAddress();
            int[] ips = StringUtils.toCodePoints(hostAddress);
            int sums = 0;
            for (int ip: ips) {
                sums += ip;
            }
            return (sums % 1024);
        } catch (UnknownHostException e) {
            return RandomUtils.nextLong(0, 1024);
        }
    }
```
这里其实是获取了节点的IP地址，然后把ip地址中的每个字节的ascii码值相加然后对最大值取模。当然这种方法是有可能产生重复的id的。

网上还有一些其它方案，比如取IP的后面10个比特位等。

总之不管用什么方案，都要尽量保证workerid不重复，否则即便是在并发量不高的情况下，也很容易出现id重复的情况。

