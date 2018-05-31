---
title: mysql-server源码阅读02 - 文件格式
date: 2018-04-29 22:01:02
tags: mysql
---

# 一、.frm

`.frm`是mysql的表描述文件，包括以下几个部分：

* File Header Section
* File Key Information Section
* File Comment Section
* File Column Information Section

详细请参考[frm File Format](https://dev.mysql.com/doc/internals/en/frm-file-format.html)。

# 二、.ibd

`.ibd`是`Innodb`存储引擎存储表数据的文件，包括行数据和索引数据。Jeremy Cole写了一篇关于[Innodb internals,structures,behavior](https://blog.jcole.us/innodb/)的文章，详细的介绍了该文件的格式，还包括`ruby`工具。也可以参考mysql internal manuals的[Innodb Storage Engine](https://dev.mysql.com/doc/internals/en/innodb.html)。

`Innodb`数据存储模型使用“空间”（`space`）或者“表空间”（有时也称作“文件空间”）的概念，相关文件有`ibdata1`、`ibdata2`和`.ibd`，存在`.ibd`的原因是mysql支持`file per table`将一个表的数据从`ibdata*`中移到`.ibd`中存储。每个“空间“都有一个32位整数的id，用于被其他地方引用到。“空间”被划分成多个“页”（`Page`），通常每页16KB（由UNIV_PAGE_SIZE和是否压缩决定）。每“页”都有一个32位整数的id，称为`offset`，表示从“空间”起始的偏移值，所以“页0”表示“空间”（或者文件）起始处，“页1”在16384（`16 * 1024`）字节处。由于“页”id是32位整数，而每“页”有16KB大小，所以每个“空间”最大为`2^32 * 16KB=64TB`。



“空间”文件由“页”组成，最多有2的32次方个，为了便于管理，将多个“页”归为一个“块”，大小为1MB，称为`extent`。



每个“页”都有两个指针，分别指向“前”一个“页”和“后”一个“页”，组成一个双向链表的结构。

# 三、索引

`Innodb`有两种索引类型，一种是`聚集索引（Clustered Index）`，另一个种是`二级索引（Secondary Index）`。这两种索引类型的结构都一样，使用B+树构成，每一个节点都包含一个KEY和一个VALUE。Mysql为每一张`Innodb`表创建一个`聚集索引`，用于存在“行数据”，“行数据”的主键作为索引的KEY，当没有主键时，mysql内部生成一个主键，行中非主键的字段作为VALUE。`二级索引`就是使用`create index`创建的索引，创建二级索引时指定的列作为KEY，VALUE就是主键。有关B+树结构可以参考[维基百科](https://zh.wikipedia.org/zh-hans/B%2B%E6%A0%91)，节点的创建、调整等可以参考[可视化数据结构](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)。



索引页包含`用户记录`和`系统记录`，其中`系统记录`是两个特殊的记录，分别为`infimum`和`supremum`，而`用户记录`就是实际的数据。每一个记录都有一个`next record`指针，以此在“页”内构成单向链表，首节点是`infimum`，`infimum`的`next record`指向第一条`用户记录`，最后一条`用户记录`的`next record`指向`supremum`，`supremum`的`next record`为`NULL`。需要注意的是“记录”为单向链表，“页”为双向链表。



索引页分为`root`页、`internal`页和`leaf`页，其中`leaf`页的数据就是“行数据”，键为主键，`root`页和`internal`页的数据是`internal`页或者`leaf`页的指针，而键表示主键的最小值。



假设有如下表：

```sql
CREATE TABLE t_btree (
  i INT NOT NULL,
  s CHAR(10) NOT NULL,
  PRIMARY KEY(i)
) ENGINE=InnoDB;
```

插入7条数据：

```sql
insert into t_btree values(0, 'A'),(1, 'B'),(2, 'C'),(3, 'D'),(4, 'E'),(5, 'F'),(6, 'G'),(7, 'H');
```

则聚集索引如下图"B+树结构"所示：

![B+树结构](http://jcole.us/blog/files/innodb/20130109/72dpi/B_Tree_Structure.png "B+树结构")

以`Page 4`来说，它有两条用户记录，第一条用户记录表示`主键i大于等于0`的记录在`Page 6`，第二条记录表示`主键i大于等于2`的记录在`Page 7`。