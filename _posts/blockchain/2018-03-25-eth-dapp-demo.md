---
layout: post
title: 以太坊DApp开发的入门示例
category: blockchain
tags: [blockchain,以太坊,DAPP,demo]
---

## 环境准备

ubuntu 16.04， 64位

还需要安装以太坊相关的环境:
* nodejs
* truffle
* solidity
* testrpc

可以参考我之前的一篇文章:

http://blog.csdn.net/pony_maggie/article/details/79531534

另外，本篇还会用到webpack，安装教程网上也有很多。这部分如果不熟悉的话请自行查阅学习下。需要注意的是本篇我用的webpack版本是3.x，本文写作时webpack4.x已经发布。4.x改动还是比较大，建议大家使用3.x的版本运行本文中的代码示例。




## 编写智能合约

首先在用户目录下新建conference目录，进入目录执行truffle init，该命令会建立如下的子目录和文件:

* contracts/:  智能合约存放的目录，默认情况下已经帮你创建 Migrations.sol合约。
* migrations/: 存放部署脚本
* test/: 存放测试脚本
* truffle.js: truffle的配置文件


修改truffle.js文件，改成如下:

```
module.exports = {
  networks: {
        development: {
            host: "localhost",
            port: 8545,
            network_id: "*" // 匹配任何network id
         }
    }
};
```

这里是设置我们稍后要部署智能合约的位置, 否则会报网络错误。

开启一个终端，输入testrpc运行测试节点。testrpc是一个完整的在内存中的区块链测试环境，启动 testrpc 经后，会默认创建10个帐号，Available Accounts是帐号列表，Private Keys是相对应的帐号密钥。


进入contracts目录，这里是存放合约代码的地方。我们可以使用sublime等工具编写测试合约代码。我这里只贴出部分代码，文章最后会给出完整源码的地址。

```
pragma solidity ^0.4.19;

contract Conference {  // can be killed, so the owner gets sent the money in the end

	address public organizer;
	mapping (address => uint) public registrantsPaid;
	uint public numRegistrants;
	uint public quota;

	event Deposit(address _from, uint _amount); // so you can log the event
	event Refund(address _to, uint _amount); // so you can log the event

	function Conference() {
		organizer = msg.sender;		
		quota = 100;
		numRegistrants = 0;
	}

...

```

合约内容很简单，是一个针对会议的智能合约，通过它参会者可以买票，组织者可以设置参会人数上限，以及退款策略。


## 编译部署智能合约

修改migrations下的1_initial_migration.js文件，改成如下：

```
//var Migrations = artifacts.require("./Migrations.sol");
var Conference = artifacts.require("./Conference.sol");

module.exports = function(deployer) {
  //deployer.deploy(Migrations);
  deployer.deploy(Conference);
};

```

编译，

```
$ sudo truffle compile --compile-all

```

注意看下有无报错。

Truffle仅默认编译自上次编译后被修改过的文件，来减少不必要的编译。如果你想编译全部文件，可以使用--compile-all选项。

然后会多出一个build目录，**该目录下的文件都不要做任何的修改。**


部署，

```
$ sudo truffle migrate --reset
```

>这个命令会执行所有migrations目录下的js文件。如果之前执行过truffle migrate命令，再次执行，只会部署新的js文件，如果没有新的js文件，不会起任何作用。如果使用--reset参数，则会重新的执行所有脚本的部署。

测试下，在test目录新增一个conference.js测试文件，

```
var Conference = artifacts.require("./Conference.sol");

contract('Conference', function(accounts) {
  console.log("start testing");
	//console.log(accounts);
	var owner_account = accounts[0];
  var sender_account = accounts[1];


  it("Initial conference settings should match", function(done) {
  	
  	Conference.new({from: owner_account}).then(
  		function(conference) {
  			conference.quota.call().then(
  				function(quota) { 
  					assert.equal(quota, 100, "Quota doesn't match!"); 
  			}).then(
  				function() { 
  					return conference.numRegistrants.call(); 
  			}).then(
  				function(num) { 
  					assert.equal(num, 0, "Registrants doesn't match!");
  					return conference.organizer.call();
  			}).then(
  				function(organizer) { 
  					assert.equal(organizer, owner_account, "Owner doesn't match!");
  					done();
  			}).catch(done);
  	}).catch(done);
  });
  
  ...
  
```

这里只贴出部分代码，四个测试case，运行truffle test查看测试结果。
```
$ truffle test
Using network 'development'.

start testing

  Contract: Conference
    ✓ Initial conference settings should match (191ms)
    ✓ Should update quota (174ms)
    ✓ Should let you buy a ticket (717ms)
    ✓ Should issue a refund by owner only (714ms)


  4 passing (2s)

```


------------------------

## 编写web应用


在conference目录下执行npm init，然后一路回车，会生成一个名为package.json的文件，编辑这个文件，在scripts部分增加两个命令，最终如下:

```
{
  "name": "conference",
  "version": "1.0.0",
  "description": "",
  "main": "truffle-config.js",
  "directories": {
    "test": "test"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "webpack",
    "server": "webpack-dev-server --open"
  },
  "author": "",
  "license": "ISC"
}

```

package.json文件定义了这个项目所需要的各种模块，以及项目的配置信息（比如名称、版本、许可证等元数据）。npm 命令根据这个配置文件，自动下载所需的模块，也就是配置项目所需的运行和开发环境。


然后在conference目录下新建app目录，并创建index.html文件，如下:

```
<!DOCTYPE html>
<html>
<head>
  <title>Conference DApp2</title>
  <link href='https://fonts.loli.net/css?family=Open+Sans:400,700,300' rel='stylesheet' type='text/css'>
  <script type="text/javascript" src="http://code.jquery.com/jquery-1.9.1.min.js"></script>
  <script src="./app.js"></script>
</head>
<body>
  <h1>Conference DApp</h1>
  <div class="section">
    Contract deployed at: <div id="confAddress"></div>
  </div>
  <div class="section">
    Organizer: <input type="text" id="confOrganizer" />
  </div>
  <div class="section">
    Quota: <input type="text" id="confQuota" />
      <button id="changeQuota">Change</button>
      <span id="changeQuotaResult"></span>
  </div>
  <div class="section">
    Registrants: <span id="numRegistrants">0</span>
  </div>

  <hr/>

</body>
</html> 


```

然后在app目录下新建javascripts目录和styleheets目录，分别存放js脚本文件和css样式文件。真正和合约交互的就是脚本文件。

脚本文件名为app.js，部分代码如下:

```
import "../stylesheets/app.css";
import {  default as Web3 } from 'web3';
import {  default as contract } from 'truffle-contract';

import conference_artifacts from '../../build/contracts/Conference.json'

var accounts, sim;
var Conference = contract(conference_artifacts);



window.addEventListener('load', function() {
	//alert("aaaaa");
    // Checking if Web3 has been injected by the browser (Mist/MetaMask)
    if (typeof web3 !== 'undefined') {
        console.warn("Using web3 detected from external source. If you find that your accounts don't appear or you have 0 MetaCoin, ensure you've configured that source properly. If using MetaMask, see the following link. Feel free to delete this warning. :) http://truffleframework.com/tutorials/truffle-and-metamask")
        // Use Mist/MetaMask's provider
        window.web3 = new Web3(web3.currentProvider);
    } else {
        console.warn("No web3 detected. Falling back to http://localhost:8545. You should remove this fallback when you deploy live, as it's inherently insecure. Consider switching to Metamask for development. More info here: http://truffleframework.com/tutorials/truffle-and-metamask");
        // fallback - use your fallback strategy (local node / hosted node + in-dapp id mgmt / fail)
        window.web3 = new Web3(new Web3.providers.HttpProvider("http://localhost:8545"));
    }

    Conference.setProvider(web3.currentProvider);
    App.start();

    $("#changeQuota").click(function() {
        var newquota = $("#confQuota").val();
        App.changeQuota(newquota);
    });

    // Wire up the UI elements
});

...

```

这个代码我也不打算过多的解释，主要就是用JS加wweb3 API调用合约的函数而已。


到这里为止，web部分基本已经准备好了，我们只需要用webpack打包部署即可。webpack打包还需要一个配置文件，名为webpack.config.js，这个文件是告诉webpack打包的规则，涉及webpack的用法，这里不做过多的解释。

## 打包部署web应用

打包部署需要安装webpack和相关的组件，安装的方式有全局安装和局部安装两种。所谓的局部安装，是指组件都是安装在项目的目录下(conference/node_modules)。我这里采用的就是局部安装。根据我们项目的实际情况，需要安装以下组件，



```
npm install --save-dev webpack@3.0.0
npm install babel-loader --save-dev
npm install babel-core --save-dev
npm install html-loader --save-dev
npm install --save-dev webpack-dev-server@2.11.0
npm install html-webpack-plugin --save-dev
npm install truffle-contract --save-dev

npm install --save-dev style-loader css-loader
```


环境装好，可以打包了。

```
$ sudo npm run start

> conference@1.0.0 start /home/pony/ethereum/conference
> webpack

Hash: ec8b764f75c05b477d9d
Version: webpack 3.0.0
Time: 2686ms
       Asset       Size  Chunks                    Chunk Names
   bundle.js    3.36 MB       0  [emitted]  [big]  main
./index.html  740 bytes          [emitted]         
  [10] (webpack)/buildin/global.js 509 bytes {0} [built]
  [16] (webpack)/buildin/module.js 517 bytes {0} [built]
  [47] ./app/javascripts/app.js 3.85 kB {0} [built]
  [48] ./app/stylesheets/app.css 1.08 kB {0} [built]
  [49] ./node_modules/css-loader!./app/stylesheets/app.css 413 bytes {0} [built]
 [175] ./build/contracts/Conference.json 71.1 kB {0} [built]
    + 170 hidden modules
Child html-webpack-plugin for "index.html":
       [0] ./node_modules/html-webpack-plugin/lib/loader.js!./app/index.html 706 bytes {0} [built]

```

没报错的话，进入build目录可以看到bundle.js和index.html两个文件，这两个就是最终打包好的网页文件。

然后部署，

```
$ sudo npm run server

> conference@1.0.0 server /home/pony/ethereum/conference
> webpack-dev-server --open

Project is running at http://localhost:8080/
webpack output is served from /
Content not from webpack is served from ./build
404s will fallback to /index.html
Hash: ecae3662137376f80de0
Version: webpack 3.0.0

...

```
这样相当于运行了一个小型的nodejs服务器，我们可以在浏览器输入http://localhost:8080/看看效果:



![这里写图片描述](https://img-blog.csdn.net/20180325133203813?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvbnlfbWFnZ2ll/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


可以看到合约的发布地址和会议组织者地址(msg.sender)都已经成功的显示出来了，点击change按钮还可以改变quota的值。




--------------------

本文的代码我已经上传到github上。

https://github.com/pony-maggie/conference


**参考**

* https://ethfans.org/posts/101-noob-intro

  