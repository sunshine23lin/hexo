---
title: Nginx
date: 2020-12-01 14:55:34
categories: 负载均衡
tags: 负载均衡
---

#  Nginx的按照

##   准备工作

1. 打开官网http://nginx.org/下载nginx

   > nginx-1.19.5.tar.gz

2.  安装 openssl  、zlib  、 gcc  依赖

   > yum -y install make zlib zlib-devel gcc-c++ libtool openssl openssl-devel

3. 联网下载 pcre  压缩文件依赖

   >  1、wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz
   >
   > 2、tar –xvf pcre-8.37.tar.gz
   >
   > 3、./configure
   >
   > 4、install && make install

4. 安装 nginx

   > 1、tar -xvf nginx-1.19.5.tar.gz
   >
   > 2、./configure
   >
   > 3、install && make instal

   

5. 启动nginx

   > 先关闭防火墙，测试可以全部关掉，生产按照端口开发

   停止firewall

   > systemctl stop firewalld.service

   禁止firewall开机启动

   > systemctl disable firewalld.service

   查看防火墙状态

   > firewall-cmd --state

   进入/usr/local/nginx/sbin/nginx 启动命令

   > ./nginx

   ![image-20201202144955232](https://jameslin23.gitee.io/2020/12/01/ngnix//image-20201202144955232.png)

6. 可配置环境变量

   vim /etc/profile

   > PATH=$PATH:/usr/local/webserver/nginx/sbin
   > export PATH

   生效 source  /etc/profile

7. ngnix常用命令

   > 1、查看版本号
   >
   > nginx -v
   >
   > 2、启动
   >
   > nginx
   >
   > 3、停止
   >
   > nginx -s stop
   >
   > 4、重新加载
   >
   > nginx -s reload

   #  反向代理配置

   ##  反向代理-1

   ![image-20201202145725518](https://jameslin23.gitee.io/2020/12/01/ngnix//image-20201202145725518.png)

   

   ##  反向代理-2

   根据不同请求路径对应不同服务器

   ![image-20201202145811683](https://jameslin23.gitee.io/2020/12/01/ngnix//image-20201202145811683.png)

   

#  负载均衡

##  负载均衡算法

###  轮询(默认)

> 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除。

###  weight权重

> weight 代表权,重默认为 1,权重越高被分配的客户端越多

###  ip_hash

> 每个请求按方位ip的hash结果分配，这样每个访客固定一个后端服务器,可以解决session的问题

###  fair

> 按照后端服务器的响应时间来分配请求，响应时间短的优先分配

##  HTTP 负载均衡

![image-20201202150043431](https://jameslin23.gitee.io/2020/12/01/ngnix//image-20201202150043431.png)

##  TCP 负载均衡

> nginx-1.17.9使用增加了stream 模块用于一般的TCP 代理和负载均衡，ngx_stream_core_module 这个模块在1.90版本后将被启用。但是并不会默认安装

到nginx源文件目录

> ./configure  --with-stream
>
> make & make install

修改 配置文件

~~~js

stream {
     upstream kevin {
     server 192.168.10.10:8080; #这里配置成要访问的地址
     server 192.168.10.20:8081;
     }
     server {
     listen 8081; #需要监听的端口
     proxy_timeout 20s;
     proxy_pass kevin;
     }
    # 多个模块可增加多个sever+upstream
}
~~~

###  TCP负载算法

####  轮询（默认）

####  least-connected

> 对应每个请求,nginx plus选择当前连接数最少的server来处理

```java
upstream kevin {
 least_conn;
 server 192.168.10.10:8080; #这里配置成要访问的地址
 server 192.168.10.20:8081;
}
```

####  ip_hash

>  客户机的IP地址用作散列键，用于确定应该为客户机的请求选择服务器组中的哪个服务器

~~~java
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
~~~

####  hash

普通的hash算法：nginx plus选择这个server是通过user_defined 关键字，就是IP地址：$remote_addr;

~~~java

upstream kevin {
 hash $remote_addr consistent;
 server 192.168.10.10:8080 weight=5; #这里配置成要访问的地址
 server 192.168.10.20:8081 max_fails=2 fail_timeout=30s;
 server 192.168.10.30:8081 max_conns=3; #需要代理的端口，在这里我代理一一个kevin模块的接口8081
}
~~~



#  静态分离

待编辑

#  nginx限流

nginx提供两种限流的方式

- **控制速率**
- **控制并发连接数**

##  控制速率

**原理**

采用漏桶算法实现控制速率限流

漏桶(Leaky Bucket) 算法思路很简单，水(请求)先进入到漏桶里，漏桶以一定的速度出水(接口有响应速率),当水流入速度过大会直接溢出（访问频率超过接口响应速率），然后就拒绝请求，可以看出漏桶算法能强行限制数据的传输速率。

![image-20201224085740670](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224085740670.png)

**配置1**

```nginx
user  root root;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #cache
    lua_shared_dict dis_cache 128m;

    #限流设置
    limit_req_zone $binary_remote_addr zone=contentRateLimit:10m rate=2r/s;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        location /update_content {
            content_by_lua_file /root/lua/update_content.lua;
        }

        location /read_content {
            #使用限流配置
            limit_req zone=contentRateLimit;
            content_by_lua_file /root/lua/read_content.lua;
        }
    }
}
```

**配置参数说明**

- **binary_remote_addr**: 是一种key,表示基于remote_add(客户端)来做限流，binary_ 目的压缩内存占用量。
- **zone**: 定义共享内存区来存储访问信息，contentRateLimit:10m 表示一个大小为10m,名字为contentRateLimit的内存区域。1M能存储16000 IP地址访问信息，10M可以存储16万IP地址访问
- **rate**: 用于设置最大访问速率,rate=10r/s 表示每秒最多处理10请求。nginx实际上以毫秒为颗粒度来跟踪请求信息。因此10r/s实际上是限制每100毫秒处理一个请求。这意味着自上一个请求处理完，若后续100毫秒内又有请求到达，将拒接该请求。我们这里设置为2，方便测试。

**测试**

访问http://192.168.10.132/read_content?id=1，连续刷新就会报错

![image-20201224091808770](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224091808770.png)

**配置2**

上面例子限制2r/s，如果有时正常流量突然增大，超过的请求将被拒接，无法处理突发流量，可以结合**burst**参数使用来解决该问题

```nginx
server {
    listen       80;
    server_name  localhost;
    location /update_content {
        content_by_lua_file /root/lua/update_content.lua;
    }
    location /read_content {
        limit_req zone=contentRateLimit burst=4;
        content_by_lua_file /root/lua/read_content.lua;
    }
}
```

burst译为突发、爆发，表示在超过设定的处理速率后能额外处理的请求，当rate=10r/s时，将1s拆分10份，既每100ms可以处理1个请求。此处burst=4,若同时有4个请求达到,nginx会处理第一个请求。剩余3个请求将放入队列，然后每隔100ms从队列中获取一个请求进行处理。若请求大于4，将拒接处理多余的请求，直接返回503.

不过，单独使用 burst 参数并不实用。假设 burst=50 ，rate依然为10r/s，排队中的50个请求虽然每100ms会处理一个，但第50个请求却需要等待 50 * 100ms即 5s，这么长的处理时间自然难以接受。

因此，burst往往结合**nodelay** 一起使用

```nginx
limit_req zone=contentRateLimit burst=4 nodelay;
```

平均每秒允许不超过2个请求，突发不超过4个请求，并且处理突发4个请求的时候，没有延迟，等到完成之后，按照正常的速率处理。

##  控制并发连接数

**配置限制固定连接数**

ngx_http_limit_conn_module  提供了限制连接数的能力。主要是利用limit_conn_zone和limit_conn两个指令。

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    #cache
    lua_shared_dict dis_cache 128m;

    #限流设置
    limit_req_zone $binary_remote_addr zone=contentRateLimit:10m rate=2r/s;

    #根据IP地址来限制，存储内存大小10M
    limit_conn_zone $binary_remote_addr zone=addr:1m;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;
        #所有以brand开始的请求，访问本地changgou-service-goods微服务
        location /brand {
            limit_conn addr 2;
            proxy_pass http://192.168.211.1:18081;
        }

        location /update_content {
            content_by_lua_file /root/lua/update_content.lua;
        }

        location /read_content {
            limit_req zone=contentRateLimit burst=4 nodelay;
            content_by_lua_file /root/lua/read_content.lua;
        }
    }
}
```

> limit_conn_zone $binary_remote_addr zone=addr:10m;  表示限制根据用户的IP地址来显示，设置存储地址为的内存大小10M
>
> limit_conn addr 2;   表示 同一个地址只允许连接2次。

**限制每个客户端IP与服务器的连接数，同时限制与虚拟服务器的连接总数**

```nginx
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m; 
server {  
    listen       80;
    server_name  localhost;
    charset utf-8;
    location / {
        limit_conn perip 10;#单个客户端ip与服务器的连接数．
        limit_conn perserver 100; ＃限制与服务器的总连接数
        root   html;
        index  index.html index.htm;
    }
}
```





#  Keepalived高可用

![image-20201202152503644](https://jameslin23.gitee.io/2020/12/01/ngnix//image-20201202152503644.png)

##  安装keepalived

一种是用yum安装，一种是下载安装包，这里使用yum

>  yum install keepalived –y

安装之后在/etc/keepalived,有个keepalived.conf

~~~java
! Configuration File for keepalived

global_defs {
	notification_email {
		acassen@firewall.loc
		failover@firewall.loc
		sysadmin@firewall.loc
	}
	notification_email_from Alexandre.Cassen@firewall.loc
	smtp_server 192.168.10.8
	smtp_connect_timeout 30
	router_id LVS_DEVEL
	}
vrrp_script monitor_nginx {
	script "/usr/local/src/nginx_check.sh"
	interval 2 #（检测脚本执行的间隔）
	weight 2  #  检测失败（脚本返回非0）则优先级 -2
    fall 2  # 检测连续 2 次失败才算确定是真失败。会用weight减少优先级（1-255之间）
	}
vrrp_instance VI_1 {
	state MASTER # 备份服务器上将 MASTER 改为 BACKUP
	interface ens33 # 网卡
	virtual_router_id 51 # 主、备机的 virtual_router_id 必须相同
	priority 100 # 主、备机取不同的优先级，主机值较大，备份机值较小
	advert_int 1

authentication {
	auth_type PASS
	auth_pass 1111
}
	virtual_ipaddress {
	  192.168.10.33 #  VRRP H 虚拟地址
	}
	track_script {
        monitor_nginx
    }
}

~~~

检测脚本

~~~bash
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    /usr/local/webserver/nginx/sbin/nginx  #尝试重新启动nginx
    sleep 1  #睡眠1秒
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then
        killall keepalived #启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    fi
fi

~~~

启动keepalived

> systemctl start keepalived.service

查看状态

> systemctl status keepalived.service

关闭keeaplived

> systemctl stop keepalived.service
>
> 强关进程
>
> pkill keepalived

# 问题总结

**1、/etc/keepalived/chk_nginx.sh exited due to signal 15**

> vrrp_script interval的值要修改成小于chk_nginx.sh中sleep的值
>
> chk_nginx.sh sleep的值不宜过大最好不要超过3，否则也会出现exited due to signal 15问题，偶尔会出time out问题



