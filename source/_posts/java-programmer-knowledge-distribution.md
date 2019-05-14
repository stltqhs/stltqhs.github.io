---
title: Java后端开发 - 分布式
date: 2019-05-13 12:06:20
tags:
---

# 分布式系统原理

CAP理论和BASE理论。

[分布式系统原理介绍](http://blog.sciencenet.cn/home.php?mod=attachment&id=31413)。

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

