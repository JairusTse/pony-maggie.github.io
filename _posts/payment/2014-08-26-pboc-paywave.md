---
layout: post
title: 韩国PAYWAVE认证之技术篇
category: payment
tags: [pboc]
---

之前关于paywave认证写过一篇<<韩国PAYWAVE认证之韩城攻略>>,是非技术类的文章，主要介绍了饮食，实验室附近住宿等情况。后来有读者留言希望能写一篇技术攻略，本人太懒，一直拖到现在才出来。


还是希望看这篇文章的人能有一定的EMV/PBOC的应用基础，因为我会讲到很多二者之间的差异性。因为之前在韩国是做的终端方面的认证，不涉及卡片，所以下面讲到的技术细节都是从终端的角度。

## 一 什么是paywave

免不了入俗套，还是先讲概念。Paywave是visa推出的一种通过非接通道进行快速支付的解决方案，用来规范智能卡和终端的交易行为，安全等方方面面。


paywave这是visa一家推的，并没有跟其它机构或公司合作，所以算是visa的亲生仔,自然在推广起来是不遗余力的。有人可能疑惑不是有emv（由包括visa在内的三家厂商共同的推的）了吗，为什么visa还要单独再弄个paywave呢？

 

原因是emv是针对标准借贷记的，本身并没有关于非接触快速支付的规范，所以这一部分其实是空缺的，自然这些大厂商都会抢这个蛋糕（mastercard还有个paypass）。

 

当然Paywave跟emv本身也不是独立无关的，它的应用测试也是要求非接EMV level1的证书，硬件电气协议的要求是一样的。Paywave在应用业务层的测试叫功能测试(functional testing )



最后还有一个所谓的交叉测试(cross testing), 这个测试大概是模拟一些真实使用场景的测试, 前面两个如果都过了，这个一般问题不大。

 

## 二 技术点

功能测试包括MSD和qVSDC，厂商可以选择是否两个同时支持或是只支持qVSDC（只支持MSD就没意思了吧）， 在我看来MSD慢慢会被摒弃掉。要了解qVSDC，支付应用部分只要看一本规范就行了， << Visa Contactless Payment Specification>>.



是的，只有一本规范。 不像emv规范有四本，PBOC就更不必说了，十几本(额滴神，居然还有人以能记住第几本讲什么内容而沾沾自喜)。另外，如果你仔细对比一下，会发现PBOC中的qPBOC规范似乎也”参考”了不少qVSDC的内容。

 

我准备分成几个类别来阐述技术点：

 

### 关于指令以及流程的精简

qVSDC所用的apdu指令是EMV的子集, 这个可以理解,毕竟它的步骤比较少。就连应用选择中的建立候选列表也只保留PPSE这一种方式，AID选择法不支持。对于卡片，像卡片行为分析，生成应用密文，生成签名等步骤都放在收到GPO指令后一起处理从而简化流程。

 

### 预处理

要讲预处理，得先讲一个概念，TTQ(终端交易属性，4字节)，它的编码如下:

 
```
Byte 1

bit 8: 1 = MSD supported

bit 7: RFU (0)

bit 6: 1 = qVSDC supported

bit 5: 1 = EMV contact chip supported

bit 4: 1 = Offline-only reader

bit 3: 1 = Online PIN supported

bit 2: 1 = Signature supported

bit 1: 1 = Offline Data Authentication (ODA) for Online Authorizations supported.

Note: Readers compliant to this specification set TTQ byte 1 bit 1 to 0b.

Byte 2

bit 8: 1 = Online cryptogram required

bit 7: 1 = CVM required

bit 6: 1 = (Contact Chip) Offline PIN supported

bits 5-1: RFU (00000)

Byte 3

bit 8: 1 = Issuer Update Processing supported

bit 7: 1 = Mobile functionality supported (Consumer Device CVM)

bits 6-1: RFU (000000)

Byte 4

RFU ('00')
```
 

大部分比特位都是指示终端的能力，是配置好的静态数据，只有byte2的bit8和bit7是可变的, 预处理阶段终端可以获取到交易金额，然后就执行一系列的检查, 主要有以下几种:

 

**单位货币状态检查**

如果交易金额是1个单位货币(比如人民币就是1元)，就要把byte2的bit8置为1。

 

**零金额检查**

这种情况有点复杂，联机终端和仅脱机终端的处理不一样，比如有联机能力的终端，要支持两个配置，配置1的情况下，零金额要把byte2的bit8置为1。配置2的情况，要置位应用不被允许标志，在后面的流程中会用来判断交易是否继续进行。

 

**非接交易限额检查(RCTL)**

这个限额如果超了，要置位应用不被允许标志，不过visa一般建议这个开关关闭，也就是不检查。

 

**CVM限额检查**

这个如果超了，会把byte2的bit7置为1, 告诉卡片终端请求CVM做进一步的安全检查。

 

**非接最低限额检查**

这个如果超了，要把byte2的bit8置为1。

 

上述检查做完之后，最终的TTQ会在GPO命令中由终端送给卡片，然后卡片会进行行为分析决定交易接下来如何进行。另外，所有的检查都可以设置对应的开关来选择是否支持, 开关的实现方法没有特别要求，能起到开关作用就行。

### 数据检查和处理限制

如果做了读数据处理(非联机), 在读数据结束后,终端要检查是否已经读完所有的必要数据,比如, CID(tag 9f27), IAD(tag 9f10),AC(tag 9f36)以及ATC(tag9f36)等. 所以的必要的数据可以从规范中的数据表格中查到。必要的数据如果缺失，终端一定要终止交易。

处理限制的处理过程跟EMV基本是一样的，就是对卡片的有效期,应用用途控制以及异常文件的检查。

 

### 其它

我这次在做debug测试时，改过几处英文提示，这个问题看似简单，但是需要引起重视，尤其是终端真正在国外落地使用时，老外可能看不懂。比如说，挥卡的英文是tap the card或者present the card, 而不是swipe the card。还有一些交易结果的显示，一般只有三种，approve, decline and terminate。

 

在界面提示挥卡时，同时要显示当前的交易金额。交易完成后，要在凭条上打印以及屏幕上显示可用金额(AOSA)，不过如果做了issuer update processing(后面会讲到)就不能打印AOSA了。

 

**蜂鸣器提示音**

有几种情况需要加上提示音，首先是卡片可以移开场强区时，然后是交易脱机成功，再是交易失败。这几种情况最好通过频率和时间区分开提示效果。

**CVM**

终端要支持的CVM方法其实就两种，签名和联机pin，剩下就是看CTQ(tag 9f6C)和card authentication related data(tag 9f69)，这两个都是个卡片数据，9f6c第一个字节的编码如下:

```
Byte 1

bit 8: 1 = Online PIN Required

bit 7: 1 = Signature Required

bit 6: 1 = Go Online if Offline Data Authentication Fails and Reader is online capable.

bit 5: 1 = Switch Interface if Offline Data Authentication fails and Reader supports VIS.

bit 4: 1 = Go Online if Application Expired

bit 3: 1 = Switch Interface for Cash Transactions

bit 2: 1 = Switch Interface for Cashback Transactions

bit 1: RFU (0)
```

逻辑流程有点复杂，如果卡片CTQ没有返回，流程简单一些，终端优先执行签名，如果终端只支持联机pin, 就执行联机pin,如果两个都不支持，交易拒绝。卡片返回了CTQ的情况，可以参考规范的介绍，内容太多，这里不在赘述。


**联机与脱机**

对于联机交易, 和命令相关的流程只有应用选择和GPO, 脱机交易还有read record,不过DDA是在卡片离开感应区之后进行,不会影响交易的速度. 所以GPO命令中在联机情况下，没有AFL返回，这也是一个跟EMV的区别，因为在EMV中AFL是必须返回的。

需要联机的情况有很多，比如，卡片的有效期数据(tag 5f24)没有返回，终端要认为这是应用过期了，联机处理。再比如CVM执行联机pin的情况等。

脱机交易要求从卡片进行非接感应区上电开始，到最后一条read record结束，整个交互时间必须在500ms以内。


**关于fDDA**

这个是保证脱机交易安全的关键。fDDA只能算是一个精简的DDA，有以下不同点：


动态签名由GPO命令返回，不再由内部认证命令返回，而内部认证命令不再存在。


fDDA的结果不会联机发给发卡行(通过TVR反映出来，因为qVSDC中没有TVR)， 只是直接体现在脱机交易结果上。


参与生成动态签名的终端动态数据多了一些，EMV里一般就是只有9F37， qVSDC里还包括授权金额(9f02)，交易货币代码(5f2a)，卡片认证相关数据(9f69)这些TAG, 所以终端在验证签名时注意把这些TAG参与哈希运算。


**发卡行联机更新(issuer updateprocessing)**

交易联机时，发卡行有可能会在响应报文中下发脚本，终端负责检查脚本格式并转发给卡片，如果有脚本执行，后面在确认报文里还要上送脚本执行结果。


终端在下发脚本前，要对卡做外部认证，确保发卡行是合法的。

**form factor indicator**

Tag是9F6E, 4个字节。我人个理解,这个是后台用来辨别交易的终端特征和卡片特征，比如卡片是什么形式，可能是手机，终端交易时不必过多的关注这个东东，只要按普通tag保存它，然后联机时上送，不过在上送前要把第四个字节的b1~b4置为全0，表示终端是一台兼容14443协议的非接终端。


## 三 paywave的应用案例

在全球很多国家都有应用，其中以在韩国普及率最为明显，主要是有像三星,sk这样的巨头在推。比较知名的案例是跟星巴克共同推的用于店内支付的支付卡，还有像7-11这样的便利店也有应用。


在中国台湾，和花旗银行合作推一种购物卡。中国大陆地区目前只听说和工行合发过卡。


泰国的市场占有率也不错，也是跟几个大银行合作。

美国本土似乎不这么乐观，竞争太大，目前好像只有在洛杉矶和旧金山两个地区用的较多一些。



