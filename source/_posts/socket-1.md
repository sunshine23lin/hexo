---
title: socket初步认识
date: 2019-08-02 14:51:57
categories: Scoket
tags:
---

### 前言
1. 网络基础
两台计算如何进行通信
> IP:端口  <--> IP:端口

![](scoket-1/1.png)

2. TCP/IP协议
> a.TCP/IP是世界上应用最为广泛的协议,是以TCP和IP为基础的不同层次上多个协议的集合
> 也称为：TCP/IP协议族 或 TCP/IP协议栈
> TCP: Transmission Control Protocol 传输控制协议
> IP: Internet Protocol 互联网协议

3. TCP/IP模型
![](scoket-1/2.png)

4. 端口
> a.用于区分不同应用程序
> b.端口号范围为0~65535,其中0~1023为系统所保留
> c.Ip地址和端口号组成了所谓的Socket，Socket是网络上运行的程序之间双向通信链路的终结点,是TCP和UDP的基础。
> http:80  ftp:21 telnet:23

5. TCP详解
> 转载 https://blog.csdn.net/sinat_36629696/article/details/80740678

6. UPD协议
> 转载 https://blog.csdn.net/china_jeffery/article/details/78923428

### 一、Socket初步认识
1. 什么是Socket编程
> Socket是应用层与TCP/IP协议族通信的中间抽象层,它是一组接口。在设计模式中,Socket其实就是一个门面模式,它把复杂的TCP/IP协议族隐藏在Socket接口后面,对于用户来说,一组简单的接口就是全部让Socket去组织数据,以符合指定的协议。
例子：你要打电话给一个朋友，先拨号，朋友听到电话铃声后提起电话，这时你和你的朋友就建立起了连接，就可以讲话了。等交流结束，挂断电话结束此次交谈。 生活中的场景就解释了这工作原理

![](scoket-1/3.png)
2. 