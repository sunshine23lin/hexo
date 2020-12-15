---
title: RocketMQ单机模式
date: 2020-12-04 18:20:49
categories: 消息队列
tags: MQ
---

# 按照步骤

##  官网下载

http://rocketmq.apache.org/

rocketmq-all-4.6.1-bin-release.zip

**unzip rocketmq-all-4.6.1-bin-release.zip**

##  修改配置文件

**在MQ目录下 conf/2m-noslave 创建新的配置文件**

~~~properties
#broker 集群名称
brokerClusterName = cluster-bus
#broker 名称
brokerName = broker-4
#broker id
brokerId = 0
#指定nameserver 的地址端口
namesrvAddr=192.168.2.4:9876
#数据清除时间
deleteWhen=04
#文件保存时间
fileReservedTime=48
#磁盘阀值
diskMaxUsedSpaceRatio=95
#broker 角色
brokerRole=ASYNC_MASTER
#刷盘方式
flushDiskType=ASYNC_FLUSH
#CommitLog 存储路径
storePathCommitLog=/data/mq/store/commitlog
#数据存储根路径
storePathRootDir=/data/mq/store
#storePathConsumerQueue 消息队列存储路径
#storePathConsumerQueue=/home/apps/data/rocketmq/store/consumerqueue/
#storePathIndex 消息索引存储队列
#storePathIndex=/home/apps/data/rocketmq/store/index/ 
#单次从内存最大消费字节数
#maxTransferBytesOnMessageInMemory=2621440
#单次从内存最大消费信息条数   
maxTransferCountOnMessageInMemory=20000
#单次从磁盘获取信息的最大条数
maxTransferCountOnMessageInDisk=1000
#单次从磁盘获取信息的最大字节数
maxTransferBytesOnMessageInDisk=6553500
#生产者最大发送线程池
sendThreadPoolQueueCapacity=100000
#消费者最大拉取线程池
pullThreadPoolQueueCapacity=1000000
#客户管理端线程池
clientManagerThreadPoolQueueCapacity=10000000
#消费管理端线程池
consumerManagerThreadPoolQueueCapacity=10000000
#发送等待时间
waitTimeMillsInSendQueue=10000
#发送线程
sendMessageThreadPoolNums=4
#刷入内存等待时间
osPageCacheBusyTimeOutMills=3000

~~~

##  修改启动参数

如果服务器的内存足够，无需修改

MQ根目录bin目录，对runbroker.sh，runserver.sh启动文件

**runbroker.sh**

~~~bash
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=1g"

~~~

**runserver.sh**

~~~bash
JAVA_OPT="${JAVA_OPT} -server -Xms2g -Xmx2g -Xmn1g -XX:PermSize=512m -XX:MaxPermSize=512m"
# 默认是磁盘90%,当磁盘空间达到90%,MQ就没办法处理数据,此设置可以提高98%
JAVA_OPT="${JAVA_OPT} -Drocketmq.broker.diskSpaceWarningLevelRatio=0.98"
~~~

##  启动MQ

> 先启动mqnamesrv服务 nohup sh mqnamesrv >/dev/null 2>log &
>
> 再启动broker服务 nohup sh mqbroker -c ../conf/2m-noslave/broker-57.properties >/dev/null 2>log &

##  查看进程

> jps -l
