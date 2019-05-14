---
title: Java后端开发 - 分布式
date: 2019-05-13 12:06:20
tags:
---

# CAP和BASE理论

- 一致性分类：强一致(立即一致)；弱一致(可在单位时间内实现一致，比如秒级)；最终一致(弱一致的一种，一定时间内最终一致)
- CAP：一致性、可用性、分区容错性(网络故障引起)
- BASE：Basically Available（基本可用）、Soft state（软状态）和Eventually consistent（最终一致性）
- BASE理论的核心思想是：即使无法做到强一致性，但每个应用都可以根据自身业务特点，采用适当的方式来使系统达到最终一致性。

参考：[从分布式一致性谈到CAP理论、BASE理论](http://www.cnblogs.com/szlbm/p/5588543.html)

# 分布式事务

2PC的[原理](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)及实现[Raincat](https://github.com/yu199195/Raincat)和[XA](https://en.wikipedia.org/wiki/X/Open_XA)。

3PC的[原理](https://en.wikipedia.org/wiki/Three-phase_commit_protocol)。

TCC的原理及实现[ByteTCC](https://github.com/liuyangming/ByteTCC)和[hmily](https://github.com/yu199195/hmily)。

基于可靠消息的事务实现[myth](https://github.com/yu199195/myth)。

实际应用中，更多的是使用基于可靠消息的事务，使用事物补偿机制来保证分布式环境中数据的一致性。

# 一致性算法

实现[Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)算法的有[Chubby](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf)和[PhxPaxos](https://github.com/Tencent/phxpaxos)。

实现[Raft](https://www.infoq.cn/article/raft-paper)算法的有[etcd](https://github.com/etcd-io/etcd)。

实现[Zab]()算法的有[Zookeeper](https://github.com/apache/zookeeper)。

