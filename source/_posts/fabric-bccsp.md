---
title: Fabric 1.4源码解读 10：可扩展密码服务提供者BCCSP，以及可扩展国密
date: 2020-04-12 14:32:33
tags: ['区块链', 'Fabric', '密码学']
---

## 序言

密码学是当代数字信息化时代的基础技术，没有密码学，网络上的传输信息的可靠性就无法保证，比如你输入的密码会被窃取，你存在网络上的照片、文档如果没有加密，就有可能泄露。

密码学也是区块链的一项基础技术，使用密码学实现区块链中的：身份验证、数据可信、权限管理、零知识证明、可信计算等等。

Fabric提供了模块化的、可插拔的密码服务，该服务由`bccsp`模块提供，本文就谈一下BCCSP插件化设计，另外Fabric国密化也是最近2年必做的事情，所以同时介绍实现可扩展国密的思路，最后介绍一下Hyperledger社区对Fabric支持国密的开发。



## BCCSP介绍

BCCSP是Block Chain Crypto Service Provider的缩写。

`bccsp`模块它为Fabric的上层模块提供密码学服务，它包含的具体功能有：对称加密和非对称加密的密钥生成、导如、导出，数字签名和验证，对称加密和解密、摘要计算。

`bccsp`模块为了密码服务的扩展性，定义了`BCCSP`接口，上层模块调用`BCCSP`接口中定义的方法，而不直接调用具体的实现函数，实现和具体密码学实现的解耦，当`bccsp`使用不同密码学实现时，上层模块无需修改，这种解耦是通过**依赖反转**实现的。

bccsp模块中当前有2种密码实现，它们都是bccsp中的密码学插件：SW和PKCS11，SW代表的是国际标准加密的软实现，SW是software的缩写，PKCS11代指硬实现。

![](https://lessisbetter.site/images/2020-04-fabric-bccsp.png)

> 扩展阅读：PKCS11是PKCS系列标准中的第11个，它定义了应用层和底层加密设备的交互标准，比如过去在电脑上，插入USBKey用网银转账时，就需要走USBKey中的硬件进行数字签名，这个过程就需要使用PCKS11。

密码学通常有软实现和硬实现，软实现就是常用的各种加密库，比如Go中`crypto`包，硬实现是使用加密机提供的一套加密服务。软实现和硬实现的重要区别是，密码算法的安全性强依赖随机数，软实现利用的是OS的伪随机数，而硬实现利用的是加密机生成的随机数，所以硬实现的安全强度要高于软实现。

让Fabric支持国密时，就需要在bccsp中新增一个国密插件`GM`，只在bccsp中增加GM并不是完成的Fabric国密改造，下文再详细介绍。

## SW介绍

SW是国际标准加密的软实现插件，它包含了ECDSA算法、RSA算法、AES算法，以及SHA系列的摘要算法。

`BCCSP`接口定义了以下方法，其实对密码学中的函数进行了一个功能分类：
- `KeyGen`：密钥生成，包含对称和非对称加密
- `KeyDeriv`：密钥派生
- `KeyImport`：密钥导入，从文件、内存、数字证书中导入
- `GetKey`：获取密钥
- `Hash`：计算摘要
- `GetHash`：获取摘要计算实例
- `Sign`：数字签名
- `Verify`：签名验证
- `Encrypt`：数据加密，包含对称和非对称加密
- `Decrypt`：数据解密，包含对称和非对称加密

SW要做的是，把ECDSA、RSA、AES、SHA中的各种函数，对应到以上各种分类中，主要的分类如下图所示。

![](https://lessisbetter.site/images/2020-04-12-bccsp-sw.png)

从上图可以看出，密钥生成、派生、导入都包含了ECDSA、RSA、AES，签名和延签包含了ECDSA和RSA，摘要计算包含了SHA系列，加密解密包含了AES，但没有包含RSA，是因为非对称加密耗时，并不常用。

## 可插拔国密

Fabric支持国密并非仅仅在bccsp中增加1个国密实现这么简单，还需要让数字证书支持国密，让数字证书的操作符合X.509。各语言的标准库`x509`都是适配标准加密的，并不能直接用来操作国密证书。

在数字证书支持国密后，还可能需要进一步考虑，是否需要TLS证书使用国密数字证书，让通信过程使用国密算法。

另外，国密的实现有很多版本，如果需要适配不同的国密实现，就需要保证国密的可插拔和可扩展。

综上情况，你需要一个中间件，中间件中包含定义好国密接口、国密数字证书接口等，用这些接口去适配Fabric，然后当采用不同国密实现时，只需要对具体实现进行封装，去适配中间件中定义好的接口。

![](https://lessisbetter.site/images/2020-04-fabric-gm.png)

## 社区对Fabric支持国密的态度

国密有很多基于Fabric的项目，金融业是区块链场景最多的行业，金融行业又必须使用国密，所以国内对Fabric国密的改造是必须的，在《金融分布式账本安全规范》发布之后，社区也计划让Fabric支持国密，但方式是不提供具体国密实现，而是定义好接口，项目方使用哪种国密实现，去适配定义好的接口即可，这样保留了好的扩展性，与[可插拔国密](#可插拔国密)的目的是一致的，选择权交给企业。

社区支持Fabric国密的版本，预计在2.x版本发布。

## 结语

密码学在区块链中的地位是相当高的，从区块链使用最基础的密码学，到现在还在不断融入同态加密、零知识证明等前言的加密技术，未来可以在区块链上保护数据隐私的情况，提供更好的服务，区块链也可以有更多的应用场景。