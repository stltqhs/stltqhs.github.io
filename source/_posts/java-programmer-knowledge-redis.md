---
title: Java后端开发 - redis
date: 2018-10-13 08:15:43
tags: redis
---



# 基本数据类型和指令

Redis 有 5 种基础数据结构，分别为：string (字符串)、list (列表)、set (集合)、hash (哈希) 和 zset (有序集合)。

[setnx](https://redis.io/commands/setnx)实现分布式锁

[lpush](https://redis.io/commands/lpush)/[rpop](https://redis.io/commands/rpop)或[rpush](https://redis.io/commands/rpush)/[lpop](https://redis.io/commands/lpop)实现队列

[位图统计](https://redis.io/commands/bitfield)

[HyperLogLog](http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html)：[pfadd](https://redis.io/commands/pfadd)，[pfcount](https://redis.io/commands/pfcount)，[pfmerge](https://redis.io/commands/pfmerge)

由于HyperLogLog没有提供类似pfexists指令，可以使用Redis插件[Bloom Filter](https://github.com/RedisLabsModules/redisbloom)来实现类似操作。

[Redis-cell](https://github.com/brandur/redis-cell)限流

[Geohash](https://redis.io/commands/geoadd)：将二维查找变为一维查找

不要使用`keys *`，要使用[scan](https://redis.io/commands/scan)

[Redis事务实现](https://redis.io/topics/transactions)

[发布与订阅](https://redis.io/topics/pubsub)

# 过期策略

主动过期和被动过期，见[ How Redis expires keys](https://redis.io/commands/expire#how-redis-expires-keys)。

# redis内存淘汰机制

[Using Redis as an LRU cache](https://redis.io/topics/lru-cache#using-redis-as-an-lru-cache)

# 复制

[Replication](https://redis.io/topics/replication)

[Sentinal](https://redis.io/topics/sentinel)

# 集群

Redis Cluster[教程](https://redis.io/topics/cluster-tutorial)和[规范](https://redis.io/topics/cluster-spec)

[Codis](https://github.com/CodisLabs/codis)

[Twemproxy](https://github.com/twitter/twemproxy)

# 线程模型

[Redis 和 I/O 多路复用](https://blog.csdn.net/tanswer_/article/details/70196139)

# 优化

[管道](https://redis.io/topics/pipelining)

[内存优化](https://redis.io/topics/memory-optimization)

# Redis源码

[redis源码分析](http://foolchild.cn/2016/09/01/redis_source)

[Redis 设计与实现（第一版）](https://redisbook.readthedocs.io/en/latest/index.html)

[深入学习Redis（1）：Redis内存模型](https://www.cnblogs.com/kismetv/p/8654978.html)

[Debugging Redis with Xcode](https://billmill.org/xcode_redis.html)

[Writing a Custom Redis Command In C - Part 1](https://www.openmymind.net/Writing-A-Custom-Redis-Command-In-C-Part-1/)

推荐书籍：[Redis 深度历险：核心原理与应用实践](https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section)，[Redis设计与实现](https://book.douban.com/subject/25900156/)