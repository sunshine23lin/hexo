---
title: JVM篇之复制算法
date: 2020-12-30 18:20:19
categories: JVM
tags: 垃圾回收算法
---

##  复制算法

###  背景

为了解决标记-清除算法在垃圾收集效率方面的缺陷，M.L.Minsky于1963年发表了著名的论文，“使用双存储区的Lisp语言垃圾收集器CA Lisp Garbage Collector Algorithm Using Serial Secondary Storage”。M.L.Minsky在论文中描述的算法被人们称为复制（Copying）算法，它也被M.L.Minsky本人成功的引入到了Lisp语言的一个实现版本中。

###  核心思想

将活着的内存空间分为两块，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中，之后清除正在使用的内存块中的所有对象，交换两个内存的角色，最后完成垃圾回收。

##  内存图示

<img src="https://img-blog.csdnimg.cn/20200729095325197.png" alt="img" style="zoom:80%;" />

##  优点

- 高效，且不产生碎片化

##  缺点

- 耗内存，需要两倍内存空间

##  运用场景

用于新生代，由于复制算法需要复制存活的对象到另外一边，所以新生代存活率并不高，所以适合使用新生代。