---
title: RocketMQ集群模式
date: 2020-12-04 18:52:53
categories: 消息队列
tags: MQ
---

#  Master-Slave主从模式

> 生产者发数据只会发到Master上，然后slave 从Master pull数据同步下来，
> 跟master保存一份一模一样的数据。
>
> 消费者在系统获取消息时候会先发送请求到Master Broker上去,请求获取一批消息，
> 此时Master Broker是会返回一批消息给消费者系统的。然后Master Broker在返回消息给消费者系统的时候，
>
> 会根据当时Master Broker的负载情况和Slave Broker的同步情况，向消费者系统建议下一次拉取消息的时候是从Master Broker拉取还是从Slave Broker拉取。

> 在RocketMQ 4.5版本之前，都是用Slave Broker同步数据，尽量保证数据不丢失，但是一旦Master故障了，Slave是没法自动切换成Master的，在这种情况下，如果Master Broker宕机了，这时就得手动做一些运维操作，把Slave Broker重新修改一些配置，重启机器给调整为Master Broker，这是有点麻烦的，而且会导致中间一段时间不可用。Master-Slave模式不是彻底的高可用模式，他没法实现自动把Slave切换为Master

# 基于DLedger高可用自动切换

##  什么是DLedger

> DLedger有一个CommitLog机制，你把数据交给他，他会写入CommitLog磁盘文件里去，这些他做的第一件事。如下图可见，基于DLedger技术来实现Broker高可用框架，实际就是用DLedger先替换原来Broker自己来管理的CommitLog，由DLedger来管理CommitLog

![image-20201203081710737](https://jameslin23.gitee.io/2020/12/01/RocketMQ/image-20201203081710737.png)

##  DLedger基于Raft协议选举Leader Broker

> 简单来说,三台Broker机器启动时，他们都会投票自己作为Leader,然后把这个投票发送给其它Broker。
>
> 举个例子,Broker01是投票给自己的，Broker02是投票给自己的，Broker03是投票给自己的，他们都把自己的投票发送给了别
> 人。
>
> -----
>
> 第一轮选举中，Broker01会收到别人的投票，他发现自己是投票给自己，但是Broker02投票给Broker02自己，Broker03投票给Broker03自己，似乎每个人都很自私，都在投票给自己，所以第一轮选举是失败的
>
> -----------------------------
>
> 每个人会进入一个随机时间的休眠，比如说Broker01休眠3秒，Broker02休眠5秒，Broker03休眠4秒。
>
> --------
>
> Broker01必然是先苏醒过来的，他苏醒过来之后，直接会继续尝试投票给自己，并且发送自己的选票给别人。接着Broker03休眠4秒后苏醒过来，他发现Broker01已经发送来了一个选票是投给Broker01自己的，此时他自己因为没投票，所以会尊重别人的选择，就直接把票投给Broker01了，同时把自己的投票发送给别人。接着Broker02苏醒了，他收到了Broker01投票给Broker01自己，收到了Broker03也投票给了Broker01，那么他此时自己是没投票
> 的，直接就会尊重别人的选择，直接就投票给Broker01，并且把自己的投票发送给别人
>
> -------
>
> 此时所有人都会收到三张投票，都是投给Broker01的，那么Broker01就会当选为Leader。

**Raft协议中选举leader算法的简单描述，简单来说，他确保有人可以成为Leader的核心机制就是一轮选举不出来Leader的话，**
**就让大家随机休眠一下，先苏醒过来的人会投票给自己，其他人苏醒过后发现自己收到选票了，就会直接投票给那个人。**

![image-20201203082435078](https://jameslin23.gitee.io/2020/12/01/RocketMQ/image-20201203082435078.png)

##  DLedger基于Raft协议进行副本同步

**数据同步会分为两个阶段，一个是uncommitted阶段，一个是commited阶段**

> 首先Leader Broker上的DLedger收到一个条数据之后，会标记为uncommitted状态，然后通过自己的DLedgerServer组件把这个
>
> uncommitted数据发送给Follower Broker的DLedgerServer。

![image-20201203082959966](https://jameslin23.gitee.io/2020/12/01/RocketMQ/image-20201203082959966.png)

> 接着Follower Broker的DLedgerServer收到uncommitted消息之后，必须返回一个ack给Leader Broker的DLedgerServer,然后如果Leader Broker收到超过半数的Follower Broker返回的ack之后，就会将消息标记为committed状态。
>
> 然后Leader Broker上的DLedgerServer就会发送commited消息给Follower Broker机器的DLedgerServer,让他们也把消息标记commited状态

**这个就是基于Raft协议实现的两阶段完成的数据同步机制。**

##  如何使用心跳维护leader地位

**当选节点之后，首选要维护自己的leader地位，他要告诉集群其它节点，我是集群中leader，你们要成为我的follower，负责同步我的数据，并且保证只要我还活着，你们就不要妄想重新进行选举**

1. 每隔几秒钟leader节点会向所有follower节点发送心跳请求；
2. follower收到心跳请求之后，更新本地倒计时时间，同时给leader节点一个确认回复
3. leader节点收到过半数follower节点回复，则说明自己还是leader
4. 如果没有收到过半数follower节点回复，则会变更为candidate状态，重新触发选举；同样的，如果follower节点一直没有收到leader节点的心跳请求,follower节点也会变更为candidate状态，触发leader选举。

#  实践部署

##  下载MQ，要求版本4.5以上

1. https://github.com/apache/rocketmq/releases 这里下载需要的RocketMQ版本
2. unzip rocketmq-all-4.7.0-bin-release.zip

##  修改配置文件

服务器说明：(生产中应该将 NameServer 部署到其他服务器中，在这为了方便，与Broker部署在一起)

| **服务器** |    **ip**     |       **安装的服务**        |
| :--------: | :-----------: | :-------------------------: |
| 服务器1-主 | 192.168.10.8  | DLedger，Broker，NameServer |
| 服务器2-从 | 192.168.10.10 | DLedger，Broker，NameServer |
| 服务器3-从 | 192.168.10.3  | DLedger，Broker，NameServer |

**服务器1配置-Master**

```properties
#  ## 集群名
brokerClusterName = RaftCluster
## broker组名，同一个RaftClusterGroup内，brokerName名要一样
brokerName=RaftNode00
## 监听的端口
listenPort=30911
# 你设置的NameServer地址和端口
namesrvAddr=192.168.10.8:9876;192.168.10.10:9876;192.168.10.3:9876
# 数据存储路径
storePathRootDir=/tmp/rmqstore/node1
storePathCommitLog=/tmp/rmqstore/node1/commitlog
# 是否启动 DLedger
enableDLegerCommitLog=true
# DLedger Raft Group的名字，建议和 brokerName 保持一致
dLegerGroup=RaftNode00
# DLedger Group 内各节点的端口信息，同一个 Group 内的各个节点配置必须要保证一致
dLegerPeers=n1-192.168.10.8:40911;n2-192.168.10.10:40912;n3-192.168.10.3:40913
#节点 id, 必须属于 dLegerPeers 中的一个；同 Group 内各个节点要唯一
dLegerSelfId=n1
# 发送线程个数，建议配置成 Cpu 核数
sendMessageThreadPoolNums=16
```

**服务器2配置-Slave**

~~~properties
brokerClusterName = RaftCluster
brokerName=RaftNode00
listenPort=30922
namesrvAddr=192.168.10.8:9876;192.168.10.3:9876;192.168.10.10:9876
storePathRootDir=/tmp/rmqstore/node2
storePathCommitLog=/tmp/rmqstore/node2/commitlog
enableDLegerCommitLog=true
dLegerGroup=RaftNode00
dLegerPeers=n1-192.168.10.8:40911;n2-192.168.10.10:40912;n3-192.168.10.3:40913
## must be unique
dLegerSelfId=n2
sendMessageThreadPoolNums=16
~~~

**服务器3配置-Slave**

~~~properties
brokerClusterName = RaftCluster
brokerName=RaftNode00
listenPort=30933
namesrvAddr=192.168.10.8:9876;192.168.10.3:9876;192.168.10.10:9876
storePathRootDir=/tmp/rmqstore/node03
storePathCommitLog=/tmp/rmqstore/node03/commitlog
enableDLegerCommitLog=true
dLegerGroup=RaftNode00
dLegerPeers=n1-192.168.10.8:40911;n2-192.168.10.10:40912;n3-192.168.10.3:40913
## must be unique
dLegerSelfId=n3
sendMessageThreadPoolNums=16
~~~

##  启动集群

1. **在服务器1 执行**

   > 先启动mqnamesrv服务 nohup sh mqnamesrv >/dev/null 2>log &
   >
   > 再启动broker服务 nohup sh mqbroker -c ../conf/dledger/broker-node1.conf >/dev/null 2>log &

2. **在服务器2 执行**

   > 先启动mqnamesrv服务 nohup sh mqnamesrv >/dev/null 2>log &
   >
   > 再启动broker服务 nohup sh mqbroker -c ../conf/dledger/broker-node2.conf >/dev/null 2>log &

3. **在服务器3 执行**

   > 先启动mqnamesrv服务 nohup sh mqnamesrv >/dev/null 2>log &
   >
   > 再启动broker服务 nohup sh mqbroker -c ../conf/dledger/broker-node3.conf >/dev/null 2>log &

 **内存不足，修改runbroker.sh，runserver.sh 参数**

~~~bash
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=1g
~~~

**连接超时，查看防火墙状态**

查看防火墙服务状态 `systemctl status firewalld`

将防火墙关闭 `systemctl stop firewalld`

**查看集群状态**

**sh mqadmin clusterList -n 127.0.0.1:9876**

![image-20201207084050039](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207084050039.png)

**使用控制台查看**

![image-20201207084942295](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207084942295.png)

**测试生产者-消费者**

![image-20201207085123126](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207085123126.png)

![image-20201207085144645](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207085144645.png)

**测试集群高可用，关闭192.168.10.8节点，集群重新竞选master**

![image-20201207085319420](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207085319420.png)

**发送消息**

![image-20201207085522551](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207085522551.png)

**重启192.168.10.8节点,变成slave**

![image-20201207085809732](https://jameslin23.gitee.io/2020/12/04/RocketMQ集群模式/image-20201207085809732.png)

#  配置参数说明

| 参数名                    | 默认值                                                       | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| listenPort                | 10911                                                        | 接受客户端连接的监听端口                                     |
| namesrvAddr               | null                                                         | nameServer 地址                                              |
| brokerIP1                 | 网卡的 InetAddress                                           | 当前 broker 监听的 IP                                        |
| brokerIP2                 | 跟 brokerIP1 一样                                            | 存在主从 broker 时，如果在 broker 主节点上配置了 brokerIP2 属性，broker 从节点会连接主节点配置的 brokerIP2 进行同步 |
| brokerName                | null                                                         | broker 的名称                                                |
| brokerClusterName         | DefaultCluster                                               | 本 broker 所属的 Cluser 名称                                 |
| brokerId                  | 0                                                            | broker id, 0 表示 master, 其他的正整数表示 slave             |
| storePathCommitLog        | $HOME/store/commitlog/                                       | 存储 commit log 的路径                                       |
| storePathConsumerQueue    | $HOME/store/consumequeue/                                    | 存储 consume queue 的路径                                    |
| mappedFileSizeCommitLog   | 1024 * 1024 * 1024(1G)                                       | commit log 的映射文件大小                                    |
| deleteWhen                | 04                                                           | 在每天的什么时间删除已经超过文件保留时间的 commit log        |
| fileReservedTime          | 72                                                           | 以小时计算的文件保留时间                                     |
| brokerRole                | ASYNC_MASTER                                                 | SYNC_MASTER/ASYNC_MASTER/SLAVE                               |
| flushDiskType             | ASYNC_FLUSH                                                  | SYNC_FLUSH/ASYNC_FLUSH SYNC_FLUSH 模式下的 broker 保证在收到确认生产者之前将消息刷盘。ASYNC_FLUSH 模式下的 broker 则利用刷盘一组消息的模式，可以取得更好的性能。 |
| enableDLegerCommitLog     | 是否启动 DLedger                                             | true                                                         |
| dLegerGroup               | DLedger Raft Group的名字，建议和 brokerName 保持一致         | RaftNode00                                                   |
| dLegerPeers               | DLedger Group 内各节点的端口信息，同一个 Group 内的各个节点配置必须要保证一致 | n0-127.0.0.1:40911;n1-127.0.0.1:40912;n2-127.0.0.1:40913     |
| dLegerSelfId              | 节点 id, 必须属于 dLegerPeers 中的一个；同 Group 内各个节点要唯一 | n0                                                           |
| sendMessageThreadPoolNums | 发送线程个数，建议配置成 Cpu 核数                            | 16                                                           |