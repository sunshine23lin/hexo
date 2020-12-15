---
title: redis分布式锁
date: 2020-12-07 20:01:38
categories: Nosql
tags: 缓存
---

#  分布式锁

##  什么是分布式锁?

分布式锁就是控制分布式系统或不同系统之间共同访问共享资源的一种锁实现,如果不同的系统或同一个系统的不同主机之间共享了某个资源时，往往需要互斥来防止彼此干扰来保证一致性。

## 分布式锁需要具备哪些条件

1. **互斥性**:在任意一个时刻,只有一个客户端持有锁
2. **无死锁**:即便持有锁的客户端崩溃或者其他意外事件,锁仍然可以被获取。
3. **容错**:只要大部分redis节点都活着,客户端就可以获取锁和释放锁

##  分布式锁实现有哪些？

- **数据库**
- **Memcached（add命令）**
- **Redis（setnx命令）**
- **Zookeeper（临时节点）**

#  分布式锁实现

 ##  SET key value NX

假设有两个客户端A和B，A获取到分布式的锁。A执行了一会，突然A所在的服务器断电了（或者其他什么的），也就是客户端A挂了。这时出现一个问题，这个锁一直存在，且不会被释放，其他客户端永远获取不到锁。如下示意图

![image-20201207205319341](https://jameslin23.gitee.io/2020/12/07/redis分布式锁/image-20201207205319341.png)

##  SET lockKey value NX EX 30

**注: 要保证设置过期时间和设置锁具有原子性**

此时又出现一个问题,过程如下

1. 客户端A获取锁成功,过期时间30秒
2. 客户端A在某个程序上阻塞了50秒
3. 30秒时间到了，锁自动释放了
4. 客户端B获取对应同一资源的锁
5. 客户端A从阻塞恢复过来,释放掉了客户端B持有的锁

![image-20201207210005131](https://jameslin23.gitee.io/2020/12/07/redis分布式锁/image-20201207210005131.png)

这时会有两个问题

1. 过期时间如何保证大于业务执行时间?
2. 如何保证锁不会被误删除?

**先来解决如何保证锁不会被误删除这个问题。 这个问题可以通过设置value为当前客户端生成的一个随机字符串，且保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。**

##  lua脚本释放锁具备原子性

~~~java
  public void unlock() {
        // 使用lua脚本进行原子删除操作
        String checkAndDelScript = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                                    "return redis.call('del', KEYS[1]) " +
                                    "else " +
                                    "return 0 " +
                                    "end";
        jedis.eval(checkAndDelScript, 1, lockKey, lockValue);
    }
~~~



#  Redisson实现分布式锁

##  Redisson原理

![img](https://jameslin23.gitee.io/2020/12/07/redis分布式锁/1090617-20190618183025891-1248337684.jpg)

1. **加锁机制**

   线程去获取锁,获取成功:执行lua脚本,保存数据到redis数据库

   线程去获取锁,获取失败:一直通过while循环尝试获取锁,获取成功后，执行lua脚本，保存数据到redis数据库

2. **watch dog自动延期机制**

   它的作用就是 线程1 业务还没有执行完，锁时间就过了，线程1 还想持有锁的话，就会启动一个watch dog后台线程，不断的延长锁key的生存时间。

3. **LUA脚本**

   主要是如果你的业务逻辑复杂的话，通过封装在lua脚本中发送给redis，而且redis是单线程的，这样就保证这段复杂业务逻辑执行的**原子性**。

4. **可重入锁**

   Hash数据类型的key值包含了当前线程信息。

   下面是redis存储的数据

   ![img](https://jameslin23.gitee.io/2020/12/07/redis分布式锁/1090617-20190618183037704-975536201.png)

   这里表面数据类型是Hash类型,Hash类型相当于我们java的 `<key,<key1,value>>` 类型,这里key是指 'redisson'

   它的有效期还有9秒，我们再来看里们的key1值为`078e44a3-5f95-4e24-b6aa-80684655a15a:45`它的组成是:

   guid + 当前线程的ID。后面的value是就和可重入加锁有关。

   **举图说明**

   ![img](https://jameslin23.gitee.io/2020/12/07/redis分布式锁/1090617-20190618183046827-1994396879.png)

   上面这图的意思就是可重入锁的机制，它最大的优点就是相同线程不需要在等待锁，而是可以直接进行相应操作。

   

## Redis分布式锁缺陷

**Redis分布式锁会有个缺陷，就是在Redis哨兵模式下:**

`客户端1` 对某个`master节点`写入了redisson锁，此时会异步复制给对应的 slave节点。但是这个过程中一旦发生 master节点宕机，主备切换，slave节点从变为了 master节点。

这时`客户端2` 来尝试加锁的时候，在新的master节点上也能加锁，此时就会导致多个客户端对同一个分布式锁完成了加锁。

这时系统在业务语义上一定会出现问题，**导致各种脏数据的产生**。

`缺陷`在哨兵模式或者主从模式下，如果 master实例宕机的时候，可能导致多个客户端同时完成加锁。

##  RLock接口

**Redisson分布式锁是基于RLock接口,RedissonLock实现RLock接口**。

~~~java
public interface RLock extends Lock, RExpirable, RLockAsync
~~~

很明显RLock是继承Lock锁,所以他有Lock锁的所有特征,比如lock,unlock,trylock等特征，同时它很多新特性:强制锁释放,带有效期的锁。

##  RLock锁API

~~~java
public interface RLock {
    //----------------------Lock接口方法-----------------------

    /**
     * 加锁 锁的有效期默认30秒
     */
    void lock();
    /**
     * tryLock()方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false .
     */
    boolean tryLock();
    /**
     * tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，
     * 在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。
     *
     * @param time 等待时间
     * @param unit 时间单位 小时、分、秒、毫秒等
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    /**
     * 解锁
     */
    void unlock();
    /**
     * 中断锁 表示该锁可以被中断 假如A和B同时调这个方法，A获取锁，B为获取锁，那么B线程可以通过
     * Thread.currentThread().interrupt(); 方法真正中断该线程
     */
    void lockInterruptibly();

    //----------------------RLock接口方法-----------------------
    /**
     * 加锁 上面是默认30秒这里可以手动设置锁的有效时间
     *
     * @param leaseTime 锁有效时间
     * @param unit      时间单位 小时、分、秒、毫秒等
     */
    void lock(long leaseTime, TimeUnit unit);
    /**
     * 这里比上面多一个参数，多添加一个锁的有效时间
     *
     * @param waitTime  等待时间
     * @param leaseTime 锁有效时间
     * @param unit      时间单位 小时、分、秒、毫秒等
     */
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
    /**
     * 检验该锁是否被线程使用，如果被使用返回True
     */
    boolean isLocked();
    /**
     * 检查当前线程是否获得此锁（这个和上面的区别就是该方法可以判断是否当前线程获得此锁，而不是此锁是否被线程占有）
     * 这个比上面那个实用
     */
    boolean isHeldByCurrentThread();
    /**
     * 中断锁 和上面中断锁差不多，只是这里如果获得锁成功,添加锁的有效时间
     * @param leaseTime  锁有效时间
     * @param unit       时间单位 小时、分、秒、毫秒等
     */
    void lockInterruptibly(long leaseTime, TimeUnit unit);  
}
  
~~~

##  RedissonLock实现类

~~~java
public class RedissonLock extends RedissonExpirable implements RLock
~~~

###  void lock()方法

~~~java
    @Override
    public void lock() {
        try {
            lockInterruptibly();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
~~~

发现lock锁里面进去其实用的是lockInterruptibly(中断锁,表示可以被中断),而且捕获异常后用Thread.currentThread().interupt()来真正中断当前线程,其实它们是搭配一起使用的

~~~java
   /**
     * 1、带上默认值调另一个中断锁方法
     */
    @Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly(-1, null);
    }
    /**
     * 2、另一个中断锁的方法
     */
    void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException 
    /**
     * 3、这里已经设置了锁的有效时间默认为30秒  （commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout()=30）
     */
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    /**
     * 4、最后通过lua脚本访问Redis,保证操作的原子性
     */
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
~~~



###  tryLock(long waitTime,long leaseTime,TimeUnit unit)

~~~java
 @Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long time = unit.toMillis(waitTime);
        long current = System.currentTimeMillis();
        long threadId = Thread.currentThread().getId();
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        //1、 获取锁同时获取成功的情况下，和lock(...)方法是一样的 直接返回True，获取锁False再往下走
        if (ttl == null) {
            return true;
        }
        //2、如果超过了尝试获取锁的等待时间,当然返回false 了。
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(threadId);
            return false;
        }

        // 3、订阅监听redis消息，并且创建RedissonLockEntry，其中RedissonLockEntry中比较关键的是一个 Semaphore属性对象,用来控制本地的锁请求的信号量同步，返回的是netty框架的Future实现。
        final RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
        //  阻塞等待subscribe的future的结果对象，如果subscribe方法调用超过了time，说明已经超过了客户端设置的最大wait time，则直接返回false，取消订阅，不再继续申请锁了。
        //  只有await返回true，才进入循环尝试获取锁
        if (!await(subscribeFuture, time, TimeUnit.MILLISECONDS)) {
            if (!subscribeFuture.cancel(false)) {
                subscribeFuture.addListener(new FutureListener<RedissonLockEntry>() {
                    @Override
                    public void operationComplete(Future<RedissonLockEntry> future) throws Exception {
                        if (subscribeFuture.isSuccess()) {
                            unsubscribe(subscribeFuture, threadId);
                        }
                    }
                });
            }
            acquireFailed(threadId);
            return false;
        }

       //4、如果没有超过尝试获取锁的等待时间，那么通过While一直获取锁。最终只会有两种结果
        //1)、在等待时间内获取锁成功 返回true。2）等待时间结束了还没有获取到锁那么返回false。
        while (true) {
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(leaseTime, unit, threadId);
            // 获取锁成功
            if (ttl == null) {
                return true;
            }
           //   获取锁失败
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }
        }
    }
~~~

tryLock一般用于特定满足需求的场合,但不建议作为一般需求的分布式锁，一般分布式锁建议用void lock(long leaseTime,TimeUnit unit)。因此性能上考虑，在高并发情况下后者效率是前者的好几倍。

###  unlock()

~~~java
@Override
    public void unlock() {
        // 1.通过 Lua 脚本执行 Redis 命令释放锁
        Boolean opStatus = commandExecutor.evalWrite(getName(), LongCodec.INSTANCE,
                RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; " +
                        "end;" +
                        "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                        "return nil;" +
                        "end; " +
                        "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                        "if (counter > 0) then " +
                        "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                        "return 0; " +
                        "else " +
                        "redis.call('del', KEYS[1]); " +
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; "+
                        "end; " +
                        "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()),
                LockPubSub.unlockMessage, internalLockLeaseTime,
                getLockName(Thread.currentThread().getId()));
        // 2.非锁的持有者释放锁时抛出异常
        if (opStatus == null) {
            throw new IllegalMonitorStateException(
                    "attempt to unlock lock, not locked by current thread by node id: "
                            + id + " thread-id: " + Thread.currentThread().getId());
        }
        // 3.释放锁后取消刷新锁失效时间的调度任务
        if (opStatus) {
            cancelExpirationRenewal();
        }
    }
~~~

使用EVAL命令执行Lua脚本来释放锁:

1. key 不存在,说明锁已经释放,直接执行`public`命令发布释放锁消息并返回1
2. key存在,但field在 Hash 中不存在，说明自己不是锁持有者，无权释放锁，返回 `nil`。
3. 因为锁可重入,所以释放锁时不能把所有已获取的锁全部释放掉,一次只能释放一把锁,因此执行`hincrby`对锁的值减1。
4. 释放一把锁后，如果还有剩余的锁,则刷新锁的失效时间并返回`0`；如果刚才释放的已经是最后一把锁，则执行 `del` 命令删除锁的 key，并发布锁释放消息，返回 `1`。

`注意`这里有个实际开发过程中，容易出现很容易出现上面第二步异常，非锁的持有者释放锁时抛出异常。比如下面这种情况

~~~java
  //设置锁1秒过去
        redissonLock.lock("redisson", 1);
        /**
         * 业务逻辑需要咨询2秒
         */
        redissonLock.release("redisson");
      /**
       * 线程1 进来获得锁后，线程一切正常并没有宕机，但它的业务逻辑需要执行2秒，这就会有个问题，在 线程1 执行1秒后，这个锁就自动过期了，
       * 那么这个时候 线程2 进来了。在线程1去解锁就会抛上面这个异常（因为解锁和当前锁已经不是同一线程了）
       */
~~~



