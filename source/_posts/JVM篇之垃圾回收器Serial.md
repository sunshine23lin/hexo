---
title: JVM篇之垃圾回收器Serial
date: 2020-12-30 20:21:10
categories: JVM
tags: 垃圾回收器
---

##  背景

<img src="https://jameslin23.gitee.io/2020/12/30/JVM篇之垃圾回收器Serial/image-20201230202351458.png" alt="image-20201230202351458" style="zoom:50%;" />

图中展示了7种作用于不同分代的收集器，如果两个收集器相连，说明他们可以搭配使用。虚拟机所处区域表示它是属于新生代还是老年代收集器

**新生代收集器**：Serial、ParNew、Paraller Scavenge

**老年代收集器**：CMS、Serial Old、Parallel Old

**整堆收集器**：G1

##  Serial

serial收集器是最基本、发展历史最悠久的收集器。

**特点：**单线程、简单高效，对应当个CPU的环境，Serial收集器由于没有线程交互的开销，专心做垃圾回收自然获得最高单线程效率，

收集进行垃圾时，必须暂停其他所有的工作线程（Stop The World）,直到它结束。

**运用场景**：适用于Clinet模式下的虚拟机

Serial/Serial Old收集器运用图

<img src="https://jameslin23.gitee.io/2020/12/30/JVM篇之垃圾回收器Serial/image-20201230203617306.png" alt="image-20201230203617306" style="zoom: 80%;" />

##  Serial Old

Serial Old是Serial收集器的老年代版本

**特点**：同样是单线程收集器，采用标记-压缩算法

**应用场景**：适用于Clinet模式下的虚拟机

- 在JDK1.5以及以前的版本中与Parallel Scavenge收集器搭配使用
- 作为CMS收集器的后备方案，在并发收集Concurent Mode Failure时使用

##  ParNew

ParNew收集器其实是Serial收集器的多线程版本。

除了使用多线程其余均行为均和Serial收集器一模一样（参数控制、收集算法、Stop the World、对象分配规则、回收策略）

**特点：**多线程、ParNew收集器默认开启的收集线程与CPU的数量相同，在CPU非常多的环境中，可以使用-XX:ParallelGCThreads参数来限制垃圾回收器的线程数。

**应用场景：**ParNew收集器是许多运行在Server模式下的虚拟机中首选新生代收集器，他经常与CMS搭配。

ParNew/Serial Old组合收集器示意图

<img src="https://jameslin23.gitee.io/2020/12/30/JVM篇之垃圾回收器Serial/image-20201230205646468.png" alt="image-20201230205646468" style="zoom:80%;" />