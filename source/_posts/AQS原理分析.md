---
title: AQS原理分析
date: 2020-12-27 15:17:02
categories: 并发编程
tags: AQS
---

##  AQS概述

AQS(Abstract Queued Synchronizer) 抽象队列同步器，定义了一套多线程访问共享资源的同步框架，许多同步类实现都依赖与它，常用ReentrantLock/Semaphoe/CountDownLatch

由一个资源状态int state和同步器（FIFO双向阻塞队列组成）

![image-20201227152453788](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227152453788.png)

###  同步器

同步器依赖内部的同步队列（FIFO）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会使用**CAS**将当前线程以及等待状态等信息构造成一个节点Node并将其加入同步队列**尾部**，同时会阻塞当前线程，当同步状态释放时，会把首节点的线程唤醒，并其再尝试获取同步状态。

同步队列的节点Node用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点。

同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后续节点，而后续节点将会在获取同步状态成功时将自己设置为首节点。

**注：由于获取成功状态的线程只有一个，所有设置首节点不需要使用cas，而插入尾部竞争的线程有很多个，所以需要使用CAS。**

![image-20201227170001772](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227170001772.png)

###  独占式

同步器的acquire方法

```java
public final void acquire(int arg){
    if(!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE),arg))
        selfInterrupt;
}
```

上述代码主要完成同步锁状态获取、节点构造、加入同步队列以及在同步队列中自旋等待相关工作。

其主要逻辑是：首先调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点并通过addWaiter(Node node)方法将节点加入到同步队列的尾部,最后调用acquireQueued(Node node,int arg)方法，**使得该节点以“死循环”的方式获取同步状态。如果获取不到则阻塞节点的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或者阻塞线程中断来实现。**

当前线程在“死循环”中获取同步状态，而只有前驱节点是头节点才能尝试获取同步状态，这个为什么？

- **头节点是成功获取到同步状态的节点，而头节点的线程释放了同步状态之后，将会唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点。**
- **维护同步队列的FIFO原则**

**总结：在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程通过CAS加入到队列尾部并在队列中；移除队列的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态。然后唤醒头节点后继节点。**

###  共享式

共享式获取与独占式获取最主要的区别是在与同一个时刻能有多个线程获取同步状态

在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态，tryAcquireShared(int arg)方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是**tryAcquireShared(int arg)方法返回值大于等于0**。可以看到，在doAcquireShared(int arg)方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。

与独占式一样，共享式获取也需要释放同步状态，通过调用releaseShared(int arg)方法可以释放同步状态

```java
public final boolean releaseShard(int arg){
    if(tryReleaseShard(arg)){
        	doReleaseShared();
            return true;
    }
    reture false;
}
```

该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点。对于能够支持多个线程同时访问的并发组件（比如Semaphore），它和独占式主要区别在于tryReleaseShared(int arg)方法必须确保同步状态（或者资源数）线程安全释放，一般是通过**循环**和**CAS**来保证的，因为释放同步状态的操作会同时来自多个线程。

##  Condition接口

任意一个java对象，都拥有一组监视器方法（定义在object上）,主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与Synchronized同步关键字配合,可以实现等待/通知。condition接口也提供了类似Object的监听器方法，与Lock配合可以实现等待/通知模式。

Object的监听器方法与Condition接口的对比

| 对比项                                               | Object     | Condition                                                    |
| ---------------------------------------------------- | ---------- | ------------------------------------------------------------ |
| 前置条件                                             | 获取对象锁 | 调用Lock.lock()获取对象锁，调用Lock.newCondition()获取Condition对象 |
| 调用方式                                             | 直接调用   | 直接调用，例如condition.await()                              |
| 等待队列个数                                         | 一个       | 多个                                                         |
| 当前线程释放锁进入等待状态                           | 支持       | 支持                                                         |
| 当前线程释放锁并进入等待状态，在等待状态中不响应中断 | 不支持     | 支持                                                         |
| 当前线程释放锁并进入超时等待状态                     | 支持       | 支持                                                         |
| 当前线程释放锁并进入等待状态到将来的某个时间         | 不支持     | 支持                                                         |
| 唤醒等待队列中一个线程                               | 支持       | 支持                                                         |
| 唤醒等待队列中全部线程                               | 支持       | 支持                                                         |

###  实现分析

ConditionObject是同步器AQS的内部类，因为Condition的操作需要相关联的锁，所以作为同步器的内部类也比较合理。每个condition对象都包含一个队列，该队列是Condition对象实现等待/通知功能的关键。

**等待队列**

等待队列是一个FIFO的队列，在队列中每个节点都包含一个线程引用，该线程就是在condition对象等待的线程，如果一个线程调用了condition.await()方法，那么该线程将会释放锁，构造成节点加入等待队列并进行等待中。实际上，节点的定义复用了同步器中节点的定义，也就是说，**同步队列和等待队列中节点类型都是同步器静态内部类**

一个Condition包含一个等待队列，Condition拥有首节点firstWaiter和尾节点lastWaiter。当前线程调用Condtion.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列，等待队列基本构造

![image-20201227172155980](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227172155980.png)

Condition拥有首尾节点的引用，而新增节点只需要将原有的尾节点nextWaiter指向它，并且更新尾节点即可。**更新的过程并没有使用CAS保证，原因在于调用await()方法的线程必定获取了锁的线程，也就是该过程是由锁来保证线程安全的。**

在Object的监视器模式上，一个对象拥有一个同步队列和等待队列

**在Lock拥有一个同步队列和多个等待队列。**

![image-20201227173153293](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227173153293.png)

Condition的实现是同步器的内部类，因此每个condition实例都能够访问同步器提供的方法，相当于每个condition都拥有所属同步器的引用。

###  等待

从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，想当于同步队列的首节点（获取锁的节点）移动到Condition的等待队列中。

`ConditionObject的await方法`

```java
public final void await() throws InterruptedException {
        if (Thread.interrupted())
              throw new InterruptedException();
       // 当前线程加入等待队列
       Node node = addConditionWaiter();
      // 释放同步状态，也就是释放锁
      int savedState = fullyRelease(node);
      int interruptMode = 0;
      while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
       if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
           break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
          interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
          unlinkCancelledWaiters();
    if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

调用该方法的线程成功获取了锁的线程，也就是同步队里的首节点，该方法会将当前线程构造节点并加入等到队里中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态

当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException。

![image-20201227174555960](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227174555960.png)

###  通知

调用condition的signal()方法，将会在唤醒等待队列中等待时间最长的节点（首节点），在唤醒之前，将节点移到同步队列。

`ConditionObject的signal方法`

```java
public final void signal() {
      if (!isHeldExclusively())
          throw new IllegalMonitorStateException();
      Node first = firstWaiter;
       if (first != null)
       doSignal(first);
}
```

调用该方法的前置条件是当前线程必须获取锁，可以看到signal()进行了isHeldExclusively()检查，也就是当前线程必须获取了锁的线程，接着获取等待队列的首节点，将其移动到同步队列并使用**LockSupport**唤醒节点中的线程。

![image-20201227175438751](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201227175438751.png)

通过调用同步器的enq(Node node)方法，等待队列中节点线程安全地移动到同步队列。当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程。

被唤醒的线程，将从await()方法中的while循环中退出，进而调用同步器的acquireQueued()方法加入获取同步状态的竞争中。

Condition的signalAll()方法，相当于对等待队列中的每个节点均执行一次signal()方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。