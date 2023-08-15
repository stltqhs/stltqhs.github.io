---
title: Kafka
tags: mq
draft: true
date: 2020-12-23 11:06:13
toc: true
---


开源社区的消息队列工具非常多，包括 RabbitMQ，ActiveMQ，ZeroMQ和Kafka，其中 Kafka 因它强劲的性能倍受推崇。本文将讲述 kafka 高性能背后的原因。

# 存储架构

本节参考 [How Kafka’s Storage Internals Work](https://thehoard.blog/how-kafkas-storage-internals-work-3a29b02e026) 整理出来。

## Topic

Topic 就是队列，kafka 支持多个 topic，往 topic 写入消息的服务称为生产者，往 topic 消费消息的服务称为消费者。一个 topic 支持多个生产者和多个消费者，对于消费者的消费数据的场景会有所不同，下文将详细说明。

## Partition

partition 称为分区，一个 topic 会包含多个分区，Kafka 集群内的每个节点都会承担一部分分区，一般是平均。假设某个 topic 被设置成 3 分区，在一个三节点的 kafka 集群，每个节点承担一个分区。当一个生产者向一个 topic 写入数据时，根据 key 来决定写哪一个 partition，如果 key 没有给出，则按照 round-robin 的方式选择一个分区写入数据。写入的消息是按照顺序存放在磁盘的，如图-1所示。

![img](https://miro.medium.com/max/1400/1*9W02uviSfU_QSHjaNTnNXQ.png)

*图-1 生产者写入数据*

如果数据一直写入，分区的文件就会越来越大，需要想办法删除过期数据。

## Segment

为了防止数据膨胀，Partition 需要再切分为 segment，也就是说，一个 partition 是由多个 segment 组成的。当向一个分区写入数据时，kafka 会判断当前分区的 segment 是否超过限定大小（比如限制每个 segment 文件不能超过 500MB），如果超过时，kafka 会为该分区创建一个新的 segment 文件，新消息就往这个 segment 文件写入。

一般地，segment 文件名以文件内第一个消息的 offset 来命名。offset 是消息的位置，比如第 0 号消息的 offset 就是 0，每新写入一个消息，offset 就需要自增。

![img](https://miro.medium.com/max/1400/1*bZ-fWeb2KG_KhYv2EKDvhA.png)

*图-2 Parition的数据逻辑结构*

图-2表示当前有 3 个 segment 文件，已经用不同颜色来标记，segment 为 0 的文件包括 0，1和2号消息。当前最后一个 segment 文件 segment 6 包含 3 个消息，已经写满，当来了 9 号消息时，需要新建一个 segment 文件，文件名包括 9 这个 offset。

在磁盘中，一个 partition 就是一个目录，每一个 segment 就是由一个索引文件和日志文件组成。

```sh
$ tree kafka | head -n 6
kafka
├── events-1
│ ├── 00000000003064504069.index
│ ├── 00000000003064504069.log
│ ├── 00000000003065011416.index
│ ├── 00000000003065011416.log
```

日志文件也就是数据文件，存放消息的文件。一个消息包括 offset，key，value，写入时间戳，消息大小，压缩算法名，检验和（checksum）和消息的版本。

磁盘中的数据的格式与生产者发送的格式一直，而且消费者消费到该消息时，数据格式也是与生产者写入的格式一致。这种机制可以充分利用 [zero copy](https://developer.ibm.com/articles/j-zerocopy/) 技术，减少消息在内核空间和数据空间复制的成本。

我们用 `kafka-run-class.sh` 脚本执行 `kafka.tools.DumpLogSegments` 读取一个日志文件，通过打印的数据能看到消息的结构。

```sh
$ bin/kafka-run-class.sh kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files /data/kafka/events-1/00000000003065011416.log | head -n 4
Dumping /data/kafka/appusers-1/00000000003065011416.log
Starting offset: 3065011416
offset: 3065011416 position: 0 isvalid: true payloadsize: 2820 magic: 1 compresscodec: NoCompressionCodec crc: 811055132 payload: {"name": "Travis", msg: "Hey, what's up?"}
offset: 3065011417 position: 1779 isvalid: true payloadsize: 2244 magic: 1 compresscodec: NoCompressionCodec crc: 151590202 payload: {"name": "Wale", msg: "Starving."}
```

索引文件用来映射 offset 到日志文件的偏移量。由于 kafka 支持从指定的 offset 开始消费数据，当找到数据所在的 segment 文件后，还需要知道从文件的哪个位置开始读取数据。索引文件便是辅助快速查找日志文件的功能。如-3所示。

![img](https://miro.medium.com/max/1400/1*EkswTKX292hDl921ktyq-w.png)

*图-3 索引文件映射*

由于索引文件很小，完全可以载入内存，当查找一个 offset 的文件偏移量时，可以使用二分查找，快速定位。

索引文件由 8 字节组成，4 字节用来存储相对于 segment 文件的第一个 offset 的值，这样就可以从 0 开始，而且用 4 字节也足够。例如，加入 segment 文件的第一个 offset 的值是 10000000000000000000 ，接下来的 offset 是 10000000000000000001 和 10000000000000000002 ，那么索引文件存储时就是 0，1和2。

需要特别指出：segment 文件的 offset 小于等于该文件第一个消息的 offset。

## 生产者和消费者

一个 topic，不管有多少个分区，可以设置任意个生产者写入数据。

一个 topic，一个分区只能被一个消费者消费，其他消费者就不能消费这个分区的数据，除非之前的消费者关闭。

每个分区都有一个 leader 负责读写数据。

## 高性能

站在存储架构的角度来看，kafka 高性能体现在：

* 消息的顺序写入和顺序消费
* Zero Copy 技术的应用；

由于这个特性，Kafka 被大量应用在大数据场景。而在事务型业务中，一般消费者每次处理一条消息就做一次同步提交，相等于是一个 2pc 操作，kafka 的高性能优势便不明显。

要想利用 kafka 的高性能，你需要：

* 数据批量写入；
* 数据批量消费；
* 多分区；
* 一个分区、一个消费者、多个工作线程；

# 集群

本节参考 [Kafka Replication](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Replication) 整理出来，本节所说的复制算法，是 kafka 早期的复制算法，存在数据丢失和消息乱序的问题，具体下文会说到。

## 消息复制

存储系统的高可用离不开多副本，Kafka 集群需要使用多副本复制功能。Kafka 以 partition 为单位进行多节点复制，每一个 partition 由一个 leader 负责写入，多个 follower 节点负责同步 leader 的数据。

先来说说 *primary-backup* 和 *quorum-based* 两种数据复制策略，他们都需要先分配一个 leader 来接受用户写入的数据，然后 leader 将数据传播给其他节点 follower 节点。但是传播的方式有区别。使用该算法的有 Zab 和 Raft 算法。

以 *primary-backup* 为复制策略的系统，leader 需要等到数据全部复制给 follower 节点才能返回用户 commit 消息，如果一个节点 down 掉，leader 继续尝试下一个节点。在该策略下，如果有 f 个副本节点，系统最多可以容忍 f-1 个副本节点 down 掉。

以 *quorum-based* 为复制策略的系统，leader 需要等到数据复制给大多数 follower 节点才能返回用户 commit 消息。在该策略下，如果有 2f + 1 个副本节点，系统最多可以容忍 f 个副本节点 down 掉。比如 Dynamo 就是使用这种策略，它以写多数 W 和读多数 R 来达到数据一致性，如果集群节点为 N ，这就要求 W + R > N ，也就是说写入的节点与读取的节点的集合有重合，才能保证写入的数据能够被读取到。

两种策略各有优缺点，我们来比较一下：

*  *quorum-based* 延迟性比  *primary-backup* 低；
* 在相同副本节点数量的情况下，  *primary-backup* 可以容忍更多的并发冲突；
* 如果复制因子是 2， *primary-backup* 可以容忍一个节点 down 机，而  *quorum-based*  要求两个节点必须都是正常的；

Kafka 使用的是  *primary-backup* 复制策略。

复制也可以分为同步复制和异步复制，我们先来谈谈同步复制。

每个分区都有 n 个副本节点，其中一个被选为 leader ，负责数据写入和状态维护，其余的节点称为 follower，负责复制消息。leader 维护一个 ISR 列表，ISR 是 in-sync replicas 的缩写，意思是需要同步数据的节点列表，列表里的节点需要追上 leader 节点的数据。kafka 将每个 partition 的 leader 和 ISR 存储在 Zookeeper 中。

每个副本节点维护自己本地的日志文件和一些重要的偏移量信息，比如日志文件的 offset。LEO （log end offset）表示日志文件的末尾或者最后一个 offset。HW（high watermark）表示最后一个已提交消息的 offset。每个消息日志都是周期性的刷到磁盘，flush offset 表示刷盘时的消息日志偏移量，如果一个 n 小于或者等于 flush offset ，就表示 n 和之前的数据都已经刷到磁盘。flush offset 可能大于 HW ，也可能小于 HW，因为 kafka 支持设置为只要数据到内存，就可以提交消息，而不是刷盘，这样方式会降低数据一致性，但是性能会提高很多。

![kafka_replication_v1_lost](/images/kafka_replication_v1_lost.png)

*图-4 Kafka LEO 和 HW 示意图*

### 写

当一个客户端需要向一个 topic 写入数据时，首先需要重 Zookeeper 找到该 topic 的一个 partition 的 leader，找到 leader 后，就可以向 leader 发布消息。leader 将接收的消息写入到本地日志文件，follower 节点会通过一个 socket 通道迅速从 leader 节点拉到消息，用这种方式，follower 可以从 leader 拉到所有消息，而且是跟 leader 相同顺序接收并存放到本地日志文件。follower 每拉到一个消息，就存放在本地日志文件，然后向 leader 回复一个 ACK，表示消息已经收到。如果 leader 收到所有 ISR 节点对一个消息的 ACK，就表示这个消息已提交。之后，leader 需要将 HW 设置为当前提交消息的 offset，然后回复用户消息已提交。leader 还需要将 HW 信息同步到 follower 节点，有两种方式可以完成，第一个是广播消息，第二个是 follower 拉消息时就可以直接返回 leader 当前的 HW。每个副本节点收到 HW 后，还需要将 HW 存储在磁盘，记录一下检查点（checkpoint）。

### 读

读也是从 leader 读取数据，leader 会返回 HW 之前（包含）的数据给用，HW之后的数据不会给，因为这些数据还没有提交。

### 异常场景

#### follower 异常

当一个副本节点超过了 `replica.lag.time.max.ms` 和 `replica.lag.max.messages`（高版本已不支持）设置的阈值，leader 会将这个节点从 ISR 剔除，消息会继续往剩下的 ISR 节点同步。如果失败的副本节点恢复，它首先将自己的日志截断到它自己上一次 HW 检查点的位置，然后开始从 leader 拉数据，追上 leader 的数据。当这个节点完全追上了 leader 节点，leader 节点重新将它加入到 ISR 列表中。

#### leader 异常

leader  失败有 3 种场景：

* S1 写入消息到本地日志文件之前就 crash，用户将会 timeout 让后再次向新的 leader 发送消息；
* S2 leader 在写入本地日志文件之后，回复用户 commit 之前 crash，此时有可以分两种场景；
  * 需要保证原子性，要么所有的副本节点都保存消息，要么都没有保存；
  * 用户重发消息，这种情况下，系统需要保证消息没有两次写入。因为有可能是有个副本节点已经写入了消息，并且已经提交，而之后又被选举为 leader；
* S3 leader 回复用户 commit 后 crash，新的 leader 会被选举出来，然后继续接收新的请求；

为了解决上面这 3 种异常场景，kafka 的 leader 选举时需要：

* P1 所有 ISR 存活的副本节点都需要在 Zookeeper 中注册；
* P2 第一个注册成功的副本节点称为新的 leader，新的 leader 将自己的 LEO 作为 HW；
* P3 每个副本节点都需要在 Zookeeper 中注册监听器，以便知道系统新的 leader 是谁。每当副本节点收到一个新 leader 的事件，副本节点就需要：
  * P3-a 如果该副本节点不是 leader，它就是一个 follower，这时需要将它本地的日志截断到 HW ，也就是 HW 之后的数据都要删除掉，然后在启动复制去追新 leader 的数据；
* P4 新 leader 需要等到所有存活的 ISR 节点都追上自己或者指定的时间已经过去，新 leader 就将当前的 ISR 写入到 Zookeeper，然后开始接收读写请求；

#### 数据丢失

对于失败场景 S1 来说，消息根本就没有写入成本，旧 leader 和副本节点都没有写入成功，用户收到失败或者超时信息，是属于正常情况，不会造成数据不一致的情况。

对于失败场景 S2 来说，用户同样是没有收到确认的消息，这时对于用户来说，消息可能保存成功，也可能保存不成功，所以无论新 leader 是否包含这个消息都是正确的，也不会造成数据不一致的情况。这种场景在生活当中也是常见的，比如当我们发起一笔付款后，如果出现超时（可能是手机没有信号），自己是不知道到底是支付成功还是失败，只有在异常恢复时去查看确认一下才知道。对于 kafka 来说，没有提供一个确认方法，只能在消费时，才能反映是成功还是失败了。这种情况，为了自己程序的健壮性，当然是需要重发，然后消费端自己处理幂等性。

对于失败场景 S3 来说，用户已经收到 commit 消息了，那么系统就要保证系统真的 commit ，新的 leader 也需要包含这个消息。由于 P2 的缘故，新选举成功的 leader 肯定已经包含了这个 commit 的消息，所以也不会出现数据丢失的情况。

考虑一下这个场景：*如图-4所说，如果 slave1 在 master down 机后也 down 机了，此时 slave 1 的数据会被截断到 HW，但是 down 机后有马上恢复，并且选举为新的 leader ，此时它的数据就没那么多了，出现了数据丢失的情况。事实上，此时它根本就不具备选举的的条件，因为它失败后还需要找 leader，肯定不会是自己。*

余春龙在他的著作 [《软件架构设计：大型网站技术架构与业务架构融合之道》](https://book.douban.com/subject/30443578/) 一书中也说到数据复制会丢失，他描述的场景主要是在 P2 规则的不同。

#### 异步复制

要支持异步复制，leader 接收到消息之后，直接写入本地后就回复用户消息已提交。这种情况下，数据就不是最终一致性，可能会丢掉数据。因为已经不保证副本节点可以追得上 leader ，当 leader cash 后，新选举的 leader 消息会落后，然后数据就会被截断。



0.8 之后的复制算法见 [kafka Detailed Replication Design V3](https://cwiki.apache.org/confluence/display/KAFKA/kafka+Detailed+Replication+Design+V3) 算法来完成复制功能。

# 其他

如果需要深入了解 Kafka 源码可以参考 [kafka 源码解析系列](https://matt33.com/tags/kafka/)。