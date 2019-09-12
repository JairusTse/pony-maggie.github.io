---
layout: post
title: pybitcointools源码分析之BIP32实现
category: blockchain
tags: [blockchain,比特币,pybitcointools,bip32]
---

在看本篇之前，需要了解一个很重要的背景知识。那就是 **HD钱包**和 比特币协议 **BIP32**。

关于HD钱包的概念，建议大家去看看<<精通比特币>>。BIP32可以看下下面这篇翻译:

http://blog.csdn.net/pony_maggie/article/details/76178228

-------------

开始源码分析。


```
master = bip32_master_key(safe_from_hex("000102030405060708090a0b0c0d0e0f"))
```

safe_from_hex把字符串形式的16进制数字转换成byte形式， 例如"1234"->b"\x12\x34"。
比较简单不详述。



bip32_master_key函数是用于产生符合bip32的主密钥。那边问题来了，什么是 bip32的主密钥呢？

根据bip32约定，主密钥是从一个短种子值生成的，步骤如下:


* 从（P）RNG生成所选长度（128到512位;建议256位）的种子字节序列S。

* 计算I = HMAC-SHA512（Key =“Bitcoin seed”，Data = S）

* 将I分为两个32字节序列，IL和IR。

* 使用parse256（IL）作为主密钥，IR作为主链码。

-----------

有了上面的理论支撑，再来看代码就比较容易理解了。

```
def bip32_master_key(seed, vbytes=MAINNET_PRIVATE):
    I = hmac.new(from_string_to_bytes("Bitcoin seed"), seed, hashlib.sha512).digest()

    return bip32_serialize((vbytes, 0, b'\x00'*4, 0, I[32:], I[:32]+b'\x01'))
```

I是hmac-sha512算法计算得到的，用的key是"Bitcoin seed"，data是前面传过来的短种子值: 

```
b"\x00\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f"
```

I[32:]就是IR， I[:32]是IL。根据规范IL就可以作为主密钥了，那bip32_serialize是干啥的呢？

原来是为了方便表示，bip32引入序列化的概念，过程如下:


* 4字节：版本字节（mainnet：0x0488B21E public，0x0488ADE4 private; testnet：0x043587CF public，0x04358394 private）

* 1字节：深度：主节点为0x00，级别1派生密钥为0x01。

* 4字节：父密钥的指纹（如果主密钥为0x00000000）

* 4字节：子数字。这是对于i在xi = xpar / i中的ser32（i），其中xi是键序列化。 （如果主密钥为0x00000000）

* 32字节：链码

* 33字节：公钥或私钥数据（公钥的serP（K），私钥的0x00 || ser256（k））

可以通过首先添加32个校验和位（从双SHA-256校验和派生），然后转换为Base58表示。


bip32_serialize入参有六个，我们可以和上面一一对应下，

vbytes是版本字节，0x0488ADE4。

0表示深度，这里是主密钥，深度表示为0。

b'\x00'*4在python中就是b'\x00\x00\x00\x00'，对应父密钥的指纹。

接下来的0对应子数字，在bip32_serialize函数里会转为4字节。

I[32:]也叫IR，对应链码。

I[:32]+b'\x01'，33字节对应私钥数据(这里是私钥)。



进入bip32_serialize里面，


```
def bip32_serialize(rawtuple):
    vbytes, depth, fingerprint, i, chaincode, key = rawtuple
    i = encode(i, 256, 4)

    keydata = b'\x00'+key[:-1] if vbytes in PRIVATE else key
    bindata = vbytes + from_int_to_byte(depth % 256) + fingerprint + i + chaincode + keydata

    return changebase(bindata+bin_dbl_sha256(bindata)[:4], 256, 58)

```

i的值是0， i = encode(i, 256, 4)把0转换为b'\x00\x00\x00\x00'。


keydata是就是公钥或者私钥数据(这里是私钥)。

最后拼接然后转换为base58表示。

-----------------


下面看看如何用主密钥衍生出第一个子密钥。

```
bip32_ckd(master, "0")
```

参数0是一个索引值，表示第一个子密钥。

```
def bip32_ckd(data, i):
    return bip32_serialize(raw_bip32_ckd(bip32_deserialize(data), i))
```

bip32_serialize这个之前说过了，bip32_deserialize很明显是相对的，反序列化。就是把主密钥再变回元组的表示。

所以核心的函数是raw_bip32_ckd。在分析这个函数之前还是要先来点理论知识，看看BIP32里对于主密钥(私钥)衍生子密钥是怎么说的，


函数CKDpriv（（kpar，cpar），i）→（ki，ci）从父扩展私钥计算子扩展私钥：

1. 检查 是否 i ≥ 2^31(子私钥)。

    如果是（硬化的子密钥）：让I= HMAC-SHA512（Key = cpar，Data = 0x00 || ser256（kpar）|| ser32（i））。 （注意：0x00将私钥补齐到33字节长。）

    如果不是（普通的子密钥）：让I= HMAC-SHA512（Key = cpar，Data = serP（point（kpar））|| ser32（i））。




2. 将I分为两个32字节序列，IL和IR。


3. 返回的子密钥ki是parse256（IL）+ kpar（mod n）。


4. 返回的链码ci是IR。

**如果parse256（IL）≥n或ki = 0，则生成的密钥无效，并且应继续下一个i值。 （注：概率低于1/2127）**



```
def raw_bip32_ckd(rawtuple, i):
    vbytes, depth, fingerprint, oldi, chaincode, key = rawtuple
    i = int(i)

    if vbytes in PRIVATE:
        priv = key
        pub = privtopub(key)
    else:
        pub = key

    if i >= 2**31:
        if vbytes in PUBLIC:
            raise Exception("Can't do private derivation on public key!")
        I = hmac.new(chaincode, b'\x00'+priv[:32]+encode(i, 256, 4), hashlib.sha512).digest()
    else:
        I = hmac.new(chaincode, pub+encode(i, 256, 4), hashlib.sha512).digest()

    if vbytes in PRIVATE:
        newkey = add_privkeys(I[:32]+B'\x01', priv)
        fingerprint = bin_hash160(privtopub(key))[:4]
    if vbytes in PUBLIC:
        newkey = add_pubkeys(compress(privtopub(I[:32])), key)
        fingerprint = bin_hash160(key)[:4]

    return (vbytes, depth + 1, fingerprint, i, I[32:], newkey)

```

代码比较简单，都是按照协议的流程编写的。这里只需要特别说明密钥指纹(fingerprint)的计算规则，

```
fingerprint = bin_hash160(privtopub(key))[:4]
```

根据BIP32,扩展密钥可以由序列化的ECSDA公钥K的Hash160（SHA256之后的RIPEMD160）标识。 