---
title: Memcache
date: 2020-12-09 20:50:10
categories: Nosql
tags: 缓存
---

##   memchache特点

- MC处理请求时使用多线程异步IO方法，可以合理利用CPU多核的优势，性能非常优秀。
- MC功能简单，使用内存存储数据
- MC 对缓存的数据可以设置失效期，过期后的数据会被清除；
- 失效的策略采用延迟失效，就是当再次使用数据时检查是否失效；
- 当容量存满时，会对缓存中的数据进行剔除，剔除时除了会对过期 key 进行清理，还会按 LRU 策略对数据进行剔除。



###  缺陷

- key不能超过250个字节
- value不能超过1M字节
- key的最大失效时间是30天
- 只支持K-V结构，不能供持久化和主从同步功能
- 没有原生的集群模式，需要依靠客户端来实现集群中分片写数据。

