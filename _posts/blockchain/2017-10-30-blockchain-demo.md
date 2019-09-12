---
layout: post
title: 一个简易的区块链demo
category: blockchain
tags: [blockchain,比特币,go,demo]
---


## 别人写的python版本


python版本源码地址:

https://github.com/dvf/blockchain#installation

-----------

## 环境准备

我使用的是ubuntu 16.04，其它linux版本也可以。

需要安装python3.6+(步骤省略)


安装pipenv
```
$ pip install pipenv 
```

创建虚拟执行环境(类似docker一样)


```
root@pony-virtual-machine:~# pipenv --python=python3.6
Creating a virtualenv for this project…
Using /usr/bin/python3.6 to create virtualenv…
⠋Running virtualenv with interpreter /usr/bin/python3.6
Using base prefix '/usr'
New python executable in /root/.local/share/virtualenvs/root-BuDEOXnJ/bin/python3.6
Also creating executable in /root/.local/share/virtualenvs/root-BuDEOXnJ/bin/python
Installing setuptools, pip, wheel...done.

Virtualenv location: /root/.local/share/virtualenvs/root-BuDEOXnJ
Creating a Pipfile for this project…


```

随便找一个目录下载源码

```
git clone https://github.com/dvf/blockchain.git
```




切换到源码目录

```
root@pony-virtual-machine:/usr/local# pipenv install
Pipfile.lock not found, creating…
Locking [dev-packages] dependencies…
Locking [packages] dependencies…
Updated Pipfile.lock (711973)!
Installing dependencies from Pipfile.lock (711973)…
```

## 源码分析

源码不分析了，太过简单，自己看吧。不过能看懂源码的前提是你对区块链(尤其是比特币上的区块链)的一些机制原来有一定的了解，比如工作量证明，共识机制等。

源码中有一处错误我已经在github提交了issue反馈。

访问测试
因为条件有限，我在一台主机里启动三个不同端口的服务，模拟三个网络中的节点。分别是:

A节点
``` bash
root@pony-virtual-machine:/home/pony/python/projects/blockchain# pipenv run python blockchain.py
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)

B节点
``` bash
root@pony-virtual-machine:/home/pony/python/projects/blockchain# pipenv run python blockchain.py -p 5001
 * Running on http://0.0.0.0:5001/ (Press CTRL+C to quit)
```

C节点

``` bash
root@pony-virtual-machine:/home/pony/python/projects/blockchain# pipenv run python blockchain.py --port 5002
 * Running on http://0.0.0.0:5002/ (Press CTRL+C to quit)
```

这里要注意下，因为三个节点并没有实现分布式协议，所以节点之前不会同步数据，所以下面的测试其实还是不完善

测试的流程如下:

首先我要把三个节点都注册到测试区块链的网络里，我通过A节点的接口把B节点和C节点加入进来。

```
http://localhost:5000/nodes/register

json data:

{
    "nodes": [
        "http://127.0.0.1:5001",
        "http://127.0.0.1:5002"
    ]
}

```
响应

```
{
  "message": "New nodes have been added",
  "total_nodes": [
    "127.0.0.1:5001",
    "127.0.0.1:5002"
  ]
}
```


在A节点上新增两笔交易

```
http://localhost:5000/transactions/new

json data:
{
    "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
    "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
    "amount": 102
}
```
响应
```
{
  "message": "Transaction will be added to Block 2"
}
```

```
http://localhost:5000/transactions/new

json data:
{
    "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
    "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
    "amount": 103
}
```
响应
```
{
  "message": "Transaction will be added to Block 2"
}
```


两笔交易会存在节点A的内存中(current_transactions变量)。

下面在节点A上执行“挖矿”的动作，

```
http://localhost:5000/mine

响应
{
  "index": 2,
  "message": "New Block Forged",
  "previous_hash": "1",
  "proof": 35293,
  "transactions": [
    {
      "amount": 102,
      "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
      "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
    },
    {
      "amount": 103,
      "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
      "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
    },
    {
      "amount": 1,
      "recipient": "e5fad4f985494c52ae71c31a0d958fde",
      "sender": "0"
    }
  ]
}
```

可以看到挖矿产生的新区块已经包含了我们刚才添加的两笔交易。另外还有一笔金额是1的交易，这个是用来奖励矿工的。

挖矿的结果除了把新的交易加入一个新的区块，还在区块上产生一个工作量证明的值(block的proof字段)。

这个时候新的区块就加入到了链上，我们可以获取整条链看看:

```
http://localhost:5000/chain


响应
{
  "chain": [
    {
      "index": 1,
      "previous_hash": "1",
      "proof": 100,
      "timestamp": 1508836211.1095436,
      "transactions": []
    },
    {
      "index": 2,
      "previous_hash": "1",
      "proof": 35293,
      "timestamp": 1508836541.9152732,
      "transactions": [
        {
          "amount": 102,
          "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
          "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
        },
        {
          "amount": 103,
          "recipient": "1Ez69SnzzmePmZX3WpEzMKTrcBF2gpNQ55",
          "sender": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
        },
        {
          "amount": 1,
          "recipient": "e5fad4f985494c52ae71c31a0d958fde",
          "sender": "0"
        }
      ]
    }
  ],
  "length": 2
}
```

## 用docker来搭建环境

上面我们用的是pipenv来作为执行环境，关于什么是pipenv，可以看下这篇文章:

怎么使用pipenv管理你的python项目

总之就是了类似docker一样的虚拟环境，但是是针对python的.

用docker的好处有很多，首先一个就是不依赖任何语言。其次docker运行多个实例可以使用同一个端口，更接近真实的场景。

切换到项目目录下执行

```
docker build -t blockchain .
```

命令会根据目录下的dockerfile文件创建一个docker容器, 执行成功后，用docker images可以查到该容器。
```
blockchain                     latest              aa4dbddbd6e0        3 minutes ago       143MB
```
启动容器, 我们这里还是启动三个节点。

```
docker run --rm -p 81:5000 blockchain
docker run --rm -p 82:5000 blockchain
docker run --rm -p 83:5000 blockchain
```


测试的http地址和上面一样，只不过端口地址改成了81, 82, 83

写一个go版本
我仿照这个思路写了一个go语言实现的版本。

地址:

https://github.com/pony-maggie/blockchain

  