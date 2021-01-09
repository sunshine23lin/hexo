---
title: JVM篇之垃圾回收器Parallel
date: 2020-12-30 20:58:15
categories: JVM
tags: 垃圾回收器
---

##  背景

<img src="https://jameslin23.gitee.io/2020/12/30/JVM篇之垃圾回收器Serial/image-20201230202351458.png" alt="image-20201230202351458" style="zoom:50%;" />

图中展示了7种作用于不同分代的收集器，如果两个收集器相连，说明他们可以搭配使用。虚拟机所处区域表示它是属于新生代还是老年代收集器

**新生代收集器**：Serial、ParNew、Paraller Scavenge

**老年代收集器**：CMS、Serial Old、Parallel Old

**整堆收集器**：G1

##  Parallel Scavenge

吞吐量优先的垃圾回收器，**jdk1.8**默认垃圾回收器

**特点：**属于新生代收集器，复制算法，并行的多线程收集器。

该收集器的目标是达到一个可控制的吞吐量，还有一个值得关注点：GC自适应调节策略

**GC自适应调节策略**：可设置-XX:+UseAdptiveSizePolicy参数，当开关打开时不需要手动指定新生代的大小（-xmn）、Eden与Surivor区的比例（-XX:SurvivorRation）、晋升老年代年龄（-XX：PretenureSizeThreshold）等。虚拟机会根据系统的运行状况收集性能监控信息，动态设置这些参数以提供最优的停顿时间和最高的吞吐量。

Parallel Scavenge收集器使用两个参数控制吞吐量：

- **XX:MaxGCPauseMillis** 控制最大垃圾收集停顿时间
- **XX:GCRatio** 直接设置吞吐量大小

##  Parallel Old

是Parallel Scavenge收集器的老年代版本

特点：多线程，采用标记-压缩算法

Ps/Po工作示意图

<img src="https://jameslin23.gitee.io/2020/12/30/JVM篇之垃圾回收器Parallel/image-20201231083018420.png" alt="image-20201231083018420" style="zoom:80%;" />