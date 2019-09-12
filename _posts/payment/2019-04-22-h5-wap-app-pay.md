---
layout: post
title: 一文读懂H5，APP，WAP，公众号支付等多种支付方式的区别
category: payment
tags: [支付,H5,公众号支付,小程序支付]
---

* content
{:toc}


从事支付行业开发多年，做过很多不同的场景。发现各种支付方式多样化，还有各种不同的叫法，很多人都是一知半解，容易混淆一些概念。这篇文章希望根据自己的理解，尽量的把几种支付方式说清楚。

## 线上和线下

首先从大类上，任意一种支付方式必定属于这两类。线上和线下的区别在于商品交易的场景是在实体的店铺还是互联网上（比如电商平台）。

## 线下支付场景分类

### 付款码支付

也有叫条码支付的，也有叫被扫（从用户的角度）。**其实名字不重要，关键看场景。**它的场景是这样的:

商家使用扫码枪等条码识别设备扫描用户APP上的条码（一维码或者二维码），完成收款。用户仅需出示付款码，所有收款操作由商家端完成。支付宝的示例如下图：

![image](https://cdn.nlark.com/lark/0/2018/png/5909/1545380209392-875f14c7-22b2-468b-b394-dbc216fa4e5f.png#align=left&display=inline&height=436&originHeight=474&originWidth=815&status=done&width=750)
图片来自网络

具体步骤是：

1. 用户打开支付APP（支付宝，微信或者云闪付等），找到付款码界面；
2. 收银员在商家收银系统操作生成订单，用户确认支付金额；
3. 收银员使用扫码设备（包括扫码枪，POS机等），扫描用户手机上的条码（一维码或者二维码），商家收银系统提交支付。
4. 机构后台(支付宝，微信支付，银联等)收到支付请求在后台进行交易处理。
5. 付款成功（或者失败）后商家的收银系统和用户都会有相关的提示。

在第三步，支付机构会提供支付的接口，接口提供的方式有多种，比如有SDK的方式，还有HTTP的方式。请求的报文里携带付款码的信息（是一串数字，不同的支付机构特征不一样）。

这里还是拿支付宝的条码支付举个栗子：

``` bash
https://openapi.alipay.com/gateway.do?timestamp=2013-01-01 08:08:08&method=alipay.trade.pay&app_id=2284&sign_type=RSA2&sign=ERITJKEIJKJHKKKKKKKHJEREEEEEEEEEEE&version=1.0&biz_content=
  {
    "out_trade_no":"20150320010101001",
    "scene":"bar_code,wave_code",
    "auth_code":"28763443825664394",
    "subject":"Iphone6 16G",
    "seller_id":"2088102146225135",
    "total_amount":"88.88",
    "store_id":"NJ_001"
  }
```

其中auth_code就是付款码的信息。

这种场景和传统的POS收款其实异曲同工，本人早年从事过POS机支付相关的开发工作，POS机线下收款获取的是银行的信息，而条码支付获取的是用户APP上的付款码信息。获取信息之后都是往第三方付支付机构（包括银行，支付宝，微信等）发起支付请求并最终完成整个支付流程。


![图片来自网络](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1555928627530&di=31f054b50f449c73b7a83f16e45badb1&imgtype=0&src=http%3A%2F%2Fimg.mp.itc.cn%2Fupload%2F20170719%2F40c49b8a693145ddbd9b6c62f035f4e0_th.jpg)
图片来自网络


### 扫码支付

扫码支付也是一种线下的支付方式。它和付款码的区别扫码的主体互换了。是由用户使用APP扫描商户收银端生成的二维码。支付宝的示例如下图：

![image](https://cdn.nlark.com/lark/0/2018/png/5909/1545380470338-c1a669b0-1ff5-4848-8cab-e576216905fd.png#align=left&display=inline&height=356&originHeight=368&originWidth=828&status=done&width=800)
图片来自网络

具体步骤是：

1. 商家的收银系统根据用户购买的商品生成订单信息，并根据订单信息并生成二维码；
2. 用户打开APP的扫一扫界面，扫描第一步的二维码，核对金额然后支付；
3. 用户付款后商家收银系统会拿到支付成功或者失败的结果。同时用户本身也会收到相应的结果。

扫码支付的场景多见于自动售货机。大家仔细回想下在自动售货机上购买商品的过程就明白了。


![image](http://img.dav01.com/e/2018/6/21/davinfo_215546_1529562031320_1555442136.png)

图片来自网络

## 线上支付场景


### APP支付

场景是商家在APP上集成支付宝或者微信支付的功能。比如我们使用美团外面APP订外卖，付款的时候会弹出微信APP进行支付，这就是APP支付的场景。


示例如下图：

![image](https://gw.alipayobjects.com/os/skylark-tools/public/files/2db4858c9abd822d7ff47a414712b862.png%26originHeight%3D690%26originWidth%3D1126%26size%3D108352%26status%3Ddone%26width%3D746)
图片来自网络


步骤如下：

1. 用户进入商户APP，选择商品下单、确认购买，进入支付环节。商户服务后台生成支付订单数据，然后由商户APP调用支付SDK的接口；
2. 支付APP这个时候会被调用起来进行输入密码等操作并完成支付；

APP支付一般是提供手机端的封装好的SDK给到商户的APP调用，所以有安卓和IOS两个版本，这是和HTTP接口不同的地方。这里拿支付宝的安卓端SDK举个栗子(这里只展示关键部分的代码)：

``` java

	public void payV2(View v) {
		...
		final Runnable payRunnable = new Runnable() {

			@Override
			public void run() {
				PayTask alipay = new PayTask(PayDemoActivity.this);
				Map<String, String> result = alipay.payV2(orderInfo, true);
				Log.i("msp", result.toString());
				
				Message msg = new Message();
				msg.what = SDK_PAY_FLAG;
				msg.obj = result;
				mHandler.sendMessage(msg);
			}
		};

		...
	}
```

PayTask就是SDK提供的用于发起支付的核心类，通过调用它的方法payV2发起支付请求。payV2的第一个参数就是封装的订单信息对象。

### 公众号支付

微信叫公众号支付，支付宝叫生活号支付（也有叫支付宝服务窗支付的）。其实针对的场景是相似的，微信针对的是公众号里面的商家H5页面发起支付的场景，而支付宝针对的是生活号里的商家H5页面发起的支付场景。

这个拿大家熟悉的微信公众号举个栗子，具体的流程如下:

1. 进入公众号自定义菜单，用户点击进入商户网页选择购买商品；
2. 调起微信支付控件，用户输入支付密码完成支付；
3. 商户后台得到支付成功（或者失败）的通知；

![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_1_1.jpg)
![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_1_2.jpg)

![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_1_3.jpg)
![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter7_1_4.jpg)

图片来自网络

这里有几个关键的点说明以下：

* 支付过程并不会唤起微信APP，而是启动了一个微信支付的控件
* 支付请求参数中需要携带用户的openid，这个也是由公众号支付的特殊性决定的，在微信内置浏览器中是可以获取到用户的openid的；

### H5支付和手机WAP支付

这两个其实是一回事，只是叫法不同。场景都是在普通的浏览器（非支付宝和微信内置浏览器）打开商户的H5页面发起支付，唤起支付APP进行支付的情况。支付完后跳回到商家H5页面内，最后展示支付结果。


![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter15_3_1.jpg)
![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter15_3_3.jpg)
![image](https://pay.weixin.qq.com/wiki/doc/api/img/chapter15_3_4.jpg)

图片来自网络

据我的了解，这种场景应该是目前使用的最广泛的。比如我小区的电梯里经常看到一些蛋糕的广告，扫描上面的二维码就进入了商家的H5页面，然后选择商品购买即可，非常方便！


### 小程序支付

微信和支付宝现在都再力推小程序，那怎么能少得了支付的支持呢！

小程序的支付这里不详述了，因为它和公众号支付基本是一样的。就是在配置上有些差异，比如微信小程序支付不需要配置支付目录，授权域名这些了，简化了一些流程。



### 网关支付

网关支付也是属于线上支付，但它又不同于前面介绍的互联网支付方式。它属于传统的银行网银支付。一般情况下是符合资质的第三方支付机构提供的一种支付服务。

这些机构后台会接入多家银行，然后在前台放出接口给到商家。交易的时候需要在商家的收银系统上选择银行，然后通过第三方支付机构跳转到对应的银行网银完成支付流程。

![image](https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=1127709675,839655155&fm=26&gp=0.jpg)

图片来自网络

网关支付一般是在PC端的IE浏览器进行操作，移动端也有些机构提供，但是似乎兼容性一直不太好。场景一般是有大额往来的B2B业务。
