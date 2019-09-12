---
layout: post
title: 比特币客户端Electrum使用介绍
category: blockchain
tags: [blockchain,比特币,eletrum]
---

## 简介

比特币的客户端很多，为什么选择Electrum。

首先Electrum真的很轻量，安装马上可以用，不用下载几百G的区块链账本。我之前安装bitcoin核心客户端，这是个完整节点。下载账本都要好多天。后来果断弃用了。

其次，Electrum钱包每次交易后使用新的地址，使得窥探你的余额和支付历史变得困难，安全性不错。

轻量化的概念是什么，请看下图:


![这里写图片描述](http://img.blog.csdn.net/20170715150657156?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


一个全节点的客户端需要具备该图的四个功能。而像Electrum这样的轻量级客户端，只要**钱包**和**网路路由节点**即可。



客户端下载地址:

https://electrum.org/#download


## 安装流程

比较简单，不详细描述。安装过程中会让设置一个叫 **安全种子** 的东西，是一串英文字符串。这个下个章节会讲到。

还要就是需要设置一个密码，这个是平时登录，导出等操作需要的验证密码。

## 收取比特币

请参考我的另一篇文章

[获取自己的第一枚比特币](http://blog.csdn.net/pony_maggie/article/details/73776370)


## 种子与助记码词汇

比如下面这一串词汇:

```
army van defense carry jealous true garbage claim echo media make crunch
```




这种单词的序列在比特币钱包中被设计足以重新创建种子，并且从种子那里重新创造钱包以及所有私钥。

在首次创建钱包时，带有助记码的，运行确定性钱包的钱包的应用程序将会向使
用者展示一个 12 至 24 个词的顺序。单词的顺序就是钱包的备份。

它也可以被用来恢复以及重新创造应用程序相同或者兼容的钱包的钥匙。你会发现，这种看起来更有意义的单次更加容易记忆和抄写。便于比特币钱包的备份和恢复。


助记码被定义在比特币的改进建议[bip39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)中。


下面是我用助记码恢复钱包的示例流程，找一台其它电脑，准备把我本机的钱包转移到这台电脑上。下载安装包，然后安装图示的流程即可恢复。


![这里写图片描述](http://img.blog.csdn.net/20170715150740820?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



![这里写图片描述](http://img.blog.csdn.net/20170715150751573?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


![这里写图片描述](http://img.blog.csdn.net/20170715150803865?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



![这里写图片描述](http://img.blog.csdn.net/20170715150815262?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)




## 命令行

Electrum支持python风格的命令行操作，其实这个我也很少用，因为大部分功能工具栏上都可以操作。一共支持这么多命令:

```
>> help()
[
    "addrequest", 
    "broadcast", 
    "check_seed", 
    "clearrequests", 
    "commands", 
    "create", 
    "createmultisig", 
    "decrypt", 
    "deserialize", 
    "dumpprivkeys", 
    "encrypt", 
    "freeze", 
    "getaddressbalance", 
    "getaddresshistory", 
    "getaddressunspent", 
    "getalias", 
    "getbalance", 
    "getconfig", 
    "getmasterprivate", 
    "getmerkle", 
    "getmpk", 
    "getprivatekeys", 
    "getproof", 
    "getpubkeys", 
    "getrequest", 
    "getseed", 
    "getservers", 
    "gettransaction", 
    "getunusedaddress", 
    "getutxoaddress", 
    "help", 
    "history", 
    "importprivkey", 
    "is_synchronized", 
    "ismine", 
    "listaddresses", 
    "listcontacts", 
    "listrequests", 
    "listunspent", 
    "make_seed", 
    "notify", 
    "password", 
    "payto", 
    "paytomany", 
    "restore", 
    "rmrequest", 
    "searchcontacts", 
    "serialize", 
    "setconfig", 
    "setlabel", 
    "signmessage", 
    "signrequest", 
    "signtransaction", 
    "sweep", 
    "unfreeze", 
    "validateaddress", 
    "verifymessage", 
    "version"
]
```

从名字基本可以猜到每个命令的意思，比如listaddresses可以列出该钱包的所有收款地址。

```
>> listaddresses()
[
    "13kBNVybeErYra1hmXQGhrJswgD1thEsQF", 
    "1A5aL83bJ2bSFF8fVnxfmiDxeU8K9raiZQ", 
    "13SighQBMHwwqkn3LCkc7jymvL5BKZ3jRq", 
    "16RrZuD2h7rdVzANg4PvjgkdNXXa9qDZ3b", 
    "1LUaCgb7NdSv5ZqBbWdoYfW5Zzb3MrrkJq", 
    "1FevUo7VTqxUmRQkPMRESBa5TzCDyXgit3", 
    "12ihinYSHr9Y5WZpP1UB5eH5xsYuCcNZX3", 
    ...
]
```

## 选择在线钱包

Electrum属于你电脑本地钱包，当然我们也可以选择一些知名度高的在线钱包。比如BlockChain.Info就是这种。

那么假设我需要把本地钱包导入到在线钱包，该如何操作呢？


请参考下面的连接


[如何将比特币从比特币核心钱包(Bitcoin Core)导入到blockchain的在线钱包](http://www.togof.com/qiushi/2013.htm)


## 冷钱包

Electrum还有一个比较厉害的功能时支持冷钱包。什么是冷钱包呢？

首先我们说冷钱包的目的是为了安全。

原理是这样的，首先你有两台电脑，一台永远不联网(找一台便宜的旧电脑吧)，一台会联网。
两台电脑上都安装Electrum。联网的那台是没有私钥的，只有公钥信息。联网的那台每次创建的交易，要拿到离线的电脑上签名，然后再把交易拷贝到联网电脑上广播到比特币网络中。

看起来操作比较麻烦，确实是这样的。但是这就是安全的代价。如果你的比特币比较多，我建议你可以这样弄，否则就算了。


冷钱包的具体流程参考:

http://www.8btc.com/cold-wallet



**参考**

[1] <<精通比特币>>
