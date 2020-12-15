---
title: redis哨兵模式
date: 2020-12-05 14:55:30
categories: Nosql
tags: 缓存
---

#   哨兵模式

**Sentinel(哨兵)是redis的高可用性解决方案:由一个或者多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器以及这些主服务器属下所有从服务器，并在被监视的主服务器进入下线状态时,自动将下线主服务器属下的某个从服务器升级为新的服务器，然后由新的主服务器代替已下线的主服务器继续处理请求命令,他主要功能如下:**

- **监控(Monitoring)**：Sentinel会不断地检查你的主服务器和从服务器是否运作正常。
- **通知(Notification)**：当被监控的某个 Redis 服务器出现问题时， Sentinel可以通过API向管理员或者其他应用程序发送通知。
- **故障迁移**：当主服务器不能正常工作时，Sentinel会自动进行故障迁移，也就是主从切换。
- **统一的配置管理**：连接者询问sentinel取得主从的地址。

##  哨兵原理

**Sentinel 使用核心算法是Raft算法,主要用途就是用于分布式系统,系统容错以及Leader选举,每个Sentinel都需要定期的执行以下任务:**

**主观下线**(一个Sentinel判断)

在默认情况下,Sentinel会以每秒一次的频率像所有与它创建了命令连接的实例(包括主服务器,从服务器,其它Sentinel在内)发送PING命令回复来判断实例是否在线。

实例对PING命令的回复可以分为以下两种情况：

- 有效回复:实例返回+PONG,-LOADING、-MASTERDOWN三种回复其中一种。
- 无效回复:有效回复之外的其它三种回复或者在指定时限内没有任何回复。



**客观下线**

当Sentinel将一个主服务器判断为主观下线之后,为了确认这个主服务器是否真的下线了,它会向同样监视这个主服务器的其他Sentinel进行询问,看他们是否也认为主服务器已经进入下线状态,当Sentinel从其他Sentinel那里接收到足够数量的已经判断下线之后,Sentinel就会将服务器判断为客观下线,并对主服务器执行故障转移。

![image-20201207141544018](https://jameslin23.gitee.io/2020/12/05/redis哨兵模式/image-20201207141544018.png)

**如何从主机选取主机**

**故障转移操作的第一步** 要做的就是在已下线主服务器属下的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送 `slaveof no one` 命令，将这个从服务器转换为主服务器。但是这个从服务器是怎么样被挑选出来的呢？

- 在失效主服务器属下的从服务器当中， 那些被标记为主观下线、已断线、或者最后一次回复 PING 命令的时间大于五秒钟的从服务器都会被 **淘汰**。
- 在失效主服务器属下的从服务器当中， 那些与失效主服务器连接断开的时长超过 down-after 选项指定的时长十倍的从服务器都会被 **淘汰**。
- 在 **经历了以上两轮淘汰之后** 剩下来的从服务器中， 我们选出 **复制偏移量（replication offset）最大** 的那个 **从服务器** 作为新的主服务器；如果复制偏移量不可用，或者从服务器的复制偏移量相同，那么 **带有最小运行 ID** 的那个从服务器成为新的主服务器。

#  部署

**配置文件详解**

哨兵的配置主要就是修改sentinel.conf配置文件中的参数，在Redis安装目录即可看到此配置文件，各参数详解如下:

~~~properties
# 哨兵sentinel实例运行的端口，默认26379   
port 26379 
# 哨兵sentinel的工作目录 
dir ./ 
# 是否开启保护模式，默认开启。 
protected-mode no 
# 是否设置为后台启动。 
daemonize yes 
 
# 哨兵sentinel的日志文件 
logfile:./sentinel.log 
 
# 哨兵sentinel监控的redis主节点的  
## ip：主机ip地址 
## port：哨兵端口号 
## master-name：可以自己命名的主节点名字（只能由字母A-z、数字0-9 、这三个字符".-_"组成。） 
## quorum：当这些quorum个数sentinel哨兵认为master主节点失联 那么这时 客观上认为主节点失联了   
# sentinel monitor <master-name> <ip> <redis-port> <quorum>   
sentinel monitor mymaster 127.0.0.1 6379 2 
 
# 当在Redis实例中开启了requirepass，所有连接Redis实例的客户端都要提供密码。 
# sentinel auth-pass <master-name> <password>   
sentinel auth-pass mymaster 123456   
 
# 指定主节点应答哨兵sentinel的最大时间间隔，超过这个时间，哨兵主观上认为主节点下线，默认30秒   
# sentinel down-after-milliseconds <master-name> <milliseconds> 
sentinel down-after-milliseconds mymaster 30000   
 
# 指定了在发生failover主备切换时，最多可以有多少个slave同时对新的master进行同步。这个数字越小，完成failover所需的时间就越长；反之，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为1，来保证每次只有一个slave，处于不能处理命令请求的状态。 
# sentinel parallel-syncs <master-name> <numslaves> 
sentinel parallel-syncs mymaster 1   
 
# 故障转移的超时时间failover-timeout，默认三分钟，可以用在以下这些方面： 
## 1. 同一个sentinel对同一个master两次failover之间的间隔时间。   
## 2. 当一个slave从一个错误的master那里同步数据时开始，直到slave被纠正为从正确的master那里同步数据时结束。   
## 3. 当想要取消一个正在进行的failover时所需要的时间。 
## 4.当进行failover时，配置所有slaves指向新的master所需的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来同步数据了 
# sentinel failover-timeout <master-name> <milliseconds>   
sentinel failover-timeout mymaster 180000 
 
# 当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本。一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。 
# 对于脚本的运行结果有以下规则：   
## 1. 若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10。 
## 2. 若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。   
## 3. 如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。 
# sentinel notification-script <master-name> <script-path>   
sentinel notification-script mymaster /var/redis/notify.sh 
 
# 这个脚本应该是通用的，能被多次调用，不是针对性的。 
# sentinel client-reconfig-script <master-name> <script-path> 
sentinel client-reconfig-script mymaster /var/redis/reconfig.sh 
~~~

**对应配置修改**

~~~properties
# 端口默认为26379。 
port:26379 
# 关闭保护模式，可以外部访问。 
protected-mode:no 
# 设置为后台启动。 
daemonize yes 
# 日志文件。 
logfile "/data/redis/redis-5.0.7/logs/sentinel.log"
# 指定主机IP地址和端口，并且指定当有2台哨兵认为主机挂了，则对主机进行容灾切换。 
sentinel monitor mymaster 192.168.10.8 6379 2 
# 当在Redis实例中开启了requirepass，这里就需要提供密码。 
sentinel auth-pass mymaster 123456 
# 这里设置了主机多少秒无响应，则认为挂了。 
sentinel down-after-milliseconds mymaster 3000 
# 主备切换时，最多有多少个slave同时对新的master进行同步，这里设置为默认的1。 
snetinel parallel-syncs mymaster 1 
# 故障转移的超时时间，这里设置为三分钟。 
sentinel failover-timeout mymaster 180000 
~~~

**启动三个哨兵：**

> cd /data/redis/redis-5.0.7/bin
>
> redis-sentinel  ../conf/sentinel.conf 

**查看状态**

> 1. redis-cli -p 26379 
> 2. info sentinel 



![image-20201207140008319](https://jameslin23.gitee.io/2020/12/05/redis哨兵模式/image-20201207140008319.png)