---
title: ThreadLocal
date: 2021-01-03 13:37:43
categories: 并发编程
tags: ThreadLocal
---

##  ThreadLocal

ThreadLocal是一个本地线程副本变量**工具类**。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现无状态的调用。适用于各个线程不共享变量值的操作。

##  ThreadLocal 工作原理

每个线程的内部维护一个ThreadLocalMap，它是一个Map(key,value)数据格式，key是一个弱引用。也就是ThreadLocal本身,而value存在是线程变量的值。

也就是说ThreadLocal本身并不存储线程的变量值，**它只是一个工具类，用来维护线程内部的Map,帮助存和取变量。**

<img src="https://jameslin23.gitee.io/2021/01/03/ThreadLocal/image-20210103140842502.png" alt="image-20210103140842502" style="zoom: 67%;" />

##  ThreadLocal解决Hash冲突

与HashMap不同，ThreadLocalMap结构非常简单，没有next引用,也就是ThreadLocalMap中解决Hash冲突的方式并非链表的方法，而是采用线性探测方法。所谓线性探测，就是根据key的hashcode值确定元素在table数组中的位置，如果发现这个位置已经被其它key值占用，则利用固定算法寻找一定步长的下一个位置，依次判断，直至找到能够存放的位置。

```java
/
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

/
 * Decrement i modulo len.
 */
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

##  ThreadLocal内存泄漏

ThreadLocal在ThreadLocalMap中是以一个弱引用身份被Entry中的key引用的，因此如果ThreadLocal没有外部强引用来引用它，那么ThreadLocal会在下次JVM垃圾收集时被回收。这个时候Entry中key已经被回收，但是value又是强引用不会被垃圾收集器回收。这样ThreadLocal的线程如果一直执行运行，value就一直得不到回收，这样就会发生内存泄漏。

ThreadLocalMap的key是弱引用

ThreadLocalMap中key是弱引用，而value是强引用才会导致内存泄漏的问题，至于为什么要这样设计，这样分2种情况来讨论：

- key使用强引用：这样会导致一个问题,引用的ThreadLocal对象被回收了，但是ThreadLocalMap还有持有ThreadLocal强引用，如果没有手动删除，ThreadLocal不会被回收，则会导致内存泄漏。
- key使用弱引用：这样的话，引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。

比较以上两种情况，我可以发现：由于ThreadLocalMap的生命周期跟Thread一样，如果都没有手动删除对应key,会导致内存 泄漏，但是使用弱引用可以多一层保障。

##  ThreadLocal的应用场景

ThreadLocal适用于独立变量副本的情况，比如Hibernate的session获取场景。

```java
private static final ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();

public static Session getCurrentSession(){
    Session session =  threadLocal.get();
    try {
        if(session ==null&&!session.isOpen()){
            //...
        }
        threadLocal.set(session);
    } catch (Exception e) {
        // TODO: handle exception
    }
    return session;
}
```

