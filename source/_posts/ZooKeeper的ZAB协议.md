---
title: ZooKeeper的ZAB协议
date: 2020-12-11 15:07:32
categories: 服务中心
tags: 分布式
---

##  前言

Zab（Zookeeper Atomic Broadcast）是为ZooKeeper协设计的崩溃恢复原子广播协议，它保证zookeeper集群数据的**一致性和命令的全局有序性。**

##  概念介绍

在介绍zab协议之前首先要知道zookeeper相关的几个概念，才能更好的了解zab协议。

- **集群角色**

  **Leader**: 同一时间集群总有允许有一个Leader,提供对客户端的读写功能，负责将数据同步至各个节点。

  **Follower**: 提供对客户端读功能,写请求则转发给Leader处理,当Leader崩溃失联之后，参与Leader选举

  **Observer**:不参与Leader选举

- **服务状态**

  1. LOOKING：当节点认为群集中没有Leader，服务器会进入LOOKING状态，目的是为了查找或者选举Leader；
  2. FOLLOWING：follower角色；
  3. LEADING：leader角色；
  4. OBSERVING：observer角色；

  Zookeeper是通过自身的状态来区分自己所属的角色，来执行自己应该的任务。

  **ZAB状态**Zookeeper还给ZAB定义的4中状态，反应Zookeeper从选举到对外提供服务的过程中的四个步骤。状态枚举定义：

  ~~~java
      public enum ZabState {
          ELECTION, // 集群进入选举状态，此过程会选出一个节点作为leader角色；
          DISCOVERY,// 连接上leader，响应leader心跳，并且检测leader的角色是否更改，通过此步骤之后选举出的leader才能执行真正职务；
          SYNCHRONIZATION,// 整个集群都确认leader之后，将会把leader的数据同步到各个节点，保证整个集群的数据一致性；
          BROADCAST// 过渡到广播状态，集群开始对外提供服务。
      }
  ~~~

- **Zxid**

  Zxid是Zab协议的一个事务编号,Zxid是一个64位数字,其中低32位是一个简单的单调递增计数器,针对客户每个一个事务请求,计数器+1；而高32位则代表Leader周期年代编号（epoch）。

##  选举

###  选举时机

- **服务器初始化启动**
- **服务器运行期间Leader故障**

###  启动时选举

假设一个 Zookeeper 集群中有5台服务器，id从1到5编号，并且它们都是最新启动的，没有历史数据

![image-20201211154630630](https://jameslin23.gitee.io/2020/12/11/ZooKeeper的ZAB协议/image-20201211154630630.png)

假设服务器依次启动，我们来分析一下选举过程：

**（1）服务器1启动**

发起一次选举，服务器1投自己一票，此时服务器1票数一票，不够半数以上（3票），选举无法完成。

投票结果：服务器1为1票。

服务器1状态保持为`LOOKING`。

**（2）服务器2启动**

发起一次选举，服务器1和2分别投自己一票，此时服务器1发现服务器2的id比自己大，更改选票投给服务器2。

投票结果：服务器1为0票，服务器2为2票。

服务器1，2状态保持`LOOKING`

**（3）服务器3启动**

发起一次选举，服务器1、2、3先投自己一票，然后因为服务器3的id最大，两者更改选票投给为服务器3；

投票结果：服务器1为0票，服务器2为0票，服务器3为3票。此时服务器3的票数已经超过半数（3票），**服务器3当选`Leader`**。

服务器1，2更改状态为`FOLLOWING`，服务器3更改状态为`LEADING`。

**（4）服务器4启动**

发起一次选举，此时服务器1，2，3已经不是LOOKING 状态，不会更改选票信息。交换选票信息结果：服务器3为3票，服务器4为1票。此时服务器4服从多数，更改选票信息为服务器3。

服务器4并更改状态为`FOLLOWING`。

**（5）服务器5启动**

与服务器4一样投票给3，此时服务器3一共5票，服务器5为0票。

服务器5并更改状态为`FOLLOWING`。

**最终的结果**：

服务器3是 `Leader`，状态为 `LEADING`；其余服务器是 `Follower`，状态为 `FOLLOWING`。

###  运行时期的Leader选举

在Zookeeper运行期间 `Leader` 和 `非 Leader` 各司其职，当非Leader服务器宕机或者加入不会影响Leader，但是一旦Leader服务器挂了,那么整个Zookeeper集群将**暂停对外服务**,会触发新一轮的选举。

初始状态下服务器3当选为`Leader`，假设现在服务器3故障宕机了，此时每个服务器上zxid可能都不一样，server1为99，server2为102，server4为100，server5为101

![image-20201211160106980](https://jameslin23.gitee.io/2020/12/11/ZooKeeper的ZAB协议/image-20201211160106980.png)

（1）状态变更。Leader 故障后，余下的`非 Observer` 服务器都会将自己的服务器状态变更为`LOOKING`，然后开始进入`Leader选举过程`。

（2）每个Server会发出投票。

（3）接收来自各个服务器的投票，如果其他服务器的数据比自己的新会改投票。

（4）处理和统计投票，每一轮投票结束后都会统计投票，超过半数即可当选。

（5）改变服务器的状态，宣布当选。

![image-20201211160848185](https://jameslin23.gitee.io/2020/12/11/ZooKeeper的ZAB协议/image-20201211160848185.png)

##  传递

集群在经过leader选举之后还会有连接leader和同步两个步骤，这里就不具体分析这两个步骤的流程了，主要介绍集群对外提供服务如何保证各个节点数据的一致性。

zab在广播状态中保证以下特征

- **可靠传递:** 如果消息m由一台服务器传递，那么它最终将由所有服务器传递。
- **全局有序:** 如果一个消息a在消息b之前被一台服务器交付，那么所有服务器都交付了a和b，并且a先于b。
- **全局有序:** 如果一个消息a在消息b之前被一台服务器交付，那么所有服务器都交付了a和b，并且a先于b。

**有序性**是zab协议必须要保证的一个很重要的属性，因为zookeeper是以类似目录结构的数据结构存储数据的，必须要求命名的有序性。

比如一个命名a创建路径为/test，然后命名b创建路径为/test/123，如果不能保证有序性b命名在a之前，b命令会因为父节点不存在而创建失败。

![image-20201211161516253](https://jameslin23.gitee.io/2020/12/11/ZooKeeper的ZAB协议/image-20201211161516253.png)

​              如上图所示，整个写请求类似一个**二阶段**的提交。

  当收到客户端的写请求的时候会经历以下几个步骤：

1. Leader收到客户端的写请求，生成一个事务（Proposal），其中包含了zxid；
2. Leader开始广播该事务，需要注意的是所有节点的通讯都是由一个FIFO的队列维护的；
3. Follower接受到事务之后，将事务写入本地磁盘，写入成功之后返回Leader一个ACK；
4. Leader收到过半的ACK之后，开始提交本事务，并广播事务提交信息
5. 从节点开始提交本事务。

有以上流程可知，zookeeper通过二阶段提交来保证集群中数据的一致性，因为只需要收到过半的ACK就可以提交事务，所以zookeeper的数据**并不是强一致性。**

**zab协议的有序性保证是通过几个方面来体现的，第一是，服务之前用TCP协议进行通讯，保证在网络传输中的有序性；第二，节点之前都维护了一个FIFO的队列，保证全局有序性；第三，通过全局递增的zxid保证因果有序性。**



###  状态流转

前面介绍了zookeeper服务状态有四种，ZAB状态也有四种。这里就简单介绍一个他们之间的状态流转，更能加深对zab协议在zookeeper工作流程中的作用。

![image-20201211162550315](https://jameslin23.gitee.io/2020/12/11/ZooKeeper的ZAB协议/image-20201211162550315.png)

1. 服务在启动或者和leader失联之后服务状态转为LOOKING
2. 如果leader不存在选举leader，如果存在直接连接leader，此时zab协议状态为ELECTION
3. 如果有超过半数的投票选择同一台server，则leader选举结束，被选举为leader的server服务状态为LEADING，其他server服务状态为FOLLOWING/OBSERVING
4. 所有server连接上leader，此时zab协议状态为DISCOVERY
5. leader同步数据给learner，使各个从节点数据和leader保持一致，此时zab协议状态为SYNCHRONIZATION
6. 同步超过一半的server之后，集群对外提供服务，此时zab状态为BROADCAST



