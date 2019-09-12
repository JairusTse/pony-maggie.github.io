---
layout: post
title:  基于maven的版本管理
category: arch
tags: [架构,微服务,java,maven]
---

* content
{:toc}

以前一个人开发基于maven的项目，都是简单粗暴的方式，哪管什么版本管理，需要什么在POM引入就可以了。后来管理技术团队才体会到maven的版本管理是如此强大，简直是团队协作开发利器。

这篇文章就是自己的一些经验之谈。 

## maven私有库

公司内部搭建自己的私有仓库是所有版本管理的基础，没有这个一切都免谈。

没有Nexus私服，我们所需的所有构件都需要通过maven的中央仓库或者第三方的Maven仓库下载到本地，这会带来很多问题：

* 可能因为网络问题无法下载(比如内网环境开发)
* 团队中的所有人都重复下载造成浪费
* 团队中的不同的人开发的模块不方便共享给其他成员


目前主流的方案都是基于NEXUS搭建的。架构如下：

![image](http://www.machengyu.net/assets/images/arch/maven1.jpg)

我不打算写安装过程了，网上比较多大家可以自行查找。

安装完成之后，界面是这样的（不同的版本界面可能会有差异）：

![image](http://www.machengyu.net/assets/images/arch/maven2.jpg)

* maven-central：这里存放从中央仓库或者第三方的下载的jar
* maven-releases：团队内部发布版本的jar
* maven-snapshots：团队内部开发版本的jar(后面还会细讲)

* maven-public：把上面三个仓库组合在一起对外提供服务(主要是在settings.xml中使用)


## maven snapshot

有了私钥库并不是万事大吉了。试想下这种场景，甲乙两个人各自开发一个模块，甲的模块需要依赖乙的模块，乙开发完一个版本后(比如1.1.1)就mvn:deploy到私有库，甲直接在自己的POM中引用即可。

``` xml
<dependency>
	<groupId>com.yi.develop</groupId>
	<artifactId>develop_by_yi</artifactId>
	<version>1.1.1</version>
</dependency>
```

过一段时间，乙发现代码中有个bug，于是赶紧修复然后把版本变为1.1.2发布到私有库上了。问题就来了，首先乙如果忘记告诉甲这个事情，甲就会一直引用一个低版本的乙模块，这可能会对整个应用带来致命的问题。其次即使乙及时告诉了甲这个消息，甲也要修改自己的pom文件进行更新。后面乙不断的更新版本，甲就要不断的更新POM文件。很麻烦吧？！

maven早就帮你想好了解决方案了，它引入了快照版本的概念。乙把自己开发的模块版本命名为1.1-snapshot，甲这样引用到自己的工程中：

``` xml
<dependency>
	<groupId>com.yi.develop</groupId>
	<artifactId>develop_by_yi</artifactId>
	<version>1.1-snapshot</version>
</dependency>
```

乙每次更新提交到私有库后，都会将SNAPSHOT改成一个当前时间的时间戳命名的jar，Maven在处理SNAPSHOT依赖时，会根据时间戳下载最新的jar。默认情况下，快照的版本会每天自动更新一次。如果需要实时引用到最新的依赖包，可以使用-U参数强制更新。

``` bash
mvn clean install -U 
```


## maven 版本编号规则

Maven版本号采用的是通用的三级规则：

[主版本号].[副版本号].[修复版本号]

* 主版本号一般来说代表了项目的重大的架构变更
* 副版本号一般指增加或者减少功能
* 修复版本号一般代表bug fix









