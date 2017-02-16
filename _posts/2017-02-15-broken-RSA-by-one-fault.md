---
layout: post
title: "一个位错误导致RSA私钥泄露"
categories: notes
tags: 密码学
---

论文：[On the Importance of Checking Cryptographic Protocols for Faults](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.48.9764&rep=rep1&type=pdf)

如图：

![](/store/post_data/RSA1.png)

适用于服务器端用私钥进行签名，客户端用公钥解密的场景。

如果服务器端用中国剩余定理来加速签名过程，如果计算过程中两个E中其中一个发生错误，则攻击者可以利用得到的错误计算结果反推出两个质数。

顺便复习了下RSA的加密原理。

![](/store/post_data/RSA2.png)

公钥由(N, e)组成，私钥由(N, d)组成。e由实现固定，一般取65537；由于私钥中包含了N，因此一般可以比较容易的从私钥得到公钥，反之不能。(唯一的秘密就是私钥里的d了，一般来说两个素数可以丢弃)



