---
layout: post
title: PBOC/EMV之建立应用列表
copyright: java
category: payment
tags: [pboc]
---

建立应用列表有两种方法, PSE目录选择方法和AID选择方法. 后者比较简单，看规范容易理解. 这篇文章只介绍目录选择方法.

 

步骤是这样的:

1 终端选择PSE，从卡片返回的FCI中读到SFI.

PSE本身是一个EF文件,所以通过SFI来定位, EF文件又是由很多记录组成, 通过read record来读取. 记录的个数不一定, 终端应该从第一条记录一直读，一直读到卡片返回6A83. 格式如下图:



![](http://hi.csdn.net/attachment/201203/6/0_1331036797VasP.gif)


2 终端通过read record读取SFI中的所有记录. 每读到一条记录, 就去找入口. 看上图可知, 每一条记录是一个T(70)LV, 这个V里面是由很多条T(61)LV,  终端要注意处理这种嵌套结构. 这个T(61)LV的V(也就是入口)可能是ADF，也可能是DDF. 如下图:

![](http://hi.csdn.net/attachment/201203/6/0_1331036826j28P.gif)

![](http://hi.csdn.net/attachment/201203/6/0_1331036897o4Jz.gif)

注意上面两图, 格式中的TAG顺序可以不是固定的. 比如说，在第二个图中，tag ‘4F’可以出现在’50’之后等等. 我以前一直以为这个顺序是固定的, 直到2012.12.1 EMV l2的案例升级，新增了一条案例专门测试这一点，才知道这个顺序是可以变的. 这条案例是这样写的:

 
```
Objective:

To ensure that the terminal accepts a response to READ RECORD on

a Payment Ststem Directory File with data stored in an order different

from example given in EMV Specifications.

Conditions:

LT returns ADF Entry Format data in response to READ

RECORD of the Payment System Directory in that order: ‘50’

Application Label, ‘9F12’ Application Preferred Name, ‘4F’ ADF

Name and ‘87’ Application Priority Indicator
```

如果入口是ADF，那就简单了,直接取4F的值就可以了, 然后索引指向下一条. 难点在于, 如果是DDF，那终端选择这个DDF，然后取FCI中的SFI,然后读这个SFI中的记录. 说到这里, 大家应该已经看出来了，入口是DDF的情况, 相当于又回到了步骤1, 只是把选择PSE变为选择DDF.

这种情况下，就需要保存进入DDF处理前的断点信息，以便处理完这个DDF后，可以继续当前的处理过程. 这个有点像CPU中的嵌套中断, 它的原理是利用堆栈来保存断点处的信息, 处理完中断就恢复堆栈.  这个处理过程中体现的程序上一般是有两种处理方法, 一是递归， 二是非递归. 递归的方法处理起简单，易读,但是效率比较低. 不过EMV的测试案例中的记录格式一般都比较简单，嵌套的不深. 下面给出伪代码的流程,假设建立列表的函数名为BuildListByDDF

``` c

BuildListByDDF(char *DDF)
 
{
 
    选择PSE（DDF）;
 
    读取SFI
 
    While(读SFI中的记录不返回6A83)
   {
 
      If(取tag70成功)
      {
 
          If(取tag61成功)
         {
 
             If(取到的入口是4F )
            {
                对比终端AID，如果一致(部分匹配或完全匹配)，就加入候选列表;
            }
 
           else If(取到的入口是9D)
            {
                   BuildListByDDF(9D的DDF值);//注意这里递归
                     标志位置位;
            }
 
      }
 
  }
 
 
 
    If(标志位置位)
/*标志位的作用是为了重新选择卡片，使卡片再被递归中的选择后, 恢复的到当前的状态*/
 
   {
 
      重新选择当前的DDF，并读到SFI
 
   }
 
 }
 
}
 
```

3 经过第二步之后, 候选列表基本建立起来了, 最后一步是根据AID的优先级参数进行排序, 一般我们会用一个数组变量保存应用列表, 可以让优先级最高的应用放在数组的最开始位置. 这样便于在不支持持卡人确认时，自动选择优先级最高的应用.


