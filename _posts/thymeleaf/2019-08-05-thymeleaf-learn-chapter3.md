---
layout: post
title:  跟我学spring boot thymeleaf系列文章第三章 thymeleaf表单进阶
category: thymeleaf
tags: [java,springboot,thymeleaf,表单,表达式]
---

* content
{:toc}


源码地址:

https://github.com/pony-maggie/spring-boot-thymeleaf-learn


本章介绍下thymeleaf表单一些比较复杂的操作。

## 编码

我们先实现一个测试controller，

```java
@RestController
public class UserController {
    private List<User> userList = new ArrayList<>();

    {
        userList.add(new User("1", "socks", "123456", new Date()));
        userList.add(new User("2", "admin", "111111", new Date()));
        userList.add(new User("3", "jacks", "222222", null));
    }

    @GetMapping("/")
    public ModelAndView index() {
        return new ModelAndView("user/user", "userList", userList);
    }

    @GetMapping("/user/{userId}")
    public ModelAndView userForm(@PathVariable String userId) {
        return new ModelAndView("user/userForm", "user", userList.get(0));
    }

}
```

当用户在浏览器输入"/"时，我们通过user.html页面展示所有用户的信息，当输入类似"/usre/id时，我们通过userForm.html展示某个id用户信息，展示效果我们用表单来做。

前端页面，我们先定义一个公共的头部页面， head.html，然后定义两个前面说到user.html和userForm.html文件。先把代码贴出来再说知识点。

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<!--声明static为页面片段名称-->
<head th:fragment="static">
    <link th:href="@{/webjars/bootstrap/css/bootstrap.css}" rel="stylesheet" type="text/css"/>
    <script th:src="@{/webjars/jquery/jquery.min.js}"></script>
    <link th:href="@{/webjars/bootstrap/js/bootstrap.min.js}" rel="stylesheet" type="text/css"/>
</head>
</html>
```

user.html

```html
<head>
    <meta charset="UTF-8"/>
    <title th:text="用户列表">User</title>
    <!--默认拼接前缀路径,开头请勿再添加斜杠,防止部署运行报错！-->
    <script th:replace="common/head::static"></script>
</head>
<body>
<h3>用户列表</h3>
<div th:each="user,userStat:${userList}" th:class="${userStat.even}?'even':'odd'">
    序号：<input type="text" th:value="${userStat.count}"/>
    账号：<input type="text" th:value="${user.username}"/>
    密码：<input type="password" th:value="${user.password}"/>
    时间：<input type="text" th:value="${user.createTime}"/>
    时间：<input type="text" th:value="${#dates.format(user.createTime,'yyyy-MM-dd HH:mm:ss')}"/>
</div>

<script th:inline="javascript">
    //通过内联表达式获取用户信息
    var userList = [[${userList}]];
    console.log(userList)
</script>
</body>
</html>
```

userForm.html

```html
<head>
    <meta charset="UTF-8"/>
    <title th:text="用户信息">User</title>
    <script th:replace="common/head::static"></script>
</head>
<body>
<h3>用户表单</h3>
<form class="form-horizontal" th:object="${user}">
    <input type="hidden" id="id" name="id" th:value="*{id}">
    <div class="form-group">
        <label class="col-md-2 control-label">账号:</label>
        <div class="col-md-4">
            <input class="form-control" id="username" name="username" th:value="*{username}"/>
        </div>
    </div>
    <div class="form-group">
        <label class="col-md-2 control-label">密码:</label>
        <div class="col-md-4">
            <input class="form-control" id="password" name="password" th:value="*{password}" type="password"/>
        </div>
    </div>
    <div class="form-group">
        <label class="col-md-2 control-label">时间:</label>
        <div class="col-md-4">
            <input class="form-control" id="createTime" name="createTime" th:value="*{#dates.format(createTime,'yyyy-MM-dd HH:mm:ss')}" />
        </div>
    </div>
</form>

</body>
```

head.html中用到了fragment这个关键字。 通过th:fragment定制片段 ，然后通过th:replace 填写片段路径和片段名。说白了就是一个文件引用另外一个文件的语法，但是它的强大在于颗粒度比较细，可以细到标签级别。

我们在head.html中通过链接表达式(@{})，引入一些本地的静态资源。webjars这个资源是通过pom引入的，

```xml
<dependency><!--Webjars省略版本号-->
            <groupId>org.webjars</groupId>
            <artifactId>webjars-locator-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>jquery</artifactId>
            <version>3.3.1</version>
        </dependency>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>bootstrap</artifactId>
            <version>4.1.3</version>
        </dependency>
```

th:each可以用来遍历后端传过来的list集合。在集合的迭代过程还可以获取状态变量，只需在变量后面指定状态变量名即可，状态变量可用于获取集合的下标/序号、总数、是否为单数/偶数行、是否为第一个/最后一个。

th:inline是内联表达式， 可以是文本内联，比如：

```html
<p th:inline="text">Hello, [[${session.user.name}]]!</p>
```

[[…]]之间的表达式在Thymeleaf被认为是内联表达式，输出的时候会被替换成真正的值。如果不使用th:inline="text",则会被当做字符串显示。

也可以是上面示例那样的脚本内联，比如：

```html
<script th:inline="javascript">
    //通过内联表达式获取用户信息
    var userList = [[${userList}]];
    console.log(userList)
</script>
```

这里的脚本很简单，就是获取用户列表打印出来。


## 测试
 
启动springboot，浏览器输入

```bash
http://localhost:9091/
```

会看到所有用户的列表信息。 

输入，
```bash
http://localhost:9091/user/1
```

查看某个用户的信息。


参考:

https://www.jianshu.com/p/908b48b10702