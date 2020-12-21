---
title: 大数据系列——分布式文件系统GFS
date: 2020-11-10 15:35:44
tags: 大数据
---

# 简介

GFS使用在可扩展的分布式数据密集性应用，包括 OLAP 类型的海量数据存储数据库 Bigtable。GFS 提供以下特性：

* 可根据业务量灵活扩展计算和存储资源
* 多副本保障系统高可用
* 文件原子写入

# 设计

## 架构

GFS由 Master、Chunk Server和 Application 三个组件组成，其中：

* Master 协调整个 GFS 的读写和副本复制；
* Chunk Server 实际存储文件数据；
* Application 是 GFS 的客户端 SDK；

这三个组件逻辑架构如图-1所示。

![GFS架构](/images/gfs_arch.png "GFS架构")

图-1 GFS 架构

现在分别以读和写的方式说明三个组件的交互过程。

** 读 **

* 向 Master 查文件的 chunk 索引和 chunk 位置信息，位置信息包括哪些 chunk server 存了这些 chunk 的数据
* Application 拿到chunk 索引和位置信息后缓存在本地（可减少访问 Master 的次数），然后从 Chunk Server 读 Chunk 的数据

** 写 **

* Application 向 Master 查文件的 Lease （没有时需要分配一个 Lease 给一个 Chunk Server）在哪台 Chunk Server 上，Master 返回 Lease 的 Chunk Server （称为 primary） 和两个副本的 Chunk Server（称为 secondary）
* Application 向最近的 primary 和 secondary 节点推送数据，数据是文件的内容，primary 和 secondary 将数据先存在 LRU 缓存上
* 数据推完后，向 primary 发送一个或者多个写 chunk 请求，把刚才推送的数据写到一个或者多个 chunk
* primary 按照接收的顺序写完后，把相同顺序的写请求发送到 secondary，让 secondary 写 chunk
* secondary 写完后，回复给 primary
* primary 回复给 Application，通知写成功

## 一个 Master

GFS 使用一个 master 节点协调整个 GFS 系统的工作，优点有：

* 容易协调读写和副本复制
* 实现简单

缺点有：

* 高可用性降低
* 不支持存储太多数量的文件，因为元数据都存在内存，数量多时占内存且文件查询的效率也会降低

当 Application 向 Master 查文件的 Chunk 信息时，先 GFS Client 自己会根据文件名和 IO 操作的字节偏移地址转换成 Chunk index，GFS Client 通过文件名和 Chunk index 向 Master 查文件的元数据。GFS Client 会将查到的元数据缓存在本地进程中，避免下次再访问 Master，这样可以降低 Master 的负载，也能减少网络传输延时。

## Chunk 大小

Chunk 的固定大小是 64MB，优点是：

* 一次网络连接的建立，可以读取 64MB，读取多个 Chunk 时需要多次网络连接，当 Chunk 越大，读取的 Chunk 数量就越少，RTT 也会降低
* Chunk 数量就越少，读取 Chunk 的网络连接数量也减少，网络连接的负载就会降低
* 每个 Chunk 都在 Master 占用一定的内存空间，当 Chunk 越大，存储 Chunk 元数据的数量就越少

缺点是热点问题，当读取一个大 Chunk 文件时，涉及到的磁盘读取和网络传输消耗很大，如果访问量大，流量全部压在一台机器会成为瓶颈，会降低机器性能。解决热点的问题是：

* Chunk 不宜太大，一般 64MB
* 提高复制因子，用多个副本提供服务，分担压力

## 元数据

Master 存储的元数据包括

* 文件和 chunk 名字空间
* 文件名到 chunk 的映射
* chunk 的副本位置

第一、二这两项不仅全部存在内存还需要存储到磁盘，第三项依靠副本上报、Master 只存内存。

### 内存数据结构

元数据放在内存的优点有：

* Master 访问操作速度快
* 后期使用垃圾收集和重复制（re-replication）的周期检查机制也很高效

缺点有：

* 元数据内容受到内存大小限制

### Chunk 位置

Master 不会将 chunk 的副本位置信息写入磁盘，master 只会在 chunk server 加入机器时询问 chunk 副本位置信息。chunk 副本不存磁盘的原因是：

* chunk server 周期上报 chunk 副本信息非常简单
* 相比 master 存 chunk 副本而言，可以减少当 chunk server 加入集群、离开机器、修改名字、失败、重启等等信息同步的开销
* 哪个 chunk 存在哪台 chunk server，chunk server 有最终话语权

### 操作日志

GFS 的操作日志记录了对所有文件的修改操作，记录这些操作日志的作用有：

* 提高可用性，异常重启可以根据操作日志和检查点重建内存数据
* 提高数据一致性，

操作日志的实现原理如图-2所示。

![GFS操作日志和检查点](/images/gfs_operation_log.png "操作日志和检查点")

需要注意的点：

* 操作日志非常重要，（异常）重启服务时需要重放日志，构建内存元数据
* 回复客户端写成功前，Master 必须已经将操作日志写入磁盘并且成功同步到Master副本
* 每个日志文件的大小有限制
* 当日志文件数量多时，重启服务重放日志会非常耗时，因此需要将日志合并成一个检查点文件
* 检查点文件需要一个新的后端线程完成
* 当检查点文件生成后，可以将这些日志文件删除，未合并的日志文件继续保留
* Master 可以批量将日志队列的数据写入到磁盘或者同步到Master副本，提高吞吐量

## 一致性模型

GFS 定义的文件区域（file region）一致性模型有4种：

* consistent 一致
* defined 明确
* undefined 不明确
* inconsistent 不一致

一致的意思是说多个客户端读同一个文件时内容一致；明确的意思是说读到的内容知道是哪次写入的。

GFS支持两种写入方式：

* Write 应用程序自己指定文件的 offset 写入内容
* Record Append 由 GFS 选出内容追加的 offset ，然后再写入内容

这两种写入会影响一致性模型，见表-1所示。

表-1 不同写模式影响一致性模型

|          | Write      | Record Append    |
| -------- | ---------- | ---------------- |
| 顺序成功 | 明确       | 明确并伴有不一致 |
| 并发成功 | 一致不明确 | 明确并伴有不一致 |
| 失败     | 不一致     | 不一致           |

并发写成功造成一致不明确的原因是：并发写成功完成后，所有客户端都能读到相同的内容，但是不知道这个内容是谁写入的。因为并发写入会造成一个客户端写的内容全部或者部分覆盖另一个应用写的内容，当一个应用写成功后，可能它写的内容因被覆盖而丢失。

GFS 将一个 chunk（64MB）划分成多个文件区域（64KB），每一个文件区域都有一个 32 位的检验和，用来判断文件区域是否有效。检验和跟其他元数据一样，写入到日志文件并且存储在内存中。如果应用读取的 chunk 包含无效的文件区域，GFS 并不会返回这些内容，而是继续检查下一个文件区域。

### GFS 保证的一致性和完整性

下面说明两种写文件区域失败的情况，看看 GFS 是如何处理数据一致性和完整性的。

* primary （拥有写凭证的副本）失败

![primary 写失败](/images/gfs_file_region_primary_failure.png "primary 写失败")

* secondary 失败

当 secondary 失败，意味者 primary 写成功，primary 的文件指针增加了。当客户端重试，primary 继续接着上次的文件指针位置（或者填充空白来结果对齐）继续写，写成功后，secondary 需要从 primary 的 offset 处继续写，之前的内容同样要填充空白。此时，对于 primary 来说，内容重复。



通过上面的分析发现 GFS 解决数据数据的一致性和完整性时，会留下两个问题：

* 空白填充剂
* 文件区域的数据重复

存储填充剂和重复记录的文件区域，其检验和也是正确的，对于 GFS 服务端而言，它也是正确的数据。那么这些多余的数据该如何鉴别并且丢弃呢？这要留给客户端处理。

### 客户端的隐式处理

客户端写入数据时按照文件区域大小对齐，且每个文件区域包括：

* 检验和：检查并丢弃空白填充剂
* 文件区域ID：可以过滤重复记录

客户端也可以自己维护一个文件的检查点，记录文件写入多少数据。



GFS Client API 已经集成这些功能，因为它比较通用。

# 系统交互

## 租约和写顺序

由于修改操作会影响 chunk 内容或者元数据，为了保持数据一致性，客户端写数据前必须向 master 申请写凭证（这里成为 Lease 租约）。master 会保证任意时刻，一个 chunk 至多只有一个租约有效。拥有租约的 chunk server 称为 primary，primary 的副本（可能存在一个或者多个副本）称为 secondary。一个文件的全局写顺序由 master 分配租约的顺序来保证，而一个 chunk 并发写顺序由 primary 来保证。



master 如何保证租约分配顺序？步骤如下：

* 1 master 接收到租约申请，检查 chunk 是否有存活的租约；
* 2 如果没有租约，新分配一个，找出一个副本，将其作为 primary，同时将 chunk version number （CVN）自增；
* 3 如果有租约，但是拥有租约的 primary 节点失联，master 需要等待租约过期，过期时间是 60 秒；
* 4 如果有存活的租约，直接返回给客户端；
* 5 当 primary 的租约快过期时，向 master 续约；

通过 CVN 可以检测 Lease 是否过期，比如当前 primary 的租约的 CVN 是 5，当客户端请求写入时传的 CVN 是 6，可能是因为异常原因、或者客户端缓存导致当前 primary 没有拿到最新的租约，写入失败，需要客户端重新申请再写入。



primary 负责管理 chunk 及其副本的写入，维护数据写入的一致性。primary 如何保证写的顺序？步骤如下：

* 1 客户端向 master 申请租约成功后，master  返回 primary 和 primary 的副本节点 secondary；
* 2 客户端先将文件数据推送到 primary 和 secondary，他们会将数据存在内存中一块 LRU 缓存中；
* 3 客户端再将写请求发送到 primary，因为 primary 只有一个节点，数据接收的顺序可以作为它写的顺序；
* 4 primary 将相同的写顺序同步到 secondary ，secondary 也按照这样的顺序写 chunk；
* 5 primary 和 secondary 都写入成功后，回复客户端写成功；
* 6 任意一个节点写失败时，primary 都会通知客户端重试；客户端继续回到第3步重复操作；
* 通过上面的方式就可以保证 primary 和 secodnary 写入顺序也是一致的。

## 数据流



## 原子记录追加



## 快照



# Master 操作



## 命名空间管理和锁



## 副本位置



## 创建-重复制-重平衡



## 垃圾收集

### 机制



### 讨论

## 旧数据检测



# 容错性和诊断

## 高可用

### 快速恢复

### Chunk复制

### Master复制

## 数据完整性



## 诊断工具