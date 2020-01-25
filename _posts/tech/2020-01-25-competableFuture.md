---
layout: post
title: 详解 java CompletableFuture
category: tech
tags: [java,future,异步,CompletableFuture]
---

* content
{:toc}


## 背景知识

要理解 CompletableFuture，首先要弄懂什么是 Future。因为后者是前者的扩展。本文并不打算详细的介绍 Future，毕竟不是本文的重点。

Future是java1.5增加的一个接口，提供了一种异步并行计算的能力。比如说主线程需要执行一个复杂耗时的计算任务，我们可以通过future把这个任务放在独立的线程（池）中执行，然后主线程继续处理其他任务，处理完成后再通过Future获取计算结果。

这里通过一个简单的示例带你理解下 Future。

我们有两个服务，一个是用户服务可以获取用户信息，一个是地址服务，可以通过用户id获取地址信息。如下，

```java
@AllArgsConstructor
@Data
public class PersonInfo {
    private long id;
    private String name;
    private int age;
}
```

```java
public class PersonService {

    public PersonInfo getPersonInfo(Long personId) throws InterruptedException {
        Thread.sleep(500);//模拟调用耗时
        //真实项目中，这里大部分时候是通过dao从数据库获取
        return new PersonInfo(personId, "zhangsan", 30); //返回一条
    }
}
```

```java
@AllArgsConstructor
public class AddressInfo {
    private String addressLine;
    private String city;
    private String province;
    private String country;
}
```

```java
public class AddressService {

    public AddressInfo getAddress(long personId) throws InterruptedException {
        Thread.sleep(600); //模拟调用耗时
        System.out.println("id:" + personId);
        //真实项目中，这里大部分时候是通过dao从数据库获取
        return new AddressInfo("胜利大街143号", "北京市", "北京", "中国");
    }
}
```

然后我们演示下如何在主线程使用 Future 来进行异步调用。

```java
public class FutureTest {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        long startTime = System.currentTimeMillis();

        PersonService personService = new PersonService();
        AddressService addressService = new AddressService();
        long personId = 100L;

        //调用用户服务获取用户基本信息
        FutureTask<PersonInfo> personInfoFutureTask = new FutureTask<>(new Callable<PersonInfo>() {
            @Override
            public PersonInfo call() throws Exception {
                return personService.getPersonInfo(personId);
            }
        });
        new Thread(personInfoFutureTask).start();

        Thread.sleep(300); //模拟主线程其它操作耗时

        FutureTask<AddressInfo> addressInfoFutureTask = new FutureTask<>(new Callable<AddressInfo>() {
            @Override
            public AddressInfo call() throws Exception {
                return addressService.getAddress(personId);
            }
        });
        new Thread(addressInfoFutureTask).start();

        PersonInfo personInfo = personInfoFutureTask.get();//获取个人信息结果
        AddressInfo addressInfo = addressInfoFutureTask.get();//获取地址信息结果

        System.out.println("总共用时" + (System.currentTimeMillis() - startTime) + "ms");

    }
}
```

输出：
```
总共用时909ms
```

很明显，如果我们不使用 Future，而是在主线程串行的调用，耗时会是 `500 + 300 + 600 = 1400` 毫秒。通过 Future提供的异步计算功能，我们可以多个任务并行的执行，从而提高执行效率。

**我希望你能仔细的看上面的这个示例，因为后面讲到 CompletableFuture 我会使用同一个示例。**


## 基本介绍
通过上面的例子来看，似乎 Future 本身已经很强大了。那么 CompletableFuture 又是做啥的呢？

虽然Future以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。在上面的示例中，`personInfoFutureTask.get()` 就是阻塞调用，在线程获取结果之前get方法会一直阻塞。

轮询的方式在上面的示例中没有，其实也很简单。是Future提供了一个isDone方法，我们可以在程序中不断的轮询这个方法查询执行结果。

但是，无论是阻塞方式还是轮询方式，都不够好。

* 阻塞的方式和异步编程的初衷相违背
* 轮询的方式会耗费无谓的CPU资源

正是在这样的背景下，CompletableFuture 在java8横空出世。CompletableFuture提供了一种机制可以让任务执行完成后通知监听的一方，类似设计模式中的观察者模式。


## 使用示例

首先我们来看看，上面Future那个示例，如果是用 CompletableFuture 该怎么做？

我先给出代码，
```java
public class CompetableFutureTest {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        long startTime = System.currentTimeMillis();

        PersonService personService = new PersonService();
        AddressService addressService = new AddressService();
        long personId = 100L;

        CompletableFuture<PersonInfo> personInfoCompletableFuture = CompletableFuture.supplyAsync(() -> personService.getPersonInfo(personId));
        CompletableFuture<AddressInfo> addressInfoCompletableFuture = CompletableFuture.supplyAsync(() -> addressService.getAddress(personId));

        Thread.sleep(300); //模拟主线程其它操作耗时

        PersonInfo personInfo = personInfoCompletableFuture.get();//获取个人信息结果
        AddressInfo addressInfo = addressInfoCompletableFuture.get();//获取地址信息结果

        System.out.println("总共用时" + (System.currentTimeMillis() - startTime) + "ms");

    }
}
```

首先我们实现同样的功能代码简洁很多。supplyAsync 支持异步地执行我们指定的方法，这个例子中的异步执行方法是调用service。我们也可以使用 Executor 执行异步程序，默认是 ForkJoinPool.commonPool()。


另外通过这个示例，可以发现我们完全可以使用 CompletableFuture 代替 Future。


当然 CompletableFuture 的功能远不止与此，不然它的存在就没有意义了。CompletableFuture 提供了几十种方法辅助我们操作异步任务，用好了这些方法可以写出更加简洁，高效的代码。比如下面这个例子：

```
CompletableFuture<PersonInfo> personInfoCompletableFuture = CompletableFuture.supplyAsync(() -> personService.getPersonInfo(personId));
        CompletableFuture<AddressInfo> addressInfoCompletableFuture = CompletableFuture.supplyAsync(() -> addressService.getAddress(personId));


        final CompletableFuture<Void> completableFutureAllOf =
                CompletableFuture.allOf(personInfoCompletableFuture, addressInfoCompletableFuture);

        completableFutureAllOf.get(); //执行时间以最长那个任务为准

        PersonInfo personInfo = personInfoCompletableFuture.get();//马上返回
        AddressInfo addressInfo = addressInfoCompletableFuture.get();//马上返回

```

这个示例中，allOf可以让我们把多个异步任务结果的获取整合起来，这样操作更简单，代码更简洁。

前面提到了它可以解决的痛点，就是提供了一种类似观察者模式的机制，当异步的计算结果完成后可以通知监听者。下面来看个示例，

```java
CompletableFuture<PersonInfo> personInfoCompletableFuture = CompletableFuture.supplyAsync(() -> personService.getPersonInfo(personId));
        CompletableFuture<AddressInfo> addressInfoCompletableFuture = CompletableFuture.supplyAsync(() -> addressService.getAddress(personId));


        final CompletableFuture<Void> completableFutureAllOf =
                CompletableFuture.allOf(personInfoCompletableFuture, addressInfoCompletableFuture);

        //监听执行结果，整合两个任务的结果进一步处理
        final CompletableFuture<PersonAndAddress> personAndAddressCompletableFuture = completableFutureAllOf.thenApply((voidInput) ->
                new PersonAndAddress(personInfoCompletableFuture.join(), addressInfoCompletableFuture.join()));

        personAndAddressCompletableFuture.join();//以时间长的任务为准
```

在上面这个示例中，当两个异步任务执行完毕后，我们可以通过thenApply监听到结果并进行处理。

CompletableFuture 还有很多好玩有用的功能，如果感兴趣可以自行研究下。

## 总结

通过前面的讲解，你应该对 Future 以及它的扩展接口 CompletableFuture 都有了比较深入的认识。

我个人的建议是如果你的项目是基于java8，大部分情况你应该用后者而不是前者。如果你的项目是java8之前的版本，也建议你使用第三方的工具比如 Guava 等框架提供的Future工具类。

参考：

* https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html






