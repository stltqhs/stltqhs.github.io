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

## 架构

Redis 集群架构图如图-4所示。

![Redis Cluster Architecture](/images/redis_cluster_arch.png)

*图-4 Redis 集群*

Redis 集群数据存储是非中心化，因此每个节点都存储一部分数据。图-4是由 3 个 master 节点和 4 个 slave 节点组成的集群。master 节点支持读写操作，slave 节点只支持读操作，而且需要从 master 节点复制数据。当 master 节点停止服务时，其他 2 个 master 节点没有数据副本，只能让停止服务的 master 节点的 slave 节点晋升为 master 节点继续提供服务。如果停止服务的 master 节点没有活跃可用的 slave 节点时，整个集群将会停止服务。保证集群的可用性，主要是保证 master 节点由可用的 slave 节点。集群内每个 master 节点只有一个 slave 节点时，集群停止服务的概率还是比较高的，比如 slave 晋升为 master 节点后也停止了服务，而旧 master 并为恢复变成 slave 节点，导致停止服务的 master 节点没有可用的 slave。如果每个 master 节点由两个 slave 节点，又可能造成资源浪费。图-4中，Node A Master 节点有两个 slave 节点。Redis Cluster 支持 slave 节点的平衡，自动发现哪些 master 节点没有 slave，哪些 master 的 slave 节点过多，改变 slave 节点的 master，这称为[Replica migration](https://redis.io/topics/cluster-spec#replica-migration)。

**Redis Cluster 是异步复制，使用 Gossip 协议同步分片配置信息，属于最终一致性。**

## 发现

### 键空间分布

Redis Cluster 将 key 空间分为 16384 个槽，每个 Redis 实例领取一部分槽位，直到槽位全部分配完。槽位的计算方式如下代码片段所示。

```text
HASH_SLOT = CRC16(key) mod 16384

       +------------------+
Node 1 | slot 1-5000      |
       +------------------+
Node 2 | slot 5001-10000  |
       +------------------+
Node 3 | slot 10001-16384 |
       +------------------+
```

如果使用 `multiget` 或者批命令请求 Redis 时可能会出现问题，因为多个 key 的槽位不同。可以使用 *hash tags* 将多个 key 都分配在同一个槽位。

### hash tags

有时我们希望根据 key 中的一部分来计算分片数，这样可以将相同类型的 key 都分配在同一个槽。*hash tags* 是将 key 按照特定格式命名，特定格式是 `{pattern}`，中括号内的 pattern 会拿出来计算槽位，比如 `sns:{userid}:1` 和 `sns:{userid}:2` 的 pattern 都是 userid，他们会放在同一个槽位，具体见 [Keys hash tags](https://redis.io/topics/cluster-spec#keys-hash-tags)。

### 节点属性

节点保护的属性有：

- Node id，从 `/dev/urandom` 读取的 160 位长度的字符；
- IP 和 端口；
- 一系列标记，比如是否为 master；
- 如果被标记为 slave，还包含它的 master；
- 最后一次 ping 发出时间和 pong 接收时间；
- `configuration epoch`；
- 链接状态；
- hash 槽；

查看集群节点属性的方法：

```sh
$ redis-cli cluster nodes
d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
d289c575dcbc4bdd2931585fd4339089e461a27d 127.0.0.1:6381 master - 1318428931 1318428931 3 connected 2730-4095
```

### 集群总线

每个加入到 Redis Cluster 的节点都会打开一个端口，通过该端口建立TCP网络链接用来交换集群内其他节点的消息，该端口是 Redis 端口再加10000，比如 Redis 端口是 6379，则集群总线端口就是 16379。消息交换协议是 Redis 私有消息，可通过 `cluster.c` 和 `cluster.h` 研究协议，然后伪装一个 Redis 节点用于其他目的的应用。

### 集群拓扑

由于集群内的节点是两两建立链接通信，这样就组成了一个网状结构。假设集群内有 `n` 个节点，那么每个节点需要建立 `n-1` 个长链接与剩下的节点通信。为了避免因链接坏死而无法交换消息，Redis Cluster 有一套链接保活机制：

- 首先链接属于长链接，一直打开，定期发送 ping 消息给其他节点；
- 如果其他节点长时间没有恢复 pong 消息，Redis 会重新建立一个新链接，使用新链接发送消息；

### Gossip 协议

图-4显示，Redis 集群内部，各节点使用 Gossip 协议来交换状态数据，这里说说什么是 Gossip 协议。Gossip 协议有一个特征是 Epidemic，即传染性，想想现实世界中的传染病，如果人群中有一个人感染了病毒（数据状态），如果传播途径一直存在，最终这群人将全部感染这个病毒（数据状态）。Gossip 协议来源于生活，Redis 集群内的一个节点 A 将自己的状态（数据）传播给另一个节点 B，B 检查状态（数据）的版本是否是最新的，如果是最新的状态（数据），则 B 需要加下来，并且继续将该状态（数据）传播给节点 C，这种传播策略一直执行下去，最终集群内的全部节点都有这份状态（数据）。所以 Gossip 协议是最终一致性，而非强一直性，它的优点是**非中心化，可用性高**，缺点是**需要避免数据的大量传播带来的带宽消耗，状态数据存在一定延时**。Redis 集群的传播策略如下：

- 每秒发送 PING 消息，对端节点收到 PING 消息时需要恢复 PONG 消息，PONG 消息包括节点的配置信息；
- 检查收到的配置信息是否最新，如果是最新，除了自己保存外，还需要传播给其他节点；
- 如果的配置信息不是最新，直接丢弃配置信息，不再传播，减少带宽消耗；

第2和第3步是配置更新策略，通过 configuration epoch 检查配置版本是否比自己的 epoch 大来判断配置是否最新。

### 握手

节点 A 通过集群总线端口与集群中的一个节点 B 建立链接，然后发送 `MEET` 消息。`MEET` 消息与 PING 消息类似，它会强制节点 B 把 节点 A 加入本集群，节点 B 使用 Gossip 谢谢把集群新配置传播给节点 C，最终所有集群内所有的节点都知道 A 节点是集群的成员。

发送 `MEET` 消息的方式是管理员通过命令：

```text
CLUSTER MEET ip port
```

来完成，该命令指定来节点的 IP 和端口。新节点的角色是 master 还是 slave 也可以在加入后指定。如果新节点是 slave 节点，需要从 master 节点复制数据。如果新节点是 master 节点，需要从其他 master 节点领取数据，槽位数据迁移，数据重平衡。

## 数据分片

### `MOVED` 重定向

图-4所示，3 个 master 节点 A、B、C，每个节点都负责服务一部分健空间，总共 16384 个槽位。客户端连接节点 A 执行命令 `GET X`，节点 A 会计算 X 属于哪个槽位，如果计算出来的槽位是 4567，该槽位是 B 来提供服务，因此 A 需要回复给客户端 `-MOVED 4567 Bip Bport`，此时客户端应该记下槽位的状态，然后跑到节点 B 去执行命令 `GET X` 。客户端需要自己维护槽位到主机IP和PORT的映射，以便下次直连，而不是依赖 `MOVED` 重定向，它会增加延时。`MOVED` 重定向只是数据可用性的保底策略。

一般来说，客户端应该首先通过命令 `CLUSTER SLOTS` 获得集群内所有的节点负责的槽位信息。当读写 Redis 时提前计算好应该与哪个节点直连发送命令，减少因为重定向带来的网络 RTT，它会带来延时，降低性能。

### 数据迁移

Redis 集群运行过程中支持动态的添加节点和删除节点，这就如同是给飞行中的飞机换引擎。添加和删除节点被抽象成槽位的数据迁移。添加节点，集群内其他 master 节点的槽位需要迁移到新的节点上。删除节点，被删除节点的槽位数据需要迁移到集群内的其他 master 节点。重平衡就是一个槽位从负载高的节点迁移到另一个负载低的节点，槽位的动态配置的命令有：

- [CLUSTER ADDSLOTS](https://redis.io/commands/cluster-addslots) slot1 [slot2] ... [slotN]
- [CLUSTER DELSLOTS](https://redis.io/commands/cluster-delslots) slot1 [slot2] ... [slotN]
- [CLUSTER SETSLOT](https://redis.io/commands/cluster-setslot) slot NODE node
- [CLUSTER SETSLOT](https://redis.io/commands/cluster-setslot) slot MIGRATING node
- [CLUSTER SETSLOT](https://redis.io/commands/cluster-setslot) slot IMPORTING node

迁移最基本的两个操作是 `MIGRATING` 和 `IMPORTING`，其他的 `ADDSLOTS` 和 `DELSLOTS` 都会使用这两个最基本的命令。

那么迁移是如何进行，迁移过程中的数据如何访问呢？

假如我们想把节点 A 的 slot 8 迁移到节点 B，进行下面几个步骤。

第一步，先标记 slot 状态为迁移状态，执行操作：

* 在节点 B，执行 ` CLUSTER SETSLOT 8 IMPORTING A`；
* 在节点 A，执行 `CLUSTER SETSLOT 8 MIGRATING B`；

此时 slot 8 处于迁移状态，属于不稳定状态，集群内其他节点依然认为 slot 8 是节点 A 提供服务的。

第二步，通过 `CLUSTER GETKEYSINSLOT slot count ` 获得 slot 8 下 count 个 key。因为迁移过程是循序渐进，不能一气呵成，而且大 key 迁移更慢，所以一次获得一部分 key，然后对这些 key 执行迁移操作。这部分 key 迁移完成后，继续执行 `CLUSTER GETKEYSINSLOT slot count `，直到所有的 key 都迁移完成。现在我们可以拿到 slot 下要迁移的 key ，在节点 A 执行命令：

```text
MIGRATE target_host target_port key target_database id timeout
```

这个命令可以将 key 从节点 A 迁移到 节点 B，当 key 成功迁移到节点 B 时，节点 A 需要删除 key。对于集群环境下，`target_database` 只能是 0。`MIGRATE` 命令还指定了超时时间，

这样不停的获取迁移 key 然后 `MIGRATE` 迁移 key，最终会把所有的 key 都迁移完。

第三步，当 slot 8 所有的 key 都迁移完成后，节点 A 和节点 B 都执行 `CLUSTER SETSLOT slot NODE node` 将 slot 8 设置为正常状态，此时集群稳定下来。

###  `ASK` 重定向

在数据迁移的第二步，由于 slot 迁移过程中，有些 key 已经不在 A 节点，而是在 B 节点，当继续查 slot 的 key 时，如果在 A 节点找不到，将会发起一个重定向，将查询请求转给节点 B，这里的重定向就是 `ASK`。

为什么不使用 `MOVED` 呢？因为 `MOVED` 是集群稳定下的重定向，它会让客户端更新槽位和节点的映射，记下状态，下次就可以直连。而在迁移过程中，并不需要下次直连节点 B，因为客户端下一次访问的 key 可能还在节点 A。

客户端收到 `ASK` 重定向时，转而请求节点 B 时应该要带上 `ASKING` 命令，要不然，节点 B 会认为该 key 对应的槽位不在自己这里，恢复一个 `MOVED` 重定向。因为迁移过程中，节点服务哪些槽位的配置并不会更新，只有迁移完成后才会更新。

###  操作多个 KEY 的问题

数据迁移过程中，如果使用 hash tags 访问同一个槽位的多个 key 时会存在问题，比如：

```
MSET sns:{userid}:1 sns:{userid}:2
```

可能 `sns:{userid}:1` 存在节点 A，而 `sns:{userid}:2` 存在节点 B，此时会回复 `-TRYAGAIN` 的错误。

### Slave 节点

在 Redis 集群中，同样可以将 slave 设置为 READONLY 来提高读性能。slave 提供的槽位信息与它的 master 保持一致。所以，如果客户端发起请求的 key 不在其 master 的服务访问，将会回复 `MOVED` 重定向。

## 容错性

### 心跳

Redis 集群内的节点通过 PING 和 PONG 消息交换数据，称为心跳。PING 和 PONG 的消息是同样的数据结构，都携带了重要的配置信息，这个数据结构称为心跳包，心跳包包括基本的头信息和 gossip 块信息，基本头信息包括的内容如下：

* Node ID；
* `currentEpoch` 和 `configEpoch`；
* Node 的标记，可以表示节点是 master 或者 slave
* 节点提供服务的 slot 位图信息；
* 节点的 IP 和端口，客户端可以用来连接发起 `GET` 或者 `SET`等命令；
* 节点所记录的集群状态，DOWN 还是 UP；
* 如果是 slave 节点，还包括它的 master 节点的 node id；

gossip 块信息记录的是该节点记下的其他节点信息，用来传播其他节点的配置，包括每个节点的：

- Node ID；
- 节点的 IP 和端口，客户端可以用来连接发起 `GET` 或者 `SET`等命令；
- Node 的标记；

### 故障检测

集群节点收到 PING 消息后，需要检测 PING 消息的心跳包的配置，然后回复 PONG 消息。

集群内所有的节点都会随机的向其他节点每秒发送 PING 心跳包，而且，如果一个节点与其他节点处于失联状态，该节点就应该要向这些失联节点发送 PING 心跳消息包。失联的意思是一个节点超过了半个  `NODE_TIMEOUT` 时间没有收到另一个节点发送 PING 消息或者没有收到 PONG 回复消息，无法确定节点是否还活着。

如果节点 A 认为节点 B 快失联 `NODE_TIMEOUT` 了，节点 A 会跟其他节点发送心跳，确定到底是记录因为网络分区被隔离还是节点 B 被隔离。如果节点 A 与其他节点可以正常通信，与节点 B 依然无法通信，此时节点 A 会重新建立与节点 B 的连接，确定是不是 TCP 连接坏死。

如果节点 B 失联时长超过了  `NODE_TIMEOUT`  ，此时节点 A 需要标记节点 B 为 `PFAIL` 状态。这里需要引入两个状态术语，`PFAIL` 和 `FAIL`。类似 Redis 的哨兵模式的 `SDOWN` 和 `ODOWN`，即主观下线和客观下线。

* **PFAIL**，master 和 slave 都可以标记一个节点失联时长超过 `NODE_TIMEOUT` 的状态为 `PFAIL`，这仅仅是节点自己认为的节点失联，是主观判定，还未得到集群内其他节点的认可，不会触发 slave 晋升操作；
* **FAIL**，`PFAIL` 提升到 `FAIL` 时，触发 slave 晋升操作，执行故障转移；

节点 B 的状态从 `PFAIL` 提升到 `FAIL` 的条件是：

* 节点 A 标记节点 B 为 `PFAIL`；
* 节点 A 通过 Gossip 协议交换数据收集到集群内超过半数的 master 节点对节点 B 的标记；
* 超过半数 master 节点标记节点 B 为 `PFAIL` 或者 `FAIL` 的时长在 `NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT` 时间内（`FAIL_REPORT_VALIDITY_MULT` 一般是 2）；

如果上面的条件全部满足，节点 A：

* 标记节点 B 为 `FAIL`；
* 将 `FAIL` 的消息发送给所有其他可达的节点；

`FAIL` 消息强制要求其他节点也对节点 B 标记为 `FAIL` 状态，而不管这些节点是否对节点 B 标记了 `PFAIL`。`PFAIL` 可以提升到 `FAIL`，但是 `FAIL` 不能降级为 `PFAIL`，`FAIL` 和 `PFAIL` 都会因为恢复而被清除。如果晋升长时间未执行，`FAIL` 状态也会清除，多长时间呢，是 N 倍的 NODE_TIMEOUT，N 是集群内 master 节点数。

一个节点标记失联节点为 `FAIL`，通过心跳消息，集群内的其他节点也会迅速标记失联节点是 `FAIL`，这称为*链式反应*。`FAIL` 状态的节点需要执行故障转移，slave 晋升为 master，使集群继续稳定运行。

### 故障转移

故障转移涉及到的问题：

* 如何选择晋升的 slave；
* 谁来执行晋升操作；

下面一一解决这些问题。

执行晋升操作是由 slave 节点来执行的，因为一个 master 节点可能有多个 slave，因此需要有序控制谁来执行晋升，如果第一个 slave 晋升失败，第二个 slave 接着继续执行晋升操作。

当一个 master 被标记为 `FAIL`，它所有的 slave 节点按照公式：

```text
DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds +
        SLAVE_RANK * 1000 milliseconds.
```

的等待延迟时间来执行晋升操作。其中 `SLAVE_RANK` 由复制 master 的 offset 决定，与offset 最接近的 slave，该值是 0，次接近的 slave，该值是 1，以此类推。

故障转移的流程如下。

第一步，申请晋升机会。当 slave 等待了 `DELAY` 时长后，它需要获取其他 master 的投票，申请晋升机会。

第二步，晋升投票。slave 需要通过投票才能获得执行晋升的机会，slave 需要获得超过半数的 master 投票赞成才能赢得选举，投票请求是 `FAILOVER_AUTH_REQUEST`。slave 获得了本次晋升机会，它需要增加 `configEpoch` 的值，该值需要比集群内所有的 `configEpoch` 都要大。另外，slave 还需要增加 `currentEpoch`。投票的规则是：

* master 只投给 epoch 比自己大的 slave 投票：master 使用 `lastVoteEpoch` 字段记录上次投票的 `currentEpoch`，只有当 `FAILOVER_AUTH_REQUEST` 请求的 `currentEpoch` 被此 master 记录的 `lastVoteEpoch` 大时才投赞成票；
* master 只投给它自己认为该 slave 的 master 确实是 `FAIL` 状态，否则不投票；
* `FAILOVER_AUTH_REQUEST` 的 `currentEpoch` 必须比此 master 的 `currentEpoch` 大才投赞成票；
* master 不会在 `NODE_TIMEOUT * 2` 时间内重复投给同一个 master 的多个 slave 发起的投票；
* master 不会投该 slave 的 `configEpoch` 在此 master 的 slot table 记录的 `configEpoch` 小的请求；

第三步，执行晋升。如果 slave 收到集群内超过半数的 master 赞成票，该 slave 晋升为 master 节点。它需要将配置信息传播到其他节点中。由于 slave 已经更新了配置的版本 `configEpoch`，其他节点会更新 slot 表，记录 slot 现在是哪个节点负责。

### 副本迁移

在一个 master-slave 模式下的集群节点中，如图-4所示，要保证集群可用性，必须要保证 master 节点至少有一个 slave 节点存在。节点 A 目前有两个 slave 节点，假设节点 B 停止服务，节点 B 的 slave 晋升，我们这里称“新节点 B”。假如“新节点 B”也停止服务，此时“新节点 B”没有 slave 可用，整个集群面临不可用的风险。

副本迁移可以实现将节点 A 的一个 slave 挂在“新节点 B”下，此时“新节点 B”就有一个 slave，“新节点 B”停止服务时，slave 可以顶上。

首先定义 *good slave*，*good slave* 是指 slave 没有处于 `FAIL` 状态，注意，`FAIL` 状态是集群内多个节点认为的客观状态，它表示 slave 已经真的停止服务。

副本迁移的原理是：

* 发现 master 没有 good slave；
* 从 slave 数量最多的 master 中找 slave 的 Node ID 最新的 slave；
* 迁移到没有 good slave 的节点；

选择 slave 和迁移的过程并不需要共识算法来完成。

## 配置更新策略

### hash slot table

Redis 集群将 key space 划分为 16384 个 slot，这称为 hash slot，集群内的每个 master 节点负责服务一部分 hash slot 。一个 master 节点需要记录 key space 被哪些节点维护，这样才能做到：1）重定向；2）重平衡，这里使用的是 hash slot table 来维护。结构大致如下：

```text
0 -> NULL
1 -> NULL
2 -> NULL
...
16383 -> NULL
```

上面的 hash slot table 是节点刚加入集群时的配置，还不知道哪些 slot 被哪些节点提供服务。只要它收到心跳消息，就可以更新这张表，因为心跳消息有节点维护的 slot bitmap。假设节点 A 服务 slot 1 和 slot 2，configEpoch 是 3，则表更新为：

```text
0 -> NULL
1 -> A[3]
2 -> A[3]
...
16383 -> NULL
```

如果下次收到心跳消息表示 slot 2 是节点比维护且 configEpoch 是 4，则更新表为：

```text
0 -> NULL
1 -> A[3]
2 -> B[4]
...
16383 -> NULL
```

更新表的规则有两个：

* slot 的服务节点信息是 NULL 时，直接拿心跳消息更新；
* `configEpoch` 大的配置取代 `configEpoch` 小的配置；

触发更新 hash slot table 的消息有：

* 心跳消息包；
* UPDATE 消息；

其中，UPDATE 消息是在心跳消息接收方收到过时的配置时（也就是自己的 `configEpoch` 比心跳消息中的 `configEpoch` 大时），需要向心跳消息发送方发生 UPDATE 消息，让它更新自己的 hash slot table。

### configEpoch 冲突

Redis 集群有一个特性，就是**所有的 master 节点的 configEpoch 不能相同**。

由于 Redis 集群支持人工操作 slave 晋升和数据迁移，这些操作都不需要共识算法来保证 *configEpoch* 是否冲突。

考虑一个场景，如果人工操作 slave 晋升的同时，出现了集群自动故障转移。这两个操作都会更新各自的 `configEpoch`，如果 `configEpoch` 相同，但是 hash slot table 不同，此时会出问题。

冲突的解决办法是：

* 如果一个 master 发现其他节点的 `configEpoch` 相同；
* 冲突的 master 中最小的 Node ID；
* 增加它的 `currentEpoch`，把它赋值给 `configEpoch`；

# 总结

## epoch

Redis 多次使用了 epoch 的概念来实现选举或者 `quorum` 算法，思想来自 Raft 的 Term 。每次 Redis 需要选举，都会使用一个更大的 epoch 来表示提议的版本。当选举成功后需要推送新配置时依然使用 epoch 来表示配置的版本，最有大的 epoch 才能替代小的 epoch 配置。

在 Redis Sentinel 和 Redis Cluster 的共识算法中，都使用 epoch，这些 epoch 是：

* currentEpoch
* configEpoch
* leaderEpoch

其中 currentEpoch 表示事件的版本，或者事件发生的时间，这个时间是逻辑时间，只能一直往上涨。slave 晋升投票是一个事件，使用 currentEpoch 来表示。而 Redis Cluster 的 configEpoch 表示的是配置的版本，注意：Redis 集群有一个特性，就是**所有的 master 节点的 configEpoch 不能相同**。没错 slot 发生变化，都需要更新 configEpoch，比如在 slave 晋升为 master 时，slot 的服务节点发生变化，此时需要更新 configEpoch，该值传播出去后，其他节点就可以更新了。

leaderEpoch 时 Redis Sentinel 的概念，当执行 slave 晋升操作时，需要使用共识算法来选择到底是由哪个 Sentinel 节点来执行晋升操作，被选中的 Sentinel 节点需要更新它的 leaderEpoch。

## 选举活跃性

Redis Sentinel 选举时，为了解决活跃性问题，每次由一个 Redis Sentinel 改变角色，变成选举的候选者，其他 Sentinel 进行投票。多个 Sentinel 需要使用一个随机的时间错开作为候选者进行选举的时间，减少同时由多个 Sentinel 发起选举。单一个 Sentinel 选举超时时，由其他 Sentinel 继续发起选举流程。

Redis Cluster 的 slave 晋升时，需要根据 `DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds + SLAVE_RANK * 1000 milliseconds` 错开时间，避免所有的 slave 同时发起晋升投票。

## 故障检测和故障转移

Redis Sentinel 和 Redis Cluster 都使用了一种叫做主观下线和客观下线的观念。对于 Sentinel 就是 `SDOWN` 和 `ODOWN`，对于 Cluster 就是 `PFAIL` 和 `FAIL`。处于主观下线时，可能是节点自己出现网络分区，所以还需要跟集群内其他节点通信，确定是不是自己的问题。当出现了客观下线后，才需要执行故障转移操作。

Redis Sentinel 和 Redis Cluster 都使用了类似 Raft 的共识算法来执行故障转移操作。

## 分片与一致性 hash

我们知道一致性 hash 算法解决的问题是降低数据迁移时移动的数据，Redis Cluster 使用不是一致性 hash 算法，而是一种 hash 取模的算法，这种算法与一致性 hash 算法非常接近，达到的效果也差不多。

一致性 hash 和 Redis Cluster 分片算法相似之处有：

* 都有一个有限的 key space；
* key space 划分 n 份；
* 每个节点领取一部分服务的 key space，直到全部领取完；

## 最终一致性

Redis 的异步复制造成 Redis 属于最终一致性，如果项目要求强一直都的 NoSQL，则 Redis 就不能满足。Redis 本身的持久化机制也是定期刷盘，不能保证不丢数据。当然如果拿 Redis 当做存储，其性能将大打折扣。



