---
title: Synchronized分析篇
date: 2020-12-28 20:24:22
categories: 并发编程
tags: Synchronized
---

##  Synchronized

synchronized是java提供的原子性内置锁（存在对象头里面），这种内置的并且使用者看不到的锁也称为监视器锁，使用synchronized之后，会在编译之后的同步代码块前面加上monitorenter和monitorexit字节码指令，他依赖操作系统底层互斥锁实现。它的作用主要就是要实现原子性操作和解决共享变量的内存可见性。

执行monitorenter指令时会尝试获取对象锁，如果对象没有被锁或者已经获得锁，锁的计数器+1.此时其它竞争锁的线程则进入等待队列中。

执行monitorexit指令会把计数器-1，当计数器值为0时，则锁释放，处于等待队列中的线程再继续竞争。

synchronized是排它锁，当一个线程获得锁之后，其它线程必须等待该线程释放锁才能获得锁，而且由于java中的线程和操作系统原生线程是一一对应，线程被阻塞或者唤醒时会从用户态切换到内核态，非常消耗性能。

Synchronized实际有两个队列waitSet和entryList。

```java
ObjectMonitor() {
    _count        = 0; //记录个数
    _owner        = NULL; // 运行的线程
    //两个队列
    _WaitSet      = NULL; //调用 wait 方法会被加入到_WaitSet
   _EntryList    = NULL ; //锁竞争失败，会被加入到该列表
  }
```

