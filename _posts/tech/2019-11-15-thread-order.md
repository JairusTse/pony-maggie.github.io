---
layout: post
title: 带你了解控制线程执行顺序的几种方法
category: tech
tags: [线程,join,countdownlatch,顺序]
---

* content
{:toc}

## 基本介绍

通常情况下，线程的执行顺序都是随机的，哪个获取到CPU的时间片，哪个就获得执行的机会。不过实际的项目中有时我们会有需要不同的线程顺序执行的需求。借助一些java中的线程阻塞和同步机制，我们往往也可以控制多个线程的执行顺序。

方法有很多种，本篇文章介绍几种常用的。

## 利用 thread join实现线程顺序执行

thread.join方法的可以实现如下的效果，就是挂起调用join方法的线程的执行，直到被调用的线程执行结束。听起来有点绕，举个例子解释下：

假设有t1, t2两个线程，如果在t2的线程流程中调用了 `t1.join`， 那么t2线程将会停止执行，等待t1执行结束后才会继续执行。

很显然，利用这个机制，我们可以控制线程的执行顺序，看下面的例子：

```java

public class ControlThreadDemo {

    public static void main(String[] args) {

        Thread previousThread = Thread.currentThread();

        for (int i = 0; i < 10; i++) {
            ThreadJoinDemo threadJoinDemo = new ThreadJoinDemo(previousThread);
            threadJoinDemo.start();
            previousThread = threadJoinDemo;
        }

        System.out.println("主线程执行完毕");

    }
}


public class ThreadJoinDemo extends Thread{

    private Thread previousThread;
    public ThreadJoinDemo(Thread thread) {
        this.previousThread = thread;
    }

    public void run() {
        try {
            System.out.println("线程:" + Thread.currentThread().getName() + " 等待 " + previousThread.getName());
            previousThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "开始执行");
    }


}
```

运行结果：

```bash
线程:Thread-1 等待 Thread-0
线程:Thread-5 等待 Thread-4
线程:Thread-4 等待 Thread-3
线程:Thread-3 等待 Thread-2
线程:Thread-2 等待 Thread-1
线程:Thread-0 等待 main
线程:Thread-8 等待 Thread-7
线程:Thread-7 等待 Thread-6
线程:Thread-6 等待 Thread-5
线程:Thread-9 等待 Thread-8
主线程执行完毕
Thread-0开始执行
Thread-1开始执行
Thread-2开始执行
Thread-3开始执行
Thread-4开始执行
Thread-5开始执行
Thread-6开始执行
Thread-7开始执行
Thread-8开始执行
Thread-9开始执行
```

从执行结果可以很容易理解， 程序运行起来之后，一共11个线程排好队等着执行，排在最前面的是 main 线程，然后依次是t0, t1 ...。

## 利用 CountDownLatch 控制线程的执行顺序

还是先说下 CountDownLatch 的用法，CountDownLatch 是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程执行完后再执行。借用一张经典的图：

![](http://pony-maggie.github.io/assets/images/2019/tech/11/thread/countdown.png)

CountDownLatch提供两个核心的方法，countDown和await，后者可以阻塞调用它的线程， 而前者每调用一次，计数器减去1，当计数器减到0的时候，阻塞的线程被唤醒继续执行。

### 场景1
先看一个例子，在这个例子中，主线程会等有若干个子线程执行完毕之后再执行，不过这若干个子线程之间的执行顺序是随机的。

```java
public class ControlThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(5);
        List<Thread> workers = Stream.generate(() -> new Thread(new CountDownDemo(countDownLatch))).limit(5).collect(Collectors.toList());
        workers.forEach(Thread::start);
        countDownLatch.await();

        System.out.println("主线程执行完毕");

    }
}


public class CountDownDemo implements Runnable{
    private CountDownLatch countDownLatch;

    public CountDownDemo(CountDownLatch latch) {
        this.countDownLatch = latch;
    }

    @Override
    public void run() {
        System.out.println("线程" + Thread.currentThread().getName() + "开始执行");
        countDownLatch.countDown();
    }
}

```

输出，
```bash
线程Thread-0开始执行
线程Thread-3开始执行
线程Thread-2开始执行
线程Thread-1开始执行
线程Thread-4开始执行
主线程执行完毕
```


这种场景在实际项目中有需要的场景，比如我之前看过一个案例，大概的场景是说需要下载一个大文件，开启多个线程分别下载文件的一部分，然后有一个线程最后拼接所有的文件。我们可以考虑使用 CountDownLatch 来控制并发，使拼接的线程放在最后执行。


### 场景2

这个案例带你了解下利用 CountDownLatch 控制一组线程一起执行。就好像在运动场上，教练的发令枪一响，所有运动员一起跑。我们一般在模拟线程并发执行的时候会用到这种场景。

我们把场景1的代码稍微改造一下，

```java
public class ControlThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch readyLatch = new CountDownLatch(5);
        CountDownLatch runningLatchWait = new CountDownLatch(1);
        CountDownLatch completeLatch = new CountDownLatch(5);


        List<Thread> workers = Stream.generate(() -> new Thread(new CountDownDemo2(readyLatch,runningLatchWait,completeLatch))).limit(5).collect(Collectors.toList());
        workers.forEach(Thread::start);
        readyLatch.await();//等待发令
        runningLatchWait.countDown();//发令
        completeLatch.await();//等所有子线程执行完

        System.out.println("主线程执行完毕");

    }
}

public class CountDownDemo2 implements Runnable{
    private CountDownLatch readyLatch;
    private CountDownLatch runningLatchWait;
    private CountDownLatch completeLatch;

    public CountDownDemo2(CountDownLatch readyLatch, CountDownLatch runningLatchWait, CountDownLatch completeLatch) {
        this.readyLatch = readyLatch;
        this.runningLatchWait = runningLatchWait;
        this.completeLatch = completeLatch;
    }

    @Override
    public void run() {
        readyLatch.countDown();
        try {
            runningLatchWait.await();
            System.out.println("线程" + Thread.currentThread().getName() + "开始执行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            completeLatch.countDown();
        }
    }
}


```



### 场景3

到这里，可能很多人会想问，利用 CountDownLatch 能做到像前面thread.join控制多个线程按照一个固定的先后顺序执行吗？ 

首先我要说，用 CountDownLatch 实现这种场景确实不多见，不过也不是不可以做。请继续看场景3。

```java
public class ControlThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch first = new CountDownLatch(1);
        CountDownLatch prev = first;

        for (int i = 0; i < 10; i++) {
            CountDownLatch next = new CountDownLatch(1);
            new CountDownDemo3(prev, next).start();
            prev = next;
        }
        first.countDown();
    }
}


public class CountDownDemo3 extends Thread{
    private CountDownLatch prev;
    private CountDownLatch next;

    public CountDownDemo3(CountDownLatch prev, CountDownLatch next) {
        this.prev = prev;
        this.next = next;
    }

    @Override
    public void run() {
        try {
            prev.await();
            Thread.sleep(1000);//模拟线程执行耗时
            System.out.println("线程" + Thread.currentThread().getName() + "开始执行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            next.countDown();
        }
    }
}
```

输出，
```bash
线程Thread-0开始执行
线程Thread-1开始执行
线程Thread-2开始执行
线程Thread-3开始执行
线程Thread-4开始执行
线程Thread-5开始执行
线程Thread-6开始执行
线程Thread-7开始执行
线程Thread-8开始执行
线程Thread-9开始执行

```

代码也不难理解，for循环里把10个线程串联起来，排好队等着执行。排在最前面的线程t1在等first这个计数器countDown，然后t1开始执行，执行完调用自己的next计数器 countDown 以唤醒下一个，依次类推。


## 利用 newSingleThreadExecutor 控制线程的执行顺序

java的 Executors 线程池平时工作中用得很多了，JAVA通过Executors提供了四种线程池，单线程化线程池(newSingleThreadExecutor)、可控最大并发数线程池(newFixedThreadPool)、可回收缓存线程池(newCachedThreadPool)、支持定时与周期性任务的线程池(newScheduledThreadPool)。

顾名思义，newSingleThreadExecutor 线程池只有一个线程。它存在的意义就在于控制线程执行的顺序，保证任务的执行顺序和提交顺序一致。其实保证顺序执行的原理也很简单，因为总是只有一个线程处理任务队列上的任务，先提交的任务必将被先处理。

废话不多说，上代码。

```java
public static void main(String[] args) throws InterruptedException {
        final ExecutorService executorService = Executors.newSingleThreadExecutor();

        for (int i = 0; i < 5; i++){
            final int index = i;
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    Thread.currentThread().setName("thread-" + index);
                    System.out.println("线程: " + Thread.currentThread().getName() + " 开始执行");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }

        executorService.awaitTermination(30, TimeUnit.SECONDS);
        executorService.shutdownNow();
    }
```

输出，
```bash
线程: thread-0 开始执行
线程: thread-1 开始执行
线程: thread-2 开始执行
线程: thread-3 开始执行
线程: thread-4 开始执行
线程: thread-5 开始执行
线程: thread-6 开始执行
线程: thread-7 开始执行
线程: thread-8 开始执行
线程: thread-9 开始执行
```

---------

**参考**

* https://www.baeldung.com/java-countdown-latch


