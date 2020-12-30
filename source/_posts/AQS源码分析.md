---
title: AQS源码分析
date: 2020-12-28 16:26:22
categories: 并发编程
tags: AQS
---

##  AQS简单介绍

AQS定义两种资源：Exclusive(独占，只有一个线程能执行，如果ReentranLock) 和 Share(共享，多个线程可同时执行，如Semaphore/CountDownLatch)。

不同的自定义同步器争用共享的方式也不同。自定义同步器实现时只需要实现共享资源state的获取和释放即可。至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了，自定义同步器实现主要实现以下几种方法：

- **isHeldExclusively()**：该线程是否正在独占资源。只有用到condition才需要去实现它。
- **tryAcquire(int)**：独占方式，尝试获取资源，成功则返回true,失败返回false。
- **tryRelease(int)**：独占方式，尝试释放资源，成功则返回true,失败返回false。
- **tryAcquireShared(int)**：共享方式，尝试获取资源，负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源
- **tryReleaseShared(int)**：共享方式，尝试释放资源，如果释放后允许唤醒后续等待节点返回true,否则返回false

**以ReentrantLock为例**

state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占锁并将state+1。此后，其它线程再tryAcquire()时就会失败，直到A线程unlock()到state=0(即释放锁)为止，其它线程才有机会获取该锁。释放之前，A线程自己可以重复获取此锁（state会累积），这是锁重入概念，但是主要，获取多少次就要释放多少次，这样才能保证state是能回到零。

**CountDownLatch为例**

任务分为N个子线程去执行，state也初始化为N（N和线程数一直）。这N个线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS-1,等到所有子线程都执行后（既state=0）,会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

##  源码分析

###  Node节点

Node节点是对每一个等待获取资源的线程封装，其包含了需要同步线程本身及其等待状态，如是否被阻塞、是否等待唤醒、是否被取消等。变量waitStatus则表示当前Node节点的等待状态，共有5种取值

- **CANCELLED(1)**：表示当前节点已经取消调度。当timeout或者被中断（响应中断的情况下），会触发变更此状态，进入该状态后的节点将不会再变化。
- **SIGNAL(-1)**：表示**后继节点**在等待前节点唤醒。后续节点入队时，会将前继节点的状态更新为SIGNAL。
- **CONDITION(-2)**：表示节点等待在Condition上，当其他线程调用condition的signal()方法后，CODIOTION状态节点从等待队列转移同步队列
- **PROPAGATE(-3)**：共享模式下，前继节点不仅会唤醒后继节点，同时也可能唤醒后继的后继节点。
- **0**：新节点入队时默认状态。

负值表示节点处于有效等待状态，而正值表示结点已被取消。

###  独占方式

#### 获取锁

**acquire(int)**

此方法是独占模式下线程共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取资源为止。

```java
public final void acquire(int arg){
    if(!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE),arg))
        selfInterrupt();
}
```

方法如下：

- tryAcquire()尝试直接获取资源，如果成功则直接返回（这里体现了非公平，每个线程获取锁时会直接抢占一次）
- addWaiter()将该线程加入等待队列的尾部，并标记为独占模式
- acquireQueued()使得线程阻塞在等待队列中获取资源，一直获取到资源才返回。如果整个过程中被中断，则返回true,否则返回false
- 如果线程在等待队列中被中断过，他是不响应的，只有获取资源后才进行自我中断selfInterrupt()，将中断补上

**addWaiter(Node)**

此方法用于将当前线程加入等待队列的队尾，并返回当前线程所在的节点。

```Java
 //注意：该入队方法的返回值就是新创建的节点
    private Node addWaiter(Node mode) {
        //基于当前线程，节点类型（Node.EXCLUSIVE）创建新的节点
        //由于这里是独占模式，因此节点类型就是Node.EXCLUSIVE
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        //这里为了提搞性能，首先执行一次快速入队操作，即直接尝试将新节点加入队尾
        if (pred != null) {
            node.prev = pred;
            //这里根据CAS的逻辑，即使并发操作也只能有一个线程成功并返回，其余的都要执行后面的入队操作。即enq()方法
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

```

**enq(node)**

```java
 //完整的入队操作
    private Node enq(final Node node) {
        //自旋，直到成功加入队列
        for (;;) {
            Node t = tail;
            //如果队列还没有初始化，则进行初始化，即创建一个空的头节点
            if (t == null) { 
               // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else { // //正常流程，放入队尾
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    //该循环体唯一退出的操作，就是入队成功（否则就要无限重试）
                    return t;
                }
            }
        }
    }
```

**acquireQueued(Node,int)**

通过tryAcquire()和addWaiter(),该线程获取资源失败，已经被放入队列尾部了。那线程下一步操作该干什么呢？

进入等待状态休息，直到其它线程彻底释放资源后唤醒自己。acquireQueued类似跟医院排队拿号，在等待队列中排队拿号，中间没有其它事可以做，可以休息，直到拿到号在返回。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//标记是否成功拿到资源
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过
        //又是一个“自旋”！
        for (;;) {
            final Node p = node.predecessor();//拿到前驱
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                failed = false; // 成功获取资源
                return interrupted;//返回等待过程中是否被中断过
            }

            // 其它就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
        }
    } finally {
        if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}
```

**shouldParkAfterFailedAcquire（Node,Node）**

此方法主要用于检查状态，看看自己是否可以被挂起

```Java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;//拿到前驱的状态
    if (ws == Node.SIGNAL)
        //如果已经告诉前驱拿完号后通知自己一下，那就可以被挂起
        return true;
    if (ws > 0) {
        /*
         * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
         * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

整个流程，如果前驱节点状态不是SIGNAL，那么自己不能安心去休息（挂起），需要遍历前驱节点，找到正常可以唤醒自己的前驱节点。

**parkAndCheckInterrupt()**

如果线程找到可以唤醒自己前驱节点，也就是前驱节点的状态时SIGNAL,就可以挂起

```java
 private final boolean parkAndCheckInterrupt() {
     LockSupport.park(this);//调用park()使线程进入waiting状态
     return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
}
```

park()会让自己线程进入waiting状态，在此状态下，有两种途径可以唤醒该线程

- **unpark()**
- **interrupt()**

**cancelAcquire(node)**

```java
//传入的方法参数是当前获取锁资源失败的节点
private void cancelAcquire(Node node) {
        // 如果节点不存在则直接忽略
        if (node == null)
            return;
        
        node.thread = null;

        // 跳过所有已经取消的前置节点，跟上面的那段跳转逻辑类似
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        //这个是前置节点的后继节点，由于上面可能的跳节点的操作，所以这里可不一定就是当前节点，仔细想一下。^_^
        Node predNext = pred.next;

        //把当前节点waitStatus置为取消，这样别的节点在处理时就会跳过该节点
        node.waitStatus = Node.CANCELLED;
        //如果当前是尾节点，则直接删除，即出队
        //注：这里不用关心CAS失败，因为即使并发导致失败，该节点也已经被成功删除
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    //这里的判断逻辑很绕，具体就是如果当前节点的前置节点不是头节点且它后面的节点等待它唤醒（waitStatus小于0），
                    //再加上如果当前节点的后继节点没有被取消就把前置节点跟后置节点进行连接，相当于删除了当前节点
                    compareAndSetNext(pred, predNext, next);
            } else {
                //进入这里，要么当前节点的前置节点是头结点，要么前置节点的waitStatus是PROPAGATE，直接唤醒当前节点的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```



**总结：首先会先调用tryacquire()方法尝试获取锁，如果获取得到就直接返回，通过addwaiter()方法创建一个node节点，通过CAS加入队列尾部，会一直尝试，直到成功为止。加入成功的node节点就会执行acquireQueued方法，这个办法主要是判断线程前驱节点是不是头节点，并尝试去获取资源，如果符合条件就退出自旋，并返回。如果获取不到就会进入判断是否可以挂起线程，如果可以就执行lockSupport.park挂起该线程， 等待其它线程唤醒。**

####  释放锁

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            // 0表示没有其它被唤醒的节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

```java
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            //把标记为设置为0，表示唤醒操作已经开始进行，提高并发环境下性能
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        //如果当前节点的后继节点为null，或者已经被取消
        if (s == null || s.waitStatus > 0) {
            s = null;
            //注意这个循环没有break，也就是说它是从后往前找，一直找到离当前节点最近的一个等待唤醒的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //执行唤醒操作
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

总结：先将状态设置为0，从后往前找，找到离当前节点最近的一个等待唤醒的节点，进行unpark。



###  共享方式

####  获取锁

**acquireShared(int)**

此方法是共享模式下线程获取共享资源的顶层入口。它会获取指**定量**的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止。

```java
 public final void acquireShared(int arg) {
     if (tryAcquireShared(arg) < 0)
         doAcquireShared(arg);
 }
```

这里tryAcquireShared()依然需要自定义同步器去实现。但是AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里acquireShared()的流程就是：

- tryAcquireShared()尝试获取资源，成功则直接返回
- 失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回

**doAcquireShared(int)**

此方法用于当前线程加入等待队列休息，直到其它线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);//加入队列尾部
    boolean failed = true;//是否成功标志
    try {
        boolean interrupted = false;//等待过程中是否被中断过的标志
        for (;;) {
            final Node p = node.predecessor();//前驱
            if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
                int r = tryAcquireShared(arg);//尝试获取资源
                if (r >= 0) {//成功
                    setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
                    p.next = null; // help GC
                    if (interrupted)//如果等待过程中被打断过，此时将中断补上。
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }

            //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**setHeadAndPropagate(node, r)**

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);//head指向自己
     //如果还有剩余量，继续唤醒下一个邻居线程
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

**总结：首先tryAcquireShared去获取锁，如果返回值小于0说明没有剩余资源，进入同步队列。拿到队列的第二个节点去尝试获取资源。成功就将自己设置头节点，判断是否有资源继续唤醒后继节点**

####  释放锁

**releaseShared()**

此方法是共享模式下线程释放共享资源的顶层入口，它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其它线程来获取资源

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {//尝试释放资源
        doReleaseShared();//唤醒后继结点
        return true;
    }
    return false;
}
```

**doReleaseShared()**

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);//唤醒后继
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head)// head发生变化
            break;
    }
}
```

**总结：释放资源，唤醒后继**