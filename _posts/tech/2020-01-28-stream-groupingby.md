---
layout: post
title: 关于Java使用groupingBy分组数据乱序问题
category: tech
tags: [groupingBy,hashmap,linkedhashmap,顺序,list]
---

* content
{:toc}


这是对最近做的一个项目，其中一个知识点的总结。

真实的业务场景就不说了，我来模拟下业务场景，足够说明问题就行了。

假设我有个对象，存储人员的基本信息，如下：

```java
@AllArgsConstructor
@Data
@ToString
public class PersonInfo {
    private String name;
    private int age;
    private int sex; //0表示男性，1表示女性
}
```

添加一些测试数据，

```java
        List<PersonInfo> personInfoList = new ArrayList<>();

        personInfoList.add(new PersonInfo("tom", 32, 1));
        personInfoList.add(new PersonInfo("jerry", 30, 1));
        personInfoList.add(new PersonInfo("alice", 15, 0));
        personInfoList.add(new PersonInfo("jack", 23, 1));
        personInfoList.add(new PersonInfo("aya", 32, 0));
```

打印下看看，
```java
        personInfoList.forEach(e -> System.out.println(e));

```

```
PersonInfo(name=tom, age=32, sex=1)
PersonInfo(name=jerry, age=30, sex=1)
PersonInfo(name=alice, age=15, sex=0)
PersonInfo(name=jack, age=23, sex=1)
PersonInfo(name=aya, age=32, sex=0)

```

我们可以看到，打印的顺序和我们添加的顺序是一样的。

现在我们有个需求，按照性别进行分组。

这个也不难，在 java8 环境下我们可以使用stream流的`groupingBy`很容易的实现，于是就有了下面的代码，

```java
        Map<Integer, List<PersonInfo>> map = personInfoList.stream().collect(Collectors.groupingBy(PersonInfo::getSex));

```

`groupingBy`实现类似SQL语句的“Group By”字句功能，实现根据一些属性进行分组并把结果存在Map实例。

打印结果看看是怎样的，

```java
        map.forEach((key, value) -> System.out.println(key + ": " + value));

```

```
0: [PersonInfo(name=alice, age=15, sex=0), PersonInfo(name=aya, age=32, sex=0)]
1: [PersonInfo(name=tom, age=32, sex=1), PersonInfo(name=jerry, age=30, sex=1), PersonInfo(name=jack, age=23, sex=1)]
```

结果确实是按照要求分组了，但是PersonInfo对象的顺序和我们之前添加的顺序不一样了，在有些业务场景下，这种结果往往是不允许的。（比如分页查询等）

为什么顺序会乱的？

我们先来看看这个map是哪个类型的，

```java
        System.out.println(map.getClass().getSimpleName());

```

打印出来的结果是 `HashMap`。

这个就解释了为啥顺序被打乱了， `HashMap`在存储是数据时，是先用key做hash计算，然后根据hash的结果决定这条数据的位置，因为hash本身是无序的，导致了读出的顺序是乱的。

知道了原因后，那就很容易想到解决方案了， `groupingBy`有一个重载的方法，允许我们指定map进行操作，

```java
Collector<T, ?, M> groupingBy(Function<? super T, ? extends K> classifier,
                                  Supplier<M> mapFactory,
                                  Collector<? super T, A, D> downstream)
```

注意看第二个参数， `Supplier`是java8提供的一中函数接口类型，用于提供一个对象， 根据尖括号里的定义，这里需要提供一个`Map`类型的对象。

**另外我们知道`HashMap`和`LinkedHashMap`的区别是后者通过双向链表保证数据插入的顺序和访问的顺序一致。** 于是就有了下面的代码，

```java
        Map<Integer, List<PersonInfo>> map = personInfoList.stream().collect(Collectors.groupingBy(PersonInfo::getSex, LinkedHashMap::new, Collectors.toList()));

```


打印的结果是，

```java
1: [PersonInfo(name=tom, age=32, sex=1), PersonInfo(name=jerry, age=30, sex=1), PersonInfo(name=jack, age=23, sex=1)]
0: [PersonInfo(name=alice, age=15, sex=0), PersonInfo(name=aya, age=32, sex=0)]
```

可以看到打印的顺序和原来的list顺序是一样的。


