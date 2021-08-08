---
title: 大数据系列——Bigtable
date: 2020-12-18 15:35:44
tags: 大数据
toc: true
---

# 简介

Bigtable<sup>[ [1] ](http://static.googleusercontent.com/media/research.google.com/en/us/archive/bigtable-osdi06.pdf)</sup> 是 Google 为了处理海量数据的离线计算和数据存储而开发的一套 KV 存储数据库，Bigtable 依赖 GFS 存储日志文件和数据文件。

# 数据模型

Bigtable 是 KV 存储结构，其中 K 的结构抽象成：

(row:string, column:string, time:int64) → string 

表示 K 由行键（row key）、列名（column key）和时间戳（timestamp）组成。V 表示实际存的内容，也是字节表示。Bigtable 是 row key，column key，timestamp 索引的一个映射表，由于 K 抽象成行键、列名和时间戳，KV 也可以表示成一个数据表，行键就是行号，V 是列的内容，time 是 V 的版本号。就像图-1所示。

![Bigtable 数据模型抽象](/images/bigtable_data_model_abstract.png "Bigtable 数据模型抽象")

*图-1 Bigtable 数据模型抽象*

## 行

Bigtable 的行键（row key）有任意字符串组成，内部转换成字节，最大 64 KB。转换成字节后，多个 row key 按照字节大小顺序排列，根据数值大小的范围分成多个 *tablet*（如图-1所示）。Bigtable 支持原子修改一个 row key 的内容，不管它有多少列。由于 tablet 的划分，读取相同或者相近的 tablet 速度快，因为数据空间局部性的关系，所以读取小范围的数据会非常高效。

总结一下 row key 的特点：

* 行范围组成的 *tablet*；
* 单行事务；
* 数据空间局部性特性；

## 列

Bigtable 的列（column key）由 *family:qualifier* 组成，多个 column key 组成一个集合，即列族。column key 的特点如下：

* 每一个表的 *family* 要预先定义，*qualifier* 不需要预先定义；
* *family* 是权限访问控制和内存管理的基本单位；
* 一个 *family* 存的数据格式都一样，不管有多少个 *qualifier*，这种特性对数据压缩非常有效；
* 一个表的 *family* 尽量少；
* *family* 的名称必须是可打印的字符串，而 *qualifier* 可以是任意字符串；

## 时间戳

Bigtable 使用时间戳（timestamp）来实现数据的多版本支持，timestamp 由 64 位组成的整数，一个数据的多个版本按照 timestamp 的大小降序排列，最大的 timestamp 将会最先读到。timestamp 的维护方法是：

* 可以由系统生成，按照毫秒数赋值；
* 可以是客户端自己生成，告诉 Bigtable；
* 版本数量管理需要定义，比如1）只保留最近 n 个版本；2）只保留最近几天的版本；
* Bigtable 的 gc 控制定期扫描数据，清除垃圾版本；

# 数据块

Bigtable 使用 GFS 持久存储数据，而 GFS 本身是分布式文件系统，当 Bigtable 写表文件数据时，数据会落盘到 GFS 文件系统，GFS 会通过副本机制提供高可用的底层文件存储能力。这样一来，Bigtable 就可以不用像 Mysql 那样再自己实现数据复制。

Bigtable 的 SSTable（Sorted String Table）结构用来在磁盘中存储 KV 数据，结构如图-2所示。

![SSTable 结构](/images/bigtable_sstable.png "SSTable 结构")

*图-2 SSTable 结构*

每个 SSTable 文件都包含一个或者多个块 block，每个块的大小是 64KB，块大小可以配置。每个块可以存储的 K 都是顺序排列，这样每个块就包含了 K 值范围。SSTable 文件末尾会存储这个文件所有的 block 索引，结构大概是：

```text
key1 -> block 1
key2 -> block 2
...
key m -> block p
key n -> block q
```

这个结构可以组成一个二分查找算法：

```text
key x  ------------+
                   | 判断 key x 是否 介于 key 1 和 key 2 之间，如果是，则数据应该存在 block 2
                   v
(-∞,key 1]   (key 1,key 2]  ...     (key m,key n]
    |              |                       |
    v              v                       v
 block 1         block 2                 block q
```



Bigtable 的 memtable 和 SSTable 是典型的 Log-Structure-Merge Tree <sup>[ [2](http://paperhub.s3.amazonaws.com/18e91eb4db2114a06ea614f0384f2784.pdf) ]</sup>（即 LSM 树）算法的应用，这种算法支持高效的写入操作，并发查询效率开销也不大。

Chubby 是 Google 开发的基于 Paxos <sup>[ [3](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) ]</sup>的一致性共识算法。Chubby 提供了以目录和文件组成的名字空间，用来使用分布式锁、存储 *tablet* 位置的服务，客户端直接连接 Chubby 服务。Bigtable 使用 Chubby 的场景有：

* 确保只有一个 Bigtable 的 master 节点提供服务；
* 存储 *tablet* 的位置，包括启动 Bigtable 服务所需的 METADATA 表；
* 发现 tablet server 并监听他们的状态；
* 存储 Bigtable 表映射的原书记，包括 column family 信息；
* 存储权限访问控制列表；

Bigtable 高度依赖 Chubby 服务，如果 Chubby 服务不可用，Bigtable 也不可用。

# 架构

Bigtable 保护三个组件：客户端，一个 master 和多个 tablet server。

架构图见图-3。

![Bigtable 架构](/images/bigtable_arch.png "Bigtable 架构")

*图-3 Bigtable 架构*

客户端直接连接 tablet server 写入和读取数据，通过读取 chubby 查询要读取的 *tablet* 由哪个 tablet server 负责。master 决定哪个 tablet server 服务哪个 *tablet*。在数据模型的行概念中说到，*tablet* 存储的是行范围数据（“each tablet contains all data associated with a row range”）。

Bigtable 存储一定数量的表，每个表由 *tablet* 集合组成，每个 *tablet* 包含一个或者多个行数据。一个表最开始只有一个 *tablet*，当数据量变大时，会分裂成多个 *tablet*，每个 *tablet* 大概是 100-200MB。

## master 节点

master 节点的工作职责包括：

* 分配 *tablet* 到 tablet server；
* 检查 tablet server 加入集群；
* 检查 tablet server 会话过期，移出集群；
* 平衡 tablet server 的负载，重新分配 *tablet*；
* 文件 gc；

## tablet server 节点

tablet server 节点的工作职责包括：

* 维护 *tablet* 集合；
* 服务数据读写请求；
* *tablet* 太大需要分裂；

## tablet 分配

一个 *tablet* 最多只能分配给一个 tablet server，master 负责从 tablet server 列表找出负载最低的 tablet server 然后分配该 *tablet* 。

tablet server 启动时需要向 chubby 申请独占锁，有两个作用：

* master 对 tablet server 存活检查的状态标志；
* 发现 tablet server；

当 tablet server 发现自己没有独占锁，它将停止 *tablet* 的读写请求，接着再次尝试获取独占锁，锁获取成功后等待 master 的 tablet 分配请求。

当 tablet server 退出时，它将自发的释放独占锁。

master 的检查机制如图-4所示。

![tablet server 存活检查](/images/bigtable_master_check_tablet_server.png "tablet server 存活检查")

*图-4 tablet server 存活检查*

注意：图-4中的第4步成功，第5步自然会失败，此时 tablet server 继续提供服务，master 继续执行 tablet server 的存活检查。

master 启动后分配 tablet server 见图-3，大概就是：

* master 向 chubby 获取独占锁；
* master 扫描 tablet server 注册目录，查找加入集群的 tablet server；
* master 向 tablet server 询问分配的 tablet；
* master 扫描 METADATA 表查询所有的 tablet；

注意：当 master 询问 tablet server 发现没有任何服务 root tablet，则 master 需要先找一个 tablet server 分配 root tablet 才能扫描 METADATA 表。

## 服务 tablet

Tablet server 处理 tablet 的读写请求如图-5所示。

![tablet 读写](/images/bigtable_tablet_representation.png "tablet 读写")

*图-5 tablet 读写*

对于读请求来说，tablet server 首先查 memtable，当 memtable 存储，直接返回。当 memtable 不存在，需要找 SSTable File n，SSTable File n-1，...，SSTable File 0，直到找到为止。此时要求删除操作不能把仅仅把 K 移除就完事，而是要记录 K 的值，表示这个 K 被删除。这个时候找到这个 K 取出它的值就知道其已经被删除，返回用户值不存在，而不是继续找下一个文件。

对于写操作来说，tablet server 首先写操作日志到 GFS，memtable 的数据全部都是已经写入 commit-log 日志文件。

## 压缩

当 memtable 不断的写入，内存会越来越大，当操作一定的界限，触发溢写操作。tablet server 停止当前 memtable 的写入，创建一个新的 memtable ，所有新的写操作全部写入到新的 memtable 。tablet server 后端线程将旧 memtable 写入到 GFS 的 SSTable 文件。这个过程称为 *minor compaction*，它有两个作用：

*  避免内存使用过大；
* 减少恢复时读 commit-log 的数量；

当 SSTable 文件不断增大时，tablet server 将多个 SSTable 合并，重复的 K 会合成一个 K，这也能有效的压缩数据，称为 *major compaction*。合并成功后，SSTable 文件就可以删除了。

合并 memtable 时最特殊的操作就是删除，前面已经说明。 

# 优化

## 局部化

建立 Bigtable 表时可以控制哪些 *column families* 属于哪个 *locality group*，每个 *locality group* 都是一个 SSTable 文。将不同的 *column family* 隔离可以提高读性能，比如活跃的列都放在一个 *locality group*，不活跃的列放到另一个 *locality group*。

## 压缩

客户端可以控制哪些 *locality group* 需要压缩，使用哪种压缩格式。一般来说，同一个 *locality group* 的数据都相同，压缩特性比较好，可以减少数据量。

## 缓存

Bigtable 支持两种缓存，Scan Cache 和 Block Cache。Scan Cache 缓存 SSTable 的行，而 Block Cache 缓存 SSTable 的块，前者是更高一级的缓存，后缀是底层的缓存。如果经常读取相同的行，Scan Cache 优势非常明显。如果经常读取同一范围的数据，Block Cache 优势非常明显。

## Bloom filters

前面说到当查一个 K 时，如果 memtable 不存在，需要找 SSTable File n，SSTable File n-1，...，SSTable File 0，直到找到为止。每找一个 SSTable 都需要打开文件，加载它的 block 索引，这种开销也非常大。可以使用 Bloom filter 来检查 K 是否存在 SSTable。Bloom filter 的算法意思大概是判断一个值，如果返回 false 表示该值肯定不存在，如果返回 true 表示该值可能存在。 

## commit-log 实现

有一个矛盾，如果每个 tablet 的写操作都走 commit-log，优点是恢复时读的数据少，缺点是需要写太多的文件到 GFS，写入性能降低。Bigtable 的做法是一个 tablet server 写一个 commit-log。

Commit-log 写入和恢复如图-6所示。

![Commit Log](/images/bigtable_commit_log.png "Commit Log")

*图-6 Commit Log*

Bigtable 有两个写日志线程，两个当中只有一个是活跃的线程，从 commit-log 队列读取数据，写入到 GFS 文件中。使用两个写入线程的原因是降低网络拥塞、抖动导致日志写入缓慢的情况，当活跃线程写入性能出现问题时，切换到新的写入线程，继续从 commit-log 队列读取数据写入到 GFS。每一个 table server 包含一个 commit-log 文件，图中的 commit-log 文件包含多个表的更改日志。当 tablet server 停机时，master 需要重新分配 tablet server 。如果分配给多个 tablet server，则多个 tablet server 都会去读完整的 commit-log 文件。为了提高读取性能，减少文件重复读取的情况，master 会将 commit-log 分区块并按照 <table,row name, log sequence number> 规则并行排序，每个区块 64 MB 且放在不同的 tablet server 中进行排序。

## 加速 tablet 恢复

核心问题是尽可能的在 tablet server 停止服务前完成 memtable 写入 SSTable 的操作，当 tablet 重新分配时就可以不用读取 commit-log 重放日志，而是直接加载 SSTable 到日志。

## 不变性

tablet server 的 SSTable 都是不可变的数据结构，读取时不需要加锁，并发量大。唯一可变的就是 memtable，Bigtable 对每一行都使用 Copy-on-Write 并行处理。

# 总结

MapReduce 、GFS 和 Bigtable 是 Google 大数据的产物，它们拉开了大数据时代。对应的开源实现是 Hadoop MapReduce，HDFS 和 HBase。在大数据应用中，他们常用来处理海量离线计算业务。

LSM 算法是 KV 存储的经典算法，适用于写入量大的存储应用。比如 LevelDB，RocksDB，Cassandra，TiDB 等都是用了 LSM 来提高写入性能。

由于 Bigtable 底层依赖 GFS ，所以它本身不适合用在高并发写入的场景。一般来说，tablet server 和 chunk server 尽量放在同一台机器上，此时读取可以不用跨网络，而是本地磁盘，因为数据都存在本地，只有本地磁盘没有数据时才需要由 GFS 跨网络读取。