---
title: 大数据系列——Percolator
date: 2020-12-21 09:22:29
tags: 大数据
toc: true
---

本文是笔者阅读 [《Large-scale Incremental Processing Using Distributed Transactions and Notifications》](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf) 论文整理而得。

# 摘要

更新一个不断爬取不断累加的搜索引擎的索引文档集合的任务，是数据处理任务的一种常见场景，数据处理通过计算将一个个很小的页面转换为索引，当页面发生更改时，索引也需要更新。任务介于现在的基础设施之间：存储和计算。数据库已经不能存放 10 PB 级别的数据，也不能在几千台机器中处理每天 10 亿更新量。MapReduce 和其他批处理系统不能处理仅有小部分增量更新的数据，因为他们适用于更大规模的数据量，小更新效率不高。

Google 研发了 Percolator，可以处理大数据集里的增量更新，利用它来构建 Google Web 搜索的索引。使用 Percolator 替换批量数据处理后，相同的数据量，可以将文档平均年龄降低 50%。文档年龄（age of document）的大概意思是指文档从被爬虫抓取下来到能够出现在搜索结果的时长。

# 介绍

Google 搜索引擎爬取海量的 web 文档，这些文档会被构建成一个分布式的反向索引。构建是一个批处理任务，它需要解析文档内容和链接，然后根据 Page-Rank 对文档进行排列。使用相似性检测算法（比如 simhash）去重，Page-Rank 分值最高的需要保留（被引用次数越多的文档分值越高），其他都需要丢弃，这时就需要将链接替换，避免链接指向了被丢弃的页面或者文档。

MapReduce 属于离线批处理任务，它假定文档不会被修改，每个任务都不用担心 Page-Rank 高的页面被并发修改。

如果我们重新爬取了一部分web文档，这部分文档的内容如何构建索引呢？重跑一次 MapReduce 任务非常耗时，而且本身受影响的页面也不多。使用传统的 DBMS 的事物来更新新增页面的索引也不现实，因为数据量非常庞大，DBMS 根本无法支撑。Bigtable 倒是可以支撑这么大的数据量，但是它没有提供事务，更新时会带来并发问题，导致数据错乱。

有一个理想的处理 web 索引的模型叫增量处理（incremntal processing），它可以维护一个庞大的索引集群，同时还支持一边爬取内容一边高效更新索引。该增量处理将会并发的修改索引文档，还能保持数据一致性，可以跟踪哪些数据已经被处理。

**本文要讲的增量处理系统就是 Percolator，它为用户提供 PB 级别的随机读写和更新，不需要 MapReduce 那样的全局扫描。Percolator 提供了 兼容 ACID 的事物，基于快照的隔离级别**。

除了需要关注并发，增量系统还会跟踪增量计算的状态。Percolator 提供观察者（observer），它就是一些回调代码块，当用户监听的列被修改时就会调用这串代码。Percolator 应用就是将一系列观察者结构化，每个观察者完成一个任务，然后通过更新表数据触发动作，推动下游观察者。外部流程是更新表数据时触发链式结构的第一个观察者。

Percolator 专为增量处理，并不是对现在大数据处理的补充。如果计算不能切分为一个个小更新操作（分而治之），那么它就更适合 MapReduce，比如文件排序；而且如果计算不是强一致性要求，Bigtable 也足够使用。最后，计算量在多个维度会比较大，如果计算量小，传统的 DBMS 也足够使用。

在 Google 内部，Percolator 的主要应用就是准备实时更新搜索引擎使用倒排索引。通过讲索引系统转换为增量系统，我们可以单独处理爬下来的web页面。它可以降低文档平均处理延长和文档平均年龄。

# 设计

Percolator 提供了两大抽象模型，一个是 ACID 事务模型，另一个是组织增量计算的观察者（observer）。

一个 Percolator 系统由三部分组成：Percolator 工作进程、Bigtable tablet server 和 GFS chunkserver。所有的观察者都被链接到 Percolator 的工作进程里，它扫描 Bigtable 被更改的列（事实上，Bigtable 数据发生更改，肯定会产生 WAL 日志，只需要监听这些日志就知道哪些列被修改了），然后以函数调用的方式执行对应的观察者逻辑。观察者执行事务的方式是调用 read/write RPC 到 Bigtable tablet server，然后tablet server 又调用 GFS chunkserver 的 read/write RPC。Percolator 系统还依赖两个小服务：时间戳服务和轻量级锁服务。时间戳服务提供严格单调增的时间戳（所以它是单例的），事务模型中的快照隔离级别需要使用这个属性作为事务版本号。工作进程使用轻量级锁服务提高脏数据通知效率。

![percolator_f1_percolator_and_its_dependencies](/images/percolator_f1_percolator_and_its_dependencies.jpg)

*图-1 Percolator 和它的依赖*

站在开发者的角度来看，Percolator 存储系统由一个个小的表组成，每个表都是由一个个小格子组成，格子就是行和列的索引，一个字节数组用来表示它的值。在内部，为了支持快照隔离级别，我们将每个格子都当作一系列的时间戳索引的数据，时间戳作为数据的版本号。

Percolator 的设计受到两个因素的影响：

* 大规模数据量
* 不要求极致的低延迟

宽松的延迟需求可以让我们使用懒惰模式去清理失败事务的锁，懒惰模式实现简单，可能会将事务的提交延迟个几十秒。对于 OLTP 类型的 DBMS 系统，这种延迟不能接受，但是对于构建索引文档集合的增量处理系统已经足够。Percolator 没有中心化的事务管理机制，缺少死锁监测，这种设计会增加事务冲突的延长时间（有锁等待的情况），但是却可以支撑大规模数据处理。

## Bigtable 简介

Percolator 构建在 Bigtable 分布式存储系统之上。Bigtable 对用户提供多维度的 map 数据结构：key 是 (row, column, timestamp) 元组，key 按照从小到大排序。Bigtable 提供了查找和更新行的操作，而且基于行的操作是 read-modify-write 原子操作，最小事务，或者说修改一行的数据就是一个原子操作。Bigtable 可靠的运行在由大量机器组成的集群上，而且处理着 PB 级别的数据。

一个运行的 Bigtable 由多个 tablet server 和 一个 master 组成。master 管理和协调 tablet server 加载和卸载哪些 tablet；tablet server 为 tablet 提供服务，一个 tablet 就是一系列 SSTable 格式的文件组成，是一块连续区域的键空间。SSTable 文件存储在 GFS；Bigtable 依赖 GFS 可靠的存储能力。BigTable 允许用户调整查询性能：将一组相关联的列放在一个 locality group，这些列会存放在一个 SSTable 文件（或者文件集合）中。由于其他列不在 locality group 对应的 SSTable 文件，这样就可以避免多余的磁盘文件扫描，提高查询速度。

将数据存储在 Bigtable 就塑造了 Percolator 的特性。Percolator 维护着 Bigtable 的接口：数据存放在 Bigtable 的行和列，Percolator 自己的元数据存储在一个特殊的列里（如图-5所示）。Percolator 的 API 与 Bigtable 的 API 非常相似：Percolator 特殊的元数据所在的列的操作接口就是将 Bigtable 的大多数 API 封装了一层。**最具挑战的就是提供了 Bigtable 不具备的多行事务和观察者框架**。

## 分布式事务

Percolator 提供跨行跨表的快照隔离级别的事物。Percolator 用户使用特定的语言来写事务代码（当前是 C++），中间再穿插 Percolator API 代码。下面的代码块是一个对文档做 Hash 计算的简单版本代码。

```c
bool UpdateDocument(Document doc) {
    Transaction t(&cluster);
    t.Set(doc.url(), "contents", "document", doc.contents());
    int hash = Hash(doc.contents());
    // dups table maps hash → canonical URL
    string canonical;
    if (!t.Get(hash, "canonical-url", "dups", &canonical)) {
        // No canonical yet; write myself in
        t.Set(hash, "canonical-url", "dups", doc.url());
    } // else this document already exists, ignore new copy
    return t.Commit();
}
```

*代码块-1*

在这个例子中，如果 `Commit()` 方法返回 `false`，表示事务有冲突（此例中，因为两个相同 Hash 的 URL 同时处理了 ），需要退避后再继续重试。调用 `Get()` 和 `Commit()` 会阻塞；提高并行度的方式是在线程池内执行多个事务。

增量数据处理任务也可以不使用强事务，事务可以让开发变得简单，避免将错误引入到一个长期存活的数据系统中。比如，在一个事务性的 web 索引系统中，开发者可以假设文档内容和hash与构建的索引数据完全一致。如果没有事务，系统抽风了的时候就会导致永久错误，文档内容与构建的索引数据不一致。事务也可以很轻松的让构建的索引一直保持最新且一致。注意，这个例子需要的事务能够跨行，而不仅仅是 Bigtable 能提供的单行级别的事务。

Percolator 使用 Bigtable 的 timestamp 维度来存储数据的多个版本。多版本需要提供快照级别的隔离性，表现的就是在一个 timestamp 所读到的数据都是一致的，因为它只会读取比当前 timestamp 小的数据。