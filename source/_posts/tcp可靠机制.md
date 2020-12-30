---
title: tcp可靠机制
date: 2020-12-16 09:56:00
categories: 计算机网络
tags: 计算机网络
---

##  前言

为了实现可靠性传输，需要考虑很多事情，例如数据的破坏、丢包、重复以及分片顺序混乱等问题。如不能解决这些问题，也就无从谈起可靠传输。

那么，TCP 是通过序列号、确认应答、重发控制、连接管理以及窗口控制等机制实现可靠性传输的

今天，将重点介绍 TCP 的**重传机制、滑动窗口、流量控制、拥塞控制。**

![image-20201216100004410](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201216100004410.png)

         ##  重传机制

###   超时重传

指在设置时间内未收到对方ACK确认应答报文，就会重发数据。

如果超时重发的数据，再次超时的时候，又需要重传时候，TCP的策略是超时间隔加倍。

**每当遇到一次超时重传的时候，都会将下一次超时时间间隔设为先前值的两倍。两次超时，就说明网络环境差，不宜频繁反复发送。**

**存在问题:**

超时周期可能相对较长

![image-20201216102028045](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201216102028045.png)

###  快速重传

快速重传的工作方法是当收到三个相同ACK报文时，会在定时器过期之前，重传丢失报文。

**存在问题：**

重传的时候，是重传之前的一个，还是所有？

![image-20201216102158624](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201216102158624.png)

###  SACK

SACK选择性确认

在TCP头部选项字段加一个SACK的东西，缓存发送的序列化，然后发送给发送方，让发送方可以知道哪些数据收到，哪些数据没收到。

知道这些信息，就可以只重传丢失的数据。

![image-20201216102638128](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201216102638128.png)

如果要支持 `SACK`，必须双方都要支持。在 Linux 下，可以通过 `net.ipv4.tcp_sack` 参数打开这个功能（Linux 2.4 后默认打开）。

###  D-SACK

使用SACK来告诉有哪些数据被重复接收了

**ACK丢包**

![image-20201216102903448](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201216102903448.png)

- 「接收方」发给「发送方」的两个 ACK 确认应答都丢失了，所以发送方超时后，重传第一个数据包（3000 ~ 3499）
- **于是「接收方」发现数据是重复收到的，于是回了一个 SACK = 3000~3500**，告诉「发送方」 3000~3500 的数据早已被接收了，因为 ACK 都到了 4000 了，已经意味着 4000 之前的所有数据都已收到，所以这个 SACK 就代表着 `D-SACK`。
- 这样「发送方」就知道了，数据没有丢，是「接收方」的 ACK 确认报文丢了。

**网络延迟**

![图片](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZeuicRMlA8rKvl5AVLibhibDhgGSKaMMbmPiaUQmCvR4cz2kQ5OV8SqaPIwdkZ7T19uEs5WxCc6h66x7w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 数据包（1000~1499） 被网络延迟了，导致「发送方」没有收到 Ack 1500 的确认报文。
- 而后面报文到达的三个相同的 ACK 确认报文，就触发了快速重传机制，但是在重传后，被延迟的数据包（1000~1499）又到了「接收方」；
- **所以「接收方」回了一个 SACK=1000~1500，因为 ACK 已经到了 3000，所以这个 SACK 是 D-SACK，表示收到了重复的包。**
- 这样发送方就知道快速重传触发的原因不是发出去的包丢了，也不是因为回应的 ACK 包丢了，而是因为网络延迟了。

##  滑动窗口

##   流量控制

##  拥塞控制

