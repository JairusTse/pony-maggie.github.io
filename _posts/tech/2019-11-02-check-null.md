---
layout: post
title: 如何优雅的判断一个对象的属性是否全部为空
category: tech
tags: [java,对象,属性,空]
---

* content
{:toc}

有一些业务场景下，我们需要判断某个对象的属性是否全部为空。该怎么做呢？

马上能想到的一个方案是，一个一个判断对象中的属性。这个倒也可以，但是如果要判断的对象比较多，就得给每个对象写一个判断方法（因为每个对象的属性都不一样）。

其实我们可以利用 java 的反射机制，比较优雅的实现。代码其实也很简单，
```java
public class ObjectIsNullUitl {
    public static boolean checkFieldAllNull(Object object) throws IllegalAccessException {
        for (Field f : object.getClass().getDeclaredFields()) {
            System.out.println("name:" + f.getName());
            f.setAccessible(true);
            if (Modifier.isFinal(f.getModifiers()) && Modifier.isStatic(f.getModifiers())) {
                continue;
            }
            if (!isEmpty(f.get(object))) {
                return false;
            }
            f.setAccessible(false);
        }
        //父类public属性
        for (Field f : object.getClass().getFields()) {
            System.out.println("name:" + f.getName());
            f.setAccessible(true);
            if (Modifier.isFinal(f.getModifiers()) && Modifier.isStatic(f.getModifiers())) {
                continue;
            }
            if (!isEmpty(f.get(object))) {
                return false;
            }
            f.setAccessible(false);
        }
        return true;
    }

    private static boolean isEmpty(Object object) {
        if (object == null) {
            return true;
        }
        if (object instanceof String && (object.toString().equals(""))) {
            return true;
        }
        if (object instanceof Collection && ((Collection) object).isEmpty()) {
            return true;
        }
        if (object instanceof Map && ((Map) object).isEmpty()) {
            return true;
        }
        if (object instanceof Object[] && ((Object[]) object).length == 0) {
            return true;
        }
        return false;
    }
}
```
简单说下原理，

isEmpty 方法除了对象本身的null判断之外，还会根据对象的实际类型特殊判断，比如String类型，大部分业务场景下空串（"")也是无意义的，和null可以等效处理。

另外，这里并没有加Number类型(Integer,Byte等包装类型的父类)，这个主要是考虑到不同的业务场景对于“空值”的定义不一样，不好统一处理。

```java
if (Modifier.isFinal(f.getModifiers()) && Modifier.isStatic(f.getModifiers())) {
                continue;
            }
```
这一句是让检查忽略掉 final static 修饰的属性，当然这个如果你的业务场景不需要，也可以不加。

然后我们准备一个测试类，
```java
public class Model extends BaseModel{
    private String property1;
    private Integer property2;
    private List<String> property3;
}
```

```java
public static void main(String[] args) throws InterruptedException, IllegalAccessException {
        Model model = new Model();
        boolean ret = ObjectIsNullUitl.checkFieldAllNull(model);
        System.out.println("ret:" + ret);
    }
```

输出的结果是true，因为我们确实没有给 model 对象的属性赋值。

这里其实有个坑需要特别注意。属性如果有基本类型（int，byte 等），即使不赋值，判断的结果也永远是 false。这是因为基本类型会有默认值（比如 int 默认值是0），在反射的过程中基本类型会变成包装类型，那么 int 就会变成 Integer 对象，并且对象的 intvalue 是0。所以需要判断是否为空的对象的属性尽量不要使用基本类型。
