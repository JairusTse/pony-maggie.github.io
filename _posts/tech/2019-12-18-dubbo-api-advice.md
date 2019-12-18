---
layout: post
title: dubbo服务接口设计的几个建议
category: tech
tags: [dubbo,接口,参数,枚举,序列化]
---

* content
{:toc}

## 尽量不用独立的多个参数

比如我们有个dubbo的服务接口是这样定义的，
```java
public interface UserService {
    String sayHello1(String name);
}
```

服务实现示例如下：
```
public class UserServiceImpl implements UserService {
    public String sayHello1(String s) {
        return "hello1 " + s + "!";
    }
```

然后我们中间服务进行升级，需要增加一个入参，把接口改成了如下的方式：

```java
public interface UserService {
    String sayHello1(String name, int age);
}
```

服务实现改成了：
```
public class UserServiceImpl implements UserService {
    public String sayHello1(String s, int age) {
        return "hello1 " + s + age + "!";
    }
```

服务上线后，调用方肯定会报错，因为整个接口声明都变了，相当于两个完全不同的接口。

那如何解决呢？其实很简单。服务接口的参数类型最好是封装类，增加参数的话只是在这个类增加一个字段。示例如下：
```java
public interface UserService {
    String sayHello1(String name);
    String sayHello2(SayWorldRequest request);
```

封装类的定义：
```java
public class SayWorldRequest implements Serializable {
    private static final long serialVersionUID = 1L;

    String name;
    int age;
    ...
```

## 接口最好带有版本信息

当一个接口实现，出现不兼容升级时很有用。可以用版本号过渡，版本号不同的服务相互间不引用。

还是通过示例来说明：

假设服务端的要对接口sayHello1重写，和原来的方法不兼容，我们重写写个实现类，然后实现sayHello1方法，
```java
public class UserServiceImpl2 implements UserService {
    public String sayHello1(String s) {
        return "hello1 new" + s + "!";
    }
    
```

然后暴露服务的时候指定不同的版本，
```xml
<bean id="userService" class="com.dubbo.provider.UserServiceImpl"/>
    <bean id="userService2" class="com.dubbo.provider.UserServiceImpl2"/>
    <dubbo:service interface="com.dubbo.api.UserService" ref="userService" version="1.0"/>
    <dubbo:service interface="com.dubbo.api.UserService" ref="userService2" version="1.1"/>
```

客户端调用的时候，可以通过指定版本号调用不同的服务，
```xml
    <!-- 引用服务配置 -->
    <dubbo:reference id="userServiceClient" interface="com.dubbo.api.UserService" cluster="failfast" check="false" version="1.1"/>

```

聪明如你可能想到了，利用版本号的机制可以实现灰度发布，举个例子：

```xml
<!-- 机器A提供1.0.0版本服务 -->
<dubbo:service interface="com.dubbo.api.UserService" version="1.0.0" />
<!-- 机器B提供2.0.0版本服务 -->
<dubbo:service interface="com.dubbo.api.UserService" version="2.0.0" />
<!-- 机器C消费1.0.0版本服务 -->
<dubbo:reference id="userService" interface="com.dubbo.api.UserService" version="1.0.0" />
<!-- 机器D消费2.0.0版本服务 -->
<dubbo:reference id="userService" interface="com.dubbo.api.UserService" version="2.0.0" />

```


## 尽量少用枚举类型作为参数

《阿里巴巴java开发手册》有这么一条规定：

>【强制】二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚
举类型或者包含枚举类型的 POJO 对象。

为什么有这样的规定呢？

先给你看个例子：

接口定义，
```java
public interface UserService {
    SayWorldResponse sayHello3(SayWorldRequest request);
}
```

响应类定义，
```java
public class SayWorldResponse implements Serializable {
    private static final long serialVersionUID = 1L;

    String name;
    int age;
    EnumColor enumColor;
    
    //省略其它部分
    
```

枚举定义，
```java
public enum EnumColor {
    RED("red", 1),
    GREEN("green", 2),
    BLACK("black", 3),
    YELLOW("yellow", 4);
    
    //省略其它部分
```

服务实现，
```java
public class UserServiceImpl implements UserService {
   //省略其它部分

    public SayWorldResponse sayHello3(SayWorldRequest request) {
        SayWorldResponse response = new SayWorldResponse();
        response.setAge(100);
        response.setName("response");
        response.setEnumColor(EnumColor.YELLOW);
        return response;
    }
}
```

然后系统上线，客户端正常调用，一切风平浪静。

但是软件总会升级，过了一段时间，我们需要在枚举里新增一个类型，并且服务端在某些场景下会使用这个字段返回给客户端。代码如下：

```java
public enum EnumColor {
    RED("red", 1),
    GREEN("green", 2),
    BLACK("black", 3),
    YELLOW("yellow", 4),
    BLACK("black", 5);
    
    //省略其它部分
```

服务实现，
```java
public class UserServiceImpl implements UserService {
   //省略其它部分

    public SayWorldResponse sayHello3(SayWorldRequest request) {
        SayWorldResponse response = new SayWorldResponse();
        response.setAge(100);
        response.setName("response");
        if (条件成立) {
            response.setEnumColor(EnumColor.BLACK);

        } else {
            response.setEnumColor(EnumColor.YELLOW);

        }
        return response;
    }
}
```

这个时候就相当于埋下了一颗雷，服务端上线后，当客户端调用sayHello3方法触发条件返回 EnunColor.BLACK时，客户端就会报如下的错误：

```xml
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is com.alibaba.dubbo.rpc.RpcException: Failfast invoke providers dubbo://10.204.246.56:28511/com.dubbo.api.UserService2?anyhost=true&application=dubbo-consumer&check=false&cluster=failfast&default.check=false&default.cluster=failfast&dubbo=2.5.3&interface=com.dubbo.api.UserService&methods=sayHello1,sayHello2,sayHello3&pid=18056&revision=1.1-SNAPSHOT&side=consumer&timestamp=1576573801502&version=1.1 RandomLoadBalance select from all providers [com.alibaba.dubbo.registry.integration.RegistryDirectory$InvokerDelegete@23c21e07] for service com.dubbo.api.UserService method sayHello3 on consumer 10.204.246.56 use dubbo version 2.5.3, but no luck to perform the invocation. Last error is: Failed to invoke remote method: sayHello3, provider: dubbo://10.204.246.56:28511/com.dubbo.api.UserService2?anyhost=true&application=dubbo-consumer&check=false&cluster=failfast&default.check=false&default.cluster=failfast&dubbo=2.5.3&interface=com.dubbo.api.UserService&methods=sayHello1,sayHello2,sayHello3&pid=18056&revision=1.1-SNAPSHOT&side=consumer&timestamp=1576573801502&version=1.1, cause: com.alibaba.com.caucho.hessian.io.HessianFieldException: com.dubbo.api.SayWorldResponse.enumColor: java.lang.reflect.InvocationTargetException
com.alibaba.com.caucho.hessian.io.HessianFieldException: com.dubbo.api.SayWorldResponse.enumColor: java.lang.reflect.InvocationTargetException
  at com.alibaba.com.caucho.hessian.io.JavaDeserializer.logDeserializeError(JavaDeserializer.java:671)
  at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectFieldDeserializer.deserialize(JavaDeserializer.java:400)
  at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:233)
  at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:157)
  at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2067)
  at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1592)
  at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1576)
  ...

```

很明显，这是一个RPC调用过程中反序列的错误。为什么会有这个错误呢？解释下：

在Java中，对Enum类型的序列化与其他对象类型的序列化有所不同的，官方的说明是这样的：

>Enum constants are serialized differently than ordinary serializable or externalizable objects. The serialized form of an enum constant consists solely of its name; field values of the constant are not present in the form. To serialize an enum constant, ObjectOutputStream writes the value returned by the enum constant's name method. To deserialize an enum constant, ObjectInputStream reads the constant name from the stream; the deserialized constant is then obtained by calling the java.lang.Enum.valueOf method, passing the constant's enum type along with the received constant name as arguments. Like other serializable or externalizable objects, enum constants can function as the targets of back references appearing subsequently in the serialization stream. 
The process by which enum constants are serialized cannot be customized: any class-specific writeObject, readObject, readObjectNoData, writeReplace, and readResolve methods defined by enum types are ignored during serialization and deserialization. Similarly, any serialPersistentFields or serialVersionUID field declarations are also ignored--all enum types have a fixedserialVersionUID of 0L. Documenting serializable fields and data for enum types is unnecessary, since there is no variation in the type of data sent. 

大概意思就是说，在序列化的时候Java仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过java.lang.Enum的valueOf方法来根据名字查找枚举对象。同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法。


所以，在远程方法调用过程中，如果我们发布的客户端接口返回值中使用了枚举类型，那么服务端在升级过程中比如在接口的返回结果的枚举类型中添加了新的枚举值，那就会导致仍然在使用老的客户端的那些应用出现调用失败的情况。

那么有没有解决办法呢？当然，方法就是客户端和服务端都升级，都引用最新的API声明。不过往往实际的项目中很难保证这一点（不同的团队维护），所以我们还是要尽量避免RPC接口的返回值里面包含枚举定义。


## 总结

1. 接口定义尽量使用封装类作为入参，避免日后需要新增参数带来不便
2. 暴露服务尽量使用版本约束，方便以后升级
3. 接口的返回值尽量不适用枚举，否则容易引起反序列化的问题


