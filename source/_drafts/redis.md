---
title: Redis
draft: true
date: 2021-01-10 20:49:08
tags: NoSQL
---

# 简介

Redis 是一款高性能 NoSQL 数据库，支持内存和磁盘存储。纯内存操作，单机可以支持每秒 5 万到 10 万的 O(1) 时间复杂度读写操作。Redis 使用 IO 多路复用技术处理客户端网络读写请求，服务端内部只有一个读写操作线程，结合 Pipeline 机制可以很好的实现多命令的原子操作。

## KV

支持 get/set 这样的 KV 操作，可以给 K 设置过期时间，也支持条件设置，比如当 K 不存在时才设置一个值。除了简单的 KV 数据结构，还支持集合和队列。

## 发布订阅

发布订阅可以使用消息的广播模式，当向一个主题发送一个消息时，所有订阅该主题的客户端都会收到这个消息。

## 过期

可以对 K 设置过期时间，比如使用 `EXPIRE` 命令。过期时间被覆盖的场景是 K 被删除或者内容被重新写入覆盖，比如这些命令都可以影响过期时间： `DEL`， `SET`， `GETSET` 和所有的 `*STORE` 命令，而 `INCR`，`LPUSH`，`HSET` 都不会改变过期时间。

Redis 删除一个过期 Key 的内存有两种模式，一种是主动模式，一种是被动模式。

被动模式最简单，就是当一个过期 Key 被再次读取时，Redis 有机会去检查 Key 是否过期，如果过期，删除数据，释放内存。如果一个过期 Key 再也不访问，此时就需要主动模式来释放内存。

被动模式稍微复杂，Redis 需要定期扫描过期 Key 的集合，1 秒钟执行 10 次，操作如下：

* 测试 20 个过期 key 是否过期；
* 删除过期 key；
* 如果操作 25% 的 key 过期，继续执行第一步；

过期策略还会影响主从复制。Redis 的 slave 不会执行过期检查，它接收 master 的 `DEL` 命令。当 master 检查到一个 Key 过期时，会向 AOF 文件和 slave 发送一个 `DEL` 命令。

## 事务

`MULTI`、`EXEC`、`DISCARD` 和 `WATCH` 是实现事务的基础，Redis 的事务不支持回滚。只要是没有实现 Write-Ahead Log 都不支持回滚。Redis 将一个事务的命令全部串行原子性执行，中间不会插入其他请求的命令，**Redis 保证事务内的所有命令要么全部执行，要么都不支持**。客户端开启事务时，使用 `MULTI`，后续的所有命令都会缓存在客户端，直到执行 `EXEC` 时把命令全部发送到服务端。Redis 服务端知道这是一个原子命令串，需要全部一起执行。	当开启 AOF 时 Redis 会使用一个系统操作将事务命令全部一起写入到文件，当恢复时发现命令不完整时，需要忽略这个命令，不过这是人工处理或者由 *redis-check-aof* 工具来检查。

## 分区

分区的方式分为两种，一种是 *range partition*，另一种是 *hash partition*。

*range partition* 需要使用一个表来维护哪些 key 范围映射到哪个 Redis 实例，状态的维护一般比较复杂。

*hash partition* 只需要将 key 计算出一个数值（比如 crc32），然后对实例数取模即可，不需要维护任何状态。结合一致性哈希算法可以减少实例增减带来的数据移动开销。

分区模式下，这些操作不方便使用：

* 涉及多个 key 的操作，比如 multiget key1 key2；
* 涉及多个 key 的事务；
* 无法平衡一个大 Key，比如一个 list，它非常大；
* 数据重平衡；

## 分布式锁

基于“set if not exists”的语义。需要考虑的点有：

* 锁的活跃性：需要使用过期时间；
* 互斥：1）删除锁时必须只能删除自己获取的锁，value 需要标记自己的身份；2）主从的 failover 模式不能满足互斥；

## 存储

Redis 支持两种存储方式：

* RDB：将内存数据写入文件，是内存数据的一个镜像（`fork`进程实现），用于实现快速恢复或者全量复制；
* AOF：记录 Redis 修改操作的命令，用于增量恢复；

使用 RDB 的优势：RDB 是一个紧凑的二进制文件，占用空间容量下，利于快速恢复。使用 `fork` 实现 RDB 存储，不阻塞 Redis 处理其他请求。

使用 RDB 的劣势：由于 RDB 是内存数据镜像，当内存数据非常大时，写磁盘或者 RDB 文件也非常大，中间会消耗很长时间，增量数据就不会存在 RDB 文件中，如果依靠 RDB 文件恢复，则数据丢失的情况容易发生。

使用 AOF 的优势：通过配置策略，可以每秒执行 `fsync` 写磁盘操作，这样丢失最多也是 1 秒的数据。由于 AOF 是 `append` 操作，写磁盘的数据快，且不会修改之前保存的数据。AOF 记录的是修改命令，文件的可读性强，可人工修改。

使用 AOF 的劣势：AOF 文件不断的 `append` ，文件会越来越庞大，如果使用 AOF 文件恢复，恢复时长大。而且 AOF 记录的是修改命令，恢复时执行修改命令的效率也不高（相比 RDB 而言）。

RDB-AOF 结合可以弥补各自的劣势：Redis 在 AOF 文件头写入 RDB，然后再写增量数据，不过它的劣势是文件头部是二进制，AOF 文件的可读性降低。这种方式有别于 Log rewriting。

关于 RDB 和 AOF 各自的优缺点，详细看[Redis Persistence](https://redis.io/topics/persistence)。

Redis 文件恢复的规则：

- 如果只配置 AOF ，重启时加载 AOF 文件恢复数据；
- 如果同时配置了 RDB 和 AOF ，启动是只加载 AOF 文件恢复数据；
- 如果只配置 RDB，启动时将加载 dump 文件恢复数据；

# 复制

可以配置 Redis 的实例为另一个 Redis 实例的 slave，master-slave 执行复制的操作如下<sup>[[ 1 ](https://redis.io/topics/replication)]</sup>：

* slave 与 master 建立网络连接，不断的从 master 的命令队列同步消息；
* 当网络端口时，slave 首先尝试 *partial resynchronization* 从命令队列同步，当命令队列缓存因为溢出被刷新过，只能通过 *full resynchronization*，
* 通过 *full resynchronization*， master 需要使用 CoW 复制一块内存快照用于复制；

复制基本原理见图-1所示。

![Redis 复制](/images/redis_replication.png "Redis 复制")

*图-1 Redis 复制*

需要注意的点：

* 第 1 步，如果首次与 master 建立链接时，直接使用 `SYNC` 命令全量复制，即执行第 2 步。如果与 master 断开后重连或者 master 重新选举，首先尝试使用旧 master 的 Replication ID 进行增量数据复制；
* 第 2 步，master 会使用后端线程将内存快照数据传到 slave，不会阻塞 master 的客户端请求，与此同时，客户端写入记录也会记录到 backlog 队列缓冲区；
* 第 3 步和第 4 步是 Redis 早期版本执行的操作，由于多了一步写内存快照到文件，导致效率不高，新版本已经支持直接将内存快照数据复制到 slave；
* 第 5 步发生在全量复制已经完成，slave 开始从 backlog 队列缓冲区复制数据。**全量复制期间，backlog 会包含客户端的写入记录，有可能 backlog 发生溢出，这时 slave 又将执行全量复制**；
* 第 6 步在异步复制时不会阻塞；
* 第 7 步，每次客户端写入记录或者 TTL 的 DEL 命令，都会塞到 backlog 队列，slave 从这个队列开始增量复制；

当 slave 与 master 断开或者 slave 开始全量同步时，slave 处理客户端请求的策略是：

* 如果 slave 配置 `slave-serve-stale-data=yes`，则 slave 拒绝客户端请求；
* 如果 slave 配置 `slave-serve-stale-data=no`，则 slave 继续用老的数据集处理客户端请求，当同步完成，slave 加载新的数据集时，依然会拒绝客户端请求。如果新数据集太大，拒绝的时间也会很长；

## 开启复制

发起 master-slave 有三种方式：

- `redis.conf` 文件中配置 `slaveof [masterip] [masterport]` 选项；
- `redis-server` 命令启动时指定参数 `--slaveof [masterip] [masterport]`；
- `redis-client` 模式下通过 `slaveof [masterip][masterport]` 命令执行绑定 master；

## 全量复制

全量复制的场景：

* slave 首次与 master 建立链接；
* backlog 缓冲区发生溢出，slave 来不及复制，导致 slave 不能与 master 达到最终一致；

全量复制的方式是 master 使用 Copy-on-Write 复制内存快照（也就是 `fork` 一个进程），将内存数据同步到 slave。建立内存快照的同时，master 依然可以接收客户端读写请求，写操作会记录在 backlog 缓冲区。master 将 RDB 同步到 slave 时，还会将建立快照后的第一个写操作的 offset 传给 slave ，以便 slave 做增量复制<sup>[[ 2 ](http://www.web-lovers.com/redis-source-replication.html)]</sup>。

## 增量复制

增量复制的场景：

* slave 已经获得了 master 的某个时间点的全量记录；
* `PSYNC` 指定的 offset 还存在 master 的 backlog 缓冲区； 

slave 通过 `PSYNC replicationid offset` 请求 master 进行增量复制，master 检查 replicationid 和 offset。每个 Redis 实例切换为 master 角色时都会生成一个 replicationid，表示 “replication history“，offset 表示复制记录的位置。

请求的 replicationid 检查的规则是：

* 如果请求的 replicationid 与 master 的 replid 相同，则 replicationid 正确；
* 如果请求的 replicationid 与 master 的 replid2 相同，则 replicationid 正确；

当请求的 replicationid 检查通过时，再检查 offset，规则是：

* replicationid = master.replid 时，offset 是否存在 backlog 中，如果存在则 offset 正确；
* replicationid = master.replid2 时，offset 是否在 secondary backlog 中，如果存在则 offset 正确；

当检查没有通过时，需要 slave 执行全量复制。规则检查的代码摘抄 `replication.c` 的 `int masterTryPartialResynchronization(client *c)` 方法如下：

```c
if (strcasecmp(master_replid, server.replid) &&
        (strcasecmp(master_replid, server.replid2) ||
         psync_offset > server.second_replid_offset))
{
	// replicationid 和 secondary backlog 检查失败，执行全量复制
	// ...
}
// ...
if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
{
	// psync_offset 小于 repl_backlog_off，说明 backlog 所备份的数据的已经太新了，有一些数据被覆盖，则需要进行全量复制
	// 如果 psync_offset 大于 (server.repl_backlog_off + server.repl_backlog_histlen)，表示当前 backlog 的数据不够全，则需要进行全量复制
}
```

为什么 master 需要两个 replid 呢？

考虑一种场景，A、B、C 三个 Redis 实例，A 是 master，B 和 C 是 slave，此时 B 和 C 都从 A 增量复制数据。当 A 停机，B 选举为 master，此时 C 需要向 B 增量复制数据。由于 B 的身份切换，它需要改变自己的 replid，此时 C 请求复制时使用的 replicationid 依然是 A 的 replid。如果 B 不把 A 的 replid 作为自己的 replid2 来做检查，B 肯定要做全量复制。全量复制的开销太大，使用 replid2 可以减少全量复制的场景。

## 异步复制

图-1 中的全量复制和增量复制都是异步复制，master 并不会阻塞客户端的读写请求。如果主从切换，新的 master 数据可能存在丢失的情况。

异步复制的优点：master 性能不受影响。

异步复制的缺点：容易出现数据不一致的情况。

如果追求数据强一致时，可以使用同步复制。

## 同步复制

Redis 的同步复制需要使用 `WAIT` 命令完成，命令格式是 `WAIT numreplicas timeout`。`WAIT` 命令只能保证命令被 slave 取走，不能保证 slave 已经保存了数据。在一个 CP 系统，Redis 同步复制也不能保证像 Zookeeper 这样的数据强一致。比如，Redis 收到 `WAIT` 返回，表示之前的修改命令被同步到 slave，此时 slave 的 AOF 写磁盘策略并未执行 `fsync` 操作，也就是数据还在内存中，如果此时 slave 停机，内存数据丢失。当再次启动时又发生了主从切换，导致数据丢失不可恢复。不过 Redis 同步复制依然可以减少数据丢失的风险，提高数据的一致性，无法做到强一致。

**要保存强一致，一定要使用分布式共识算法：Paxos，Zab，Raft。**

# 哨兵和高可用

要实现高可用，必须使用多副本。使用 Redis 复制功能，可以实现多副本，此时 slave 节点不支持写入，master 节点才能写入。当 master 节点异常，slave 需要替代 master 角色，晋升为 master 节点继续提供写入能力。此时需要解决三个问题：

* 如何检测 master 节点异常；
* slave 如何晋升为 master 节点；
* 如何选择晋升的 slave 并达成一致；

## 架构

Redis Sentinel 可以解决这三个问题实现自动故障转移，Sentinel 架构见图-2<sup>[[ 3 ](http://www.web-lovers.com/redis-source-sentinel.html)]</sup>。

![Redis 高可用](http://www.web-lovers.com/assets/bimg/redis_sentinel_1.png "Redis 高可用")

*图-2 Redis 高可用*

Redis Sentinel 的职责包括 master/slave 状态检测、自动故障转移、主从切换，因此 Sentinel 本身也需要高可用，需要多副本。

### Redis 数据节点

图-2中的 Redis 数据节点是 master、slave1 和 slave2 三个节点。Redis 数据节点高可用必须要求它的副本至少是 2 个，但是 2 个数据节点时也容易出现问题。

![Redis 2 Data Node](/images/redis_2_data_node.png)

*图-3 两个 Redis 数据节点出现网络分区*

图-3 中的 master 和 slave 出现网络分区导致脑裂时，slave 被晋升为 master，同时旧 master 还在提供服务，客户端的写入依然还有大量请求走到旧 master 节点，如果网络分区恢复，旧 master 需要从新 master 复制数据，然后把自己的数据丢弃，这样就丢失了大量数据。使用下方的配置可用缓解此问题。

```text
min-replicas-to-write 1
min-replicas-max-lag 10
```

此配置要求至少有一个 slave 可用并且数据滞后最大为 10 秒。使用该配置，图-3中的旧 master 在脑裂情况下会停止接收 Client 的写请求，降低数据丢失的数量。使用该配置时，至少需要三个副本，按照图-3中的两副本，发生脑裂时新晋升的 master 和旧 master 都因为相互不能复制导致不再继续提供写请求。

三个 Redis 数据节点是最经典的高可用架构，数据节点最好是分布在不同机器、不同机架上，原则是尽量避免一个错误或异常导致高可用架构失效。比如说，如果部署在同一台机器，机器断电，高可用架构失效。

### Redis Sentinel 节点

Redis Sentinel 的节点数必须是 3 且为奇数。原因是故障转移选主时需要使用 `quorum` 协议，即大多数同意一个提议，提议才被接受。最经典的架构如图-2所示，与 Sentinel 一起，如下所示<sup>[[ 4 ](https://redis.io/topics/sentinel)]</sup>：

```text
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```

M 表示 master， S 表示 Sentinel， R 表示 Replica。

## 发现

### Redis 数据节点之间

当 Redis 数据节点与 master 通过 `replicaof host port ` 命令建立主从关系时，master 可以知道自己有几个 slave 节点，slave 也知道自己的 master 节点。

### Sentinel 和 Redis 数据节点之间

master 节点知道其 slave 节点，Sentinel 连接 master 时通过 `INFO` 命令就可以获得所有的 slave 节点，所以配置 sentinel 时只需要指定 master 节点即可：`sentinel monitor <master-group-name> <ip> <port> <quorum>`。具体如下：

- `PING` sentinel 向 redis 数据节点发送 PING 命令，检查其状态
- `INFO` sentinel 向 redis 数据节点的 master 节点发送 INFO 命令，获取其他从服务器节点信息
- `PUBLISH` sentinel 向 redis 数据节点 `__sentinel__:hello` 频道发布自己的信息及 master 节点相关的配置
- `SUBSCRIBE` sentinel 通过订阅 redis 主从服务器节点的 `__sentinel__:hello` 频道，获取正在监控相同服务的其他 sentinel 节点信息

### Sentinel 和 Sentinel 之间

Sentinel 通过订阅 master/slave 节点的 `__sentinel__:hello` 可以获取正在监控相同服务的其他 sentinel 节点信息，

- `PING` sentinel 向其他 sentinel 节点发送 PING 命令，检查节点状态
- `is-master-down-by-addr` 和其他 sentinel 协商 master 的状态，如果 master 处于 SDOWN 状态，则投票自动选出新的 master。

### Client

Redis 客户端可以通过下面的方式获取当前主从架构的 master 节点信息：

* 连接 Redis Sentinel 时，通过 `SENTINEL get-master-addr-by-name mymaster` 获得 mymaster 主从架构的 master 节点；
*  订阅 Redis Sentinel 节点的 `+switch-master` 队列，它表示切换 master 节点的事件通知；

## 状态检测

Redis Sentinel 通过发现机制找到主从服务节点，接着需要不断的向这些节点发送 PING 命令，检查节点状态。可接受的 PING 命令回复有：

* +PONG
* -LOADING
*  -MASTERDOWN

如果 master 节点超过 `is-master-down-after-milliseconds` 没有回复 PING 命令，则认为 master 节点存在异常，Sentinel 将 master 节点状态标记为 SDOWN。Sentinel 认为 master 节点服务停止的状态有两个：

* SDOWN 主观下线（Subjectively Down），即 Sentinel 自己认为 master 已经下线，可能的情况是：1）master 确实下线，2）自己跟 master 的网络不通，master 实际没下线；
* ODOWN 客观下线（Objectively Down），多个 Sentinel 对 master 的检查状态都是 SDOWN，这个状态就排除了自己与 master 网络不通的情况。

如果 Sentinel 标记 master 为 SDOWN 时，如果 master 的 PING 回复恢复时，SDOWN 标记立即清除。

只有当 master 节点状态是 ODOWN 时，Sentinel 才真的认为 master 已经下线，需要执行故障转移 failover 操作，选出新的 master 节点。要标记为 ODOWN ，Sentinel 之间还需要协商。协商的算法是非强一致性的 *Gossip* 协议。具体过程是：

* master 超过 `is-master-down-after-milliseconds` 没有回复 PING 命令，Sentinel将其标记为 SDOWN；
* sentinel 通过 `is-master-down-by-addr` 命令询问其他 sentinel 节点关于 master 节点的状态，如果有 `quorum` 或者更多的 sentinel 节点都认为 master 是 SDOWN，则将 master 节点状态标记为 ODOWN；

检查 ODOWN 的代码在 [sentinel.c](https://github.com/redis/redis/blob/4.0/src/sentinel.c) 的 `sentinelCheckObjectivelyDown()` 方法，核心代码如下：

```c
// 如果已经被判断为主观下线
if (master->flags & SRI_S_DOWN) {
    // 当前 哨兵认为下线，投票1
    quorum = 1;
    di = dictGetIterator(master->sentinels);
    // 遍历所有监控该主节点的所有 sentinel
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);
        // 如果 sentinel 投票主观下线，投票 +1
        if (ri->flags & SRI_MASTER_DOWN) quorum++;
    }
    dictReleaseIterator(di);
    // 如果投票数够了，则认为 客观下线
    if (quorum >= master->quorum) odown = 1;
}
```



## 故障转移

Sentinel 检测到 master 状态是 ODOWN，需要执行故障状态转移，执行故障状态转移需要解决的问题有：

* 使用哪个 Sentinel 执行故障状态转移，这种协调性的操作只能由一个节点操作，不能多个 Sentinel 一起执行；
* 将哪个 slave 晋升为 master；

故障转移的状态有：

* SENTINEL_FAILOVER_STATE_NONE：没有故障转移；
* SENTINEL_FAILOVER_STATE_WAIT_START：等待故障转移选举投票；
* SENTINEL_FAILOVER_STATE_SELECT_SLAVE：选择晋升的 slave；
* SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE：将 slave 晋升为 master；
* SENTINEL_FAILOVER_STATE_WAIT_PROMOTION：等待晋升结果；
* SENTINEL_FAILOVER_STATE_RECONF_SLAVES：将其他节点重新配置连接到新的 master；
* SENTINEL_FAILOVER_STATE_UPDATE_CONFIG：

故障转移的协调器选举使用 Raft 算法完成，代码在 [sentinel.c](https://github.com/redis/redis/blob/4.0/src/sentinel.c)  ，步骤如下：

* `sentinelStartFailover()` ：`failover_epoch = ++sentinel.current_epoch`，Sentinel 增加自己的 `failover_epoch`，希望自己成为故障转移的协调节点，获得其他 Sentinel 节点的同意；
* `sentinelAskMasterStateToOtherSentinels()` ：向其他 Sentinel 发送  `SENTINEL IS-MASTER-DOWN-BY-ADDR ip port sentinel.current_epoch sentinel.myid ` 命令，请求投票；
* `sentinelVoteLeader()`：其他 Sentinel 收到 `SENTINEL IS-MASTER-DOWN-BY-ADDR` 的命令，如果请求的 epoch 比当前 Sentinel 的` current_epoch` 和 `leader_epoch` 大，则更新 `current_epoch` 和 `leader_epoch` 为 请求的 epoch，并把请求的 Sentinel 当作 leader，并回复；
* `sentinelReceiveIsMasterDownReply()`：收到其他 Sentinel 对 `SENTINEL IS-MASTER-DOWN-BY-ADDR` 命令的回复，记录这些 Sentinel 的投票信息，包括 `leader` 和 `leader_epoch`；
*  `sentinelFailoverWaitStart()`：通过 `leader = sentinelGetLeader(ri, ri->failover_epoch); ` 计算投票结果，返回的 `leader` 可能是自己，也可能是其他 Sentinel 节点。如果 `leader` 是自己，将自己的状态标记为 SENTINEL_FAILOVER_STATE_SELECT_SLAVE 状态；

leader 节点选举完成后，开始选择合适的 slave 节点晋升为 master 节点，步骤如下：

* `sentinelSelectSlave()`：从备选的 slave 节点中，通过 `compareSlavesForPromotion` 排序 slave 节点列表，规则是：优先级配置 slave_priority 小优先 > 复制偏移量 slave_repl_offset 大优先 > runid 字典序小的优先；
* `sentinelFailoverSelectSlave()`：将 sentinel 的 `failover_state` 状态改成 SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE ；

接下来，状态进入到“将 slave 晋升为 master”，在 `sentinelFailoverSendSlaveOfNoOne()` 方法中 Sentinel 通过 `sentinelSendSlaveOf()` 方法向 slave 节点发送 `slaveof no one` 命令，将 slave 晋升为 master，接着 Sentinel 进入 SENTINEL_FAILOVER_STATE_WAIT_PROMOTION ，等待晋升状态。

`sentinelFailoverWaitPromotion()` 方法处理 SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 状态逻辑，主要是超时处理，表示晋升失败。如果出现晋升失败，`sentinelAbortFailover()` 方法设置 `failover_state` 为 SENTINEL_FAILOVER_STATE_NONE。

`sentinelRefreshInstanceInfo()` 方法向晋升的 master 执行 `INFO` 命令，检查状态。如果晋升成功，设置 `failover_state` 为 SENTINEL_FAILOVER_STATE_RECONF_SLAVES 。此时需要更新其他 slave 节点的配置，从新晋升的 master 节点复制数据，由 `sentinelFailoverReconfNextSlave()` 方法完成。

## epoch

Redis 多次使用了 epoch 的概念来实现选举或者 `quorum` 算法，思想来自 Raft 的 Term 。每次 Redis 需要选举，都会使用一个更大的 epoch 来表示提议的版本。当选举成功后需要推送新配置时依然使用 epoch 来表示配置的版本，最有大的 epoch 才能替代小的 epoch 配置。

选举时，为了解决活跃性问题，每次由一个 Redis Sentinel 改变角色，变成选举的候选者，其他 Sentinel 进行投票。多个 Sentinel 需要使用一个随机的时间错开作为候选者进行选举的时间，减少同时由多个 Sentinel 发起选举。单一个 Sentinel 选举超时时，由其他 Sentinel 继续发起选举流程。

## TILT 模式

由于 Redis Sentinel 的故障检测和故障转移非常依赖时间，当机器时间调整或者 Sentinel 本身卡顿，会影响状态的判定。

Redis Sentinel 使用 TILT 模式解决这个问题。当进入 TILT 模式时，Sentinel 继续监控服务节点状态，还包括下面的操作：

* 不回复任何消息（*It stops acting at all*)；
* `SENTINEL is-master-down-by-addr` 回复负数，表示状态不可信；

TILT 的启用和禁用看下面的代码：

```c
void sentinelCheckTiltCondition(void) {
    mstime_t now = mstime();
    // 最近一次执行周期函数过了多久
    mstime_t delta = now - sentinel.previous_time;
    // 如果时间为 负值 或者相隔时间时间超过 2000毫秒，则进入 TILT 模式
    if (delta < 0 || delta > SENTINEL_TILT_TRIGGER) {
        // 进入 TILT 模式
        sentinel.tilt = 1;
        // TILT 模式开始时间
        sentinel.tilt_start_time = mstime();
        sentinelEvent(LL_WARNING,"+tilt",NULL,"#tilt mode entered");
    }
    // 设置最近一次 sentinel 时间处理程序的时间
    sentinel.previous_time = mstime();
}


void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
    // ...
    // 如果处于 TILT 模式，不进行故障检测
    if (sentinel.tilt) {
        // 如果 TILT 模式的时间还没到（默认 1000*30毫秒），则直接返回
        if (mstime()-sentinel.tilt_start_time < SENTINEL_TILT_PERIOD) return;
        // 超过时间，则退出 TILT 模式
        sentinel.tilt = 0;
        sentinelEvent(LL_WARNING,"-tilt",NULL,"#tilt mode exited");
    }
    // ...
}
```

# 集群

