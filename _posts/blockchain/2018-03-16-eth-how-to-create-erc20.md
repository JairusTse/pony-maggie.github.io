---
layout: post
title: 教你如何一步步创建ERC20代币
category: blockchain
tags: [blockchain,以太坊,DAPP,ERC20,代币]
---

看这篇文章需要对以太坊，智能合约，代币等概念有基本的了解。

## 什么是ERC20

可以把ERC20简单理解成以太坊上的一个代币协议，所有基于以太坊开发的代币合约都遵守这个协议。遵守这些协议的代币我们可以认为是标准化的代币，而标准化带来的好处是兼容性好。这些标准化的代币可以被各种以太坊钱包支持，用于不同的平台和项目。说白了，你要是想在以太坊上发行代币融资，必须要遵守ERC20标准。


ERC20的标准接口是这样的:

```
  contract ERC20 {
      function name() constant returns (string name)
      function symbol() constant returns (string symbol)
      function decimals() constant returns (uint8 decimals)
      function totalSupply() constant returns (uint totalSupply);
      function balanceOf(address _owner) constant returns (uint balance);
      function transfer(address _to, uint _value) returns (bool success);
      function transferFrom(address _from, address _to, uint _value) returns (bool success);
      function approve(address _spender, uint _value) returns (bool success);
      function allowance(address _owner, address _spender) constant returns (uint remaining);
      event Transfer(address indexed _from, address indexed _to, uint _value);
      event Approval(address indexed _owner, address indexed _spender, uint _value);
    }
```

**name**

返回ERC20代币的名字，例如"My test token"。

**symbol**

返回代币的简称，例如：MTT，这个也是我们一般在代币交易所看到的名字。

**decimals**

返回token使用的小数点后几位。比如如果设置为3，就是支持0.001表示。

**totalSupply**

返回token的总供应量

**balanceOf**

返回某个地址(账户)的账户余额


**transfer**

从代币合约的调用者地址上转移_value的数量token到的地址_to，并且必须触发Transfer事件。 


**transferFrom**

从地址_from发送数量为_value的token到地址_to,必须触发Transfer事件。

transferFrom方法用于允许合同代理某人转移token。条件是from账户必须经过了approve。这个后面会举例说明。

**approve**

允许_spender多次取回您的帐户，最高达_value金额。 如果再次调用此函数，它将以_value覆盖当前的余量。


**allowance**

返回_spender仍然被允许从_owner提取的金额。


后面三个方法不好理解，这里还需要补充说明一下，

approve是授权第三方（比如某个服务合约）从发送者账户转移代币，然后通过 transferFrom() 函数来执行具体的转移操作。

账户A有1000个ETH，想允许B账户随意调用他的100个ETH，过程如下：

1. A账户按照以下形式调用approve函数approve(B,100)

2. B账户想用这100个ETH中的10个ETH给C账户，调用transferFrom(A, C, 10)

3. 调用allowance(A, B)可以查看B账户还能够调用A账户多少个token

另外，我推荐这篇文章，对这部分概念讲解的比较清楚。

https://mp.weixin.qq.com/s/foM1QWvsqGTdHxHTmjczsw



后面两个是事件，事件是为了获取日志方便提供的。前者是在代币被转移时触发，后者是在调用approve方法时触发。




## 基于ERC20编写的一个代币合约

```
pragma solidity ^0.4.16;
contract Token{
    uint256 public totalSupply;

    function balanceOf(address _owner) public constant returns (uint256 balance);
    function transfer(address _to, uint256 _value) public returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) public returns   
    (bool success);
    
    function approve(address _spender, uint256 _value) public returns (bool success);
    
    function allowance(address _owner, address _spender) public constant returns 
    (uint256 remaining);

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 
    _value);
}

contract TokenDemo is Token {
    
    string public name;                   //名称，例如"My test token"
    uint8 public decimals;               //返回token使用的小数点后几位。比如如果设置为3，就是支持0.001表示.
    string public symbol;               //token简称,like MTT
    
    function TokenDemo(uint256 _initialAmount, string _tokenName, uint8 _decimalUnits, string _tokenSymbol) public {
        totalSupply = _initialAmount * 10 ** uint256(_decimalUnits);         // 设置初始总量
        balances[msg.sender] = totalSupply; // 初始token数量给予消息发送者，因为是构造函数，所以这里也是合约的创建者
        
        name = _tokenName;                   
        decimals = _decimalUnits;          
        symbol = _tokenSymbol;
    }
    
    function transfer(address _to, uint256 _value) public returns (bool success) {
        //默认totalSupply 不会超过最大值 (2^256 - 1).
        //如果随着时间的推移将会有新的token生成，则可以用下面这句避免溢出的异常
        require(balances[msg.sender] >= _value && balances[_to] + _value > balances[_to]);
        require(_to != 0x0);
        balances[msg.sender] -= _value;//从消息发送者账户中减去token数量_value
        balances[_to] += _value;//往接收账户增加token数量_value
        Transfer(msg.sender, _to, _value);//触发转币交易事件
        return true;
    }


    function transferFrom(address _from, address _to, uint256 _value) public returns 
    (bool success) {
        require(balances[_from] >= _value && allowed[_from][msg.sender] >= _value);
        balances[_to] += _value;//接收账户增加token数量_value
        balances[_from] -= _value; //支出账户_from减去token数量_value
        allowed[_from][msg.sender] -= _value;//消息发送者可以从账户_from中转出的数量减少_value
        Transfer(_from, _to, _value);//触发转币交易事件
        return true;
    }
    function balanceOf(address _owner) public constant returns (uint256 balance) {
        return balances[_owner];
    }


    function approve(address _spender, uint256 _value) public returns (bool success)   
    { 
        allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
        return true;
    }

    function allowance(address _owner, address _spender) public constant returns (uint256 remaining) {
        return allowed[_owner][_spender];//允许_spender从_owner中转出的token数
    }
    mapping (address => uint256) balances;
    mapping (address => mapping (address => uint256)) allowed;
}
```


代码不必过多的解释，注释都写得很清楚了。

**这里可能有人会有疑问，name，totalSupply这些按照标准不应该都是方法吗，怎么这里定义的是属性变量？ 这是因为solidity会自动给public变量生成同名的getter接口。**





## 部署测试

我会提供两个环境的部署测试流程，都是亲测过的，大家可以根据自己的喜好选择。我个人平时用得比较多的是后者。




### Remix+MetaMask环境部署测试

这部分要求你的浏览器已经安装了MetaMask插件，至于什么是MetaMask以及如何安装和使用请自行搜索查询。MetaMask我们用的是测试环境的网络，在测试网络中可以申请一些以太币进行测试。

我们把代码复制到remix编译，没问题的话如下图所示点击create创建合约，参数可以按照下图的方式设置。注意环境选择injected web3，这样会打开浏览器插件MetaMask进行测试部署。


![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-92922f82f26305a5?imageMogr2/auto-orient/strip%7CimageView2/2/w/382)

点击create后会弹出合约确认界面，直接点击submit，等待合约确认。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-6ed82677f7c9d12f?imageMogr2/auto-orient/strip%7CimageView2/2/w/344)

我们可以在MetaMask里点击该笔合约提交的明细，就会跳转到以太坊的浏览器中，可以在这里看到合约的各种信息：

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-2cc1ee36784c2180?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

如上图所示，1表示该笔交易(合约也是一种交易)的hash值，2是当前合约所处的区块位置(当然是测试环境)和已经被确认的区块链数量，3是合约的创建地址，4是合约本省所在的地址。

3和4的概念容易混淆，注意理解。


进入MetaMask的token界面中，点击add token，然后我们把合约的地址复制到过去提交就可以看到我们的代币了。还可以点击代币的图标打开浏览器查看代币的详细信息。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-ca1b5e9a1e0eecca?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


到这里你已经完成了代币的开发部署。接下来我们还要看看如何进行代币的转账，这个也是代币比较常用的操作。转账我们需要结合以太坊钱包MyEtherWallet，这是个以太坊的网页版轻量级钱包，利用它可以很方便的对我们的以太币和其它代币进行管理。


转账前我们首先要把代币加入到钱包中，

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-e9d27c677069c71e?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-0b480ec3f07f374f?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



注意在上图中，我们选择的环境同样是测试环境并且和MetaMask中的环境一致。点击add custome token，输入代币地址等信息就可以看到代币了，然后进行转账操作。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-1fd7bea7527ad369?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
我们随便转入一个地址，转账完成后，发现代币余额确实减少了。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-8f636248102d7042?imageMogr2/auto-orient/strip%7CimageView2/2/w/345)

### 以太坊钱包mist+geth私有环境部署测试

我个人开发用这个环境比较多，不过这个环境安装起来比较麻烦，具体流程可以看下我以前的文章。


打开mist钱包，进入合约界面，然后点击deploy new contact，然后把代码复制进去编译。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-2fa375cf3a12733b?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

然后点击deploy

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-46c1c97cff17e6db?imageMogr2/auto-orient/strip%7CimageView2/2/w/580)

输入账户密码开始部署。

随着挖矿的进行，合约就被部署到我的geth私有环境中了，


![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-0b66c80e73eaa67b?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


回到钱包的合约界面已经可以看到合约了，
![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-0b66c80e73eaa67b?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


点击transfer ether&tokens，进入转账界面，进行转账。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-4091e8f3aa85f028?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-830b24cea44913bc?imageMogr2/auto-orient/strip%7CimageView2/2/w/578)

成功后可以看到余额已经减少，并且转入账户的余额增加。

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-aeca3d609583f5e5?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![这里写图片描述](https://upload-images.jianshu.io/upload_images/4210052-92907a470057b256?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



**参考**
* https://mp.weixin.qq.com/s/foM1QWvsqGTdHxHTmjczsw
* http://blog.csdn.net/a394268045/article/details/79616844
  