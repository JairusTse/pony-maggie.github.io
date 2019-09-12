---
layout: post
title: 谈谈自己对比特币脚本的理解
category: blockchain
tags: [blockchain,比特币,脚本]
---

# 谈谈自己对比特币脚本的理解


## 锁定脚本和解锁脚本

比特币脚本存在的意义是让每笔交易合法化，这个合法化不是人工审核而是有脚本自动执行校验的。

脚本分为锁定脚本和解锁脚本。锁定脚本和UTXO是对应的，一个UTXO中包含一个锁定脚本。

当这个UTXO要被使用时,比如alice转账给bob需要引用这个UTXO,这就产生了一笔交易。这笔交易只有被验证了才可能在比特币的网络中传播(传播后就可以被矿工加入区块链，这部分就不详述了。)

验证的时候需要的就是解锁脚本。

从上述流程可以看出: 锁定脚本和UTXO 关联，而解锁脚本和某笔交易关联。


锁定脚本被叫做scriptPubKey, 解锁脚本被叫做ScriptSig。

脚本究竟是什么形式存在的呢？其实就是一堆命令加参数。


![这里写图片描述](http://img.blog.csdn.net/20170625122248433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图中的dup, hash160等是命令，sig, pubk是参数。




## 脚本语言执行原理

脚本的执行流程基于堆栈模型。这个非常类似于我大学时期数据结构课程中讲到的表达式求值的实现逻辑。


输入的形式：表达式，例如2*(3+4)

输出的形式：运算结果

其中2,3,4相当于脚本中的参数，而*,+号是脚本的命令。

实现逻辑就是基于栈结构，初始都入栈，然后出栈判断如何操作。

比特币脚本也是类似的实现逻辑，而且它更加简单(没有优先级判断那些)。


![这里写图片描述](http://img.blog.csdn.net/20170625122301849?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图是一个很简单的脚本，就是判断2 add 3是否equal 5。下面的篇幅会详述一个实际的比特币交易脚本的执行过程。


## 数字签名和验签

比特币脚本的验证机制会用到数字签名的概念。这一部分知识是独立的一块，它基于非对称密钥算法，并不是比特币脚本特有的。如果要详细解释这一块也是很大篇幅，这里不再赘述，不懂的请自行查阅。


## 比特币地址如何生成

继续往下看之前，你需要了解公钥和比特币地址之间的关系:

以公钥 K 为输入，计算其 SHA256 哈希值，并以此结果计算 RIPEMD160 哈
希值，得到一个长度为 160 比特（20 字节）的数字：

```
A = RIPEMD160(SHA256(K))
```

公式中，K 是公钥，A 是生成的比特币地址。


## 比特币交易示例

假设alice要向bob支付0.015比特币, alice会用到一个UTXO(假设是单输入，单输出)，这个UTXO带有一个锁定脚本，为交易设置“障碍”。

锁定脚本如下:
```

OP_DUP OP_HASH160 be10f0a78f5ac63e8746f7f2e62a5663eed05788 OP_EQUALVERIFY OP_CHECKSIG

```

* OP_DUP:复制栈顶数据，然后该数据放置栈顶
* OP_HASH160:对栈顶数据执行ripemd160(sha256(data)) (这其实是两次摘要计算，不详述)
* be10f0a...:bob的比特币地址
* OP_EQUALVERIFY:对比栈顶的两个数据，如果相等都被移除
* OP_CHECKSIG:验证签名


bob如果要接收这笔比特币(另一种说法是bob可以引用该笔输出)，就要给出一个**解锁脚本**,然后解锁脚本和锁定脚本组合后执行的结果为真才能确认交易有效。

解锁脚本如下:

```
3046022100ba1427639c9f67f2ca1088d0140318a98cb1e84f604dc90ae00ed7a5f9c61cab02210094233d018f2f014a5864c9e0795f13735780cafd51b950f503534a6af246aca301
03a63ab88e75116b313c6de384496328df2656156b8ac48c75505cd20a4890f5ab

```
看起来是一堆数字，其实『签名』和『公钥』（sig & pubkey）的组合。签名是bob的私钥对该笔交易的信息加密的结果，公钥就是指的bob的公钥。

**由于私钥只有bob才知道，所以也只有他才能拿出正确的签名。**

下面是脚本执行的过程:


![这里写图片描述](http://img.blog.csdn.net/20170625122316526?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


![这里写图片描述](http://img.blog.csdn.net/20170625122327534?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9ueV9tYWdnaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


简单的几十个字节的脚本，就完成了交易的验证确保该笔转账的合法性。


## 比特币脚本变种

上文讲述的示例都是基于比特币最基本的P2PKH交易类型。现在比特币核心已经升级了很多版本，脚本的验证机制发生了不小的变化。比如说现在用的比较广泛的多重签名脚本。这些变种脚本虽然越来越复杂，但是基本思想都是基于上述原理。



**参考**

[1] Andreas M. Antonopoulos <<精通比特币>>
[2] http://www.8btc.com/understand-bitcoin-script