---
layout: post
title: PBOC/EMV之TLV编码与解码
copyright: java
category: payment
tags: [pboc,payment]
---

PBOC的IC卡大部分数据都是TLV格式的. EMV的手册简单的编码规则说明. 我下面很详细的分析TLV的编码格式并给出相应的TLV解码的伪代码.


TLV是tag, length和value的缩写.一个基本的数据元就包括上面三个域. Tag唯一标识该数据元, length是value域的长度. Value就是数据本身了. 举个例子, 下面是一个tlv格式的AID（应用标识符）字节串”9F0607A0000000031010”, 其中9F06是tag, 07是长度, A0000000031010就是AID本身的值了.

 

开发人员应该关心的是，如果有类似上面这样的一串TLV编码的字节串从卡片传过来, 怎么样从中提取我们想要的数据? 这就就是TLV解码的问题了.

 

其中BER-TLV编码是ISO定义一种规范, 然后到了PBOC/EMV里被简化了. 举一个例子, tag域在ISO里可以有多个字节,而PBOC/EMV里规定只用前两个字节. 我下面要讲的TLV解码就是基于PBOC/EMV的简化版本.

 

首先看一下tag域是怎样编码的. Tag域最多占两个字节. 编码规则如下面两幅图


![](http://513.img.pp.sohu.com.cn/images/blog/2009/11/18/15/3/125b598af7eg215.jpg)

![](http://1862.img.pp.sohu.com.cn/images/blog/2009/11/18/15/3/125b59c959bg213.jpg)



解释一下这两幅图. 第一个图是第一个字节的编码规则. b8和b7两位标识tag所属类别. 这个可以暂时不用理.  b6决定当前的TLV数据是一个单一的数据和复合结构的数据. 复合的TLV是指value域里也包含一个或多个TLV, 类似嵌套的编码格式. b5~b1如果全为1，则说明这个tag下面还有一个子字节. 占两个字节, 否则tag占一个字节.


第二幅图是说明如果tag占用两个字节, 第二个字节的编码格式. B8决定tag是否还有后绪的字节存在，因为前面说过，PBOC/EMV里的tag最多占两个字节, 所以该位保持为0.

 

清楚了上面tag编码格式,可很容易写出tag域解码的代码了. 假设，终端接收到字节串，这个字节串保存在tlvData的字节数组里, 伪代码如下:

```
if ( (tlvData[i]&0x20) != 0x20)//单一结构
 
{
 
      if ( (tlvData[i]&0x1f) == 0x1f)//tag两字节
 
       {
 
             tagIndex++;
 
                    
 
             //解析length域
 
               //解析value域
 
       }
 
     else//tag单字节
 
      {
 
          //解析length域
 
            //解析value域
 
      }
 
}
 
 else//复合结构
 
 {
 
          //复合结构可以考虑用递归的方法来实现.
 
 }

```

Length域的编码比较简单,最多有四个字节, 如果第一个字节的最高位b8为0, b7~b1的值就是value域的长度. 如果b8为1, b7~b1的值指示了下面有几个子字节. 下面子字节的值就是value域的长度.Value域的编码格式要根据具体的value所表示的数据元决定. 比如AID是由RID+PIX构成等. 这个不详述. 有了上面的知识，基本上可以写一个TLV解码器出来了.    




