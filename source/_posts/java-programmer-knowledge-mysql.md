---
title: Java后端开发 - MYSQL
date: 2018-10-13 08:15:36
tags: mysql
---



# 三范式

范式是“符合某一种级别的关系模式的集合，表示一个关系内部各属性之间的联系的合理化程度”，通俗地说就是一张数据表的表结构所符合的某种设计标准的级别。数据库范式分为1NF，2NF，3NF，BCNF，4NF，5NF。一般在我们设计关系型数据库的时候，最多考虑到BCNF就够。符合高一级范式的设计，必定符合低一级范式，例如符合2NF的关系模式，必定符合1NF。

**第一范式（1NF）指符合1NF的关系中的每个属性都不可再分**，1NF是所有关系型数据库的最基本要求。但是仅仅符合1NF的设计，仍然会存在数据冗余过大，插入异常，删除异常，修改异常的问题，需要更高级范式来解决这些问题。

**第二范式（2NF）指2NF在1NF的基础之上，消除了非主属性对于码的部分函数依赖**。

**码**的定义是关系中的某个属性或者某几个属性的组合，用于区分每个元组，设 K 为某表中的一个属性或属性组，若除 K 之外的所有属性都完全函数依赖于 K（这个“完全”不要漏了），那么我们称 K 为**候选码**，简称为**码**。在实际中我们通常可以理解为：**假如当 K 确定的情况下，该表除 K 之外的所有属性的值也就随之确定，那么 K 就是码。**一张表中可以有超过一个码。包含在任何一个码中的属性称为主属性，除去主属性就是**非主属性**。**若在一张表中，在属性（或属性组）X的值确定的情况下，必定能确定属性Y的值，那么就可以说Y函数依赖于X，写作 X → Y**。在一张表中，若 X → Y，且对于 X 的任何一个真子集（假如属性组 X 包含超过一个属性的话），X ' → Y 不成立，那么我们称 Y 对于 X **完全函数依赖**，记作 X F→ Y。假如 Y 函数依赖于 X，但同时 Y 并不完全函数依赖于 X，那么我们就称 Y **部分函数依赖**于 X，记作 X P→ Y。假如 Z 函数依赖于 Y，且 Y 函数依赖于 X （Y 不包含于 X，且 X 不函数依赖于 Y），那么我们就称 Z **传递函数依赖**于 X ，记作 X T→ Z。

根据2NF的定义，判断的依据实际上就是看数据表中**是否存在非主属性对于码的部分函数依赖**。若存在，则数据表最高只符合1NF的要求，若不存在，则符合2NF的要求。判断的方法是：

第一步：找出数据表中所有的**码**。
第二步：根据第一步所得到的码，找出所有的**主属性**。
第三步：数据表中，除去所有的主属性，剩下的就都是**非主属性**了。
第四步：查看是否存在非主属性对码的**部分函数依赖**。

消除这些部分函数依赖，只有一个办法，就是将大数据表拆分成两个或者更多个更小的数据表，在拆分的过程中，要达到更高一级范式的要求，这个过程叫做”模式分解“，模式分解的方法不是唯一的。

**第三范式（3NF）** **3NF在2NF的基础之上，消除了非主属性对于码的传递函数依赖**，也就是说， 如果存在非主属性对于码的传递函数依赖，则不符合3NF的要求。

符合3NF要求的数据库设计，**基本**上解决了数据冗余过大，插入异常，修改异常，删除异常的问题。当然，在实际中，往往为了性能上或者应对扩展的需要，经常 做到2NF或者1NF。



参考：[如何解释关系数据库的第一第二第三范式？](https://www.zhihu.com/question/24696366)

# 悲观锁和乐观锁

**悲观锁**指假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。Java synchronized 就属于悲观锁的一种实现，每次线程要修改数据时都先获得锁，保证同一时刻只有一个线程能操作数据，其他线程则会被block。

**乐观锁**指假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。Java JUC中的atomic包就是乐观锁的一种实现，AtomicInteger 通过CAS（Compare And Set）操作实现线程安全的自增。在数据库中，乐观锁的实现方法是使用数据版本（version）记录机制，当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据。

参考：[MySQL 乐观锁与悲观锁](https://www.jianshu.com/p/f5ff017db62a)

# Innodb事务隔离级别

数据库事务（简称：事务）是数据库管理系统执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成。

事务的ACID特征为原子性（ Atomicity ）、一致性（ Consistency ）、隔离性（ Isolation ）和持久性（ Durability ）。

* 原子性：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行；

* 一致性：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束；

* 隔离性：多个事务并发执行时，一个事务的执行不应影响其他事务的执行；

* 持久性：已被提交的事务对数据库的修改应该永久保存在数据库中。

Mysql的Innodb引擎支持事务，定义了4类隔离级别，分别是读取未提交，读取提交，可重复读，序列化。

* **Read Uncommitted（读取未提交）**

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

* **Read Committed（读取提交）**

一个事务只能看见已提交事务所做的改变。这种隔离级别解决了脏读的问题，但是存在不可重复读（Nonrepeatable Read）的问题，因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

* **Repeatable Read（可重复读）**

这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。这种隔离级别解决了不可重复读的问题，但是在[SQL-92](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)的定义中还存在存在幻读 （Phantom Read），Innodb中不存在幻读。幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency Control）机制解决了该问题。

* **Serializable（可串行化）**

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

不同隔离级别的读问题如下表：

| Isolation level  | Lost updates | Dirty reads | Non-repeatable reads | Phantoms    |
| ---------------- | ------------ | ----------- | -------------------- | ----------- |
| Read Uncommitted | don't occur  | may occur   | may occur            | may occur   |
| Read Committed   | don't occur  | don't occur | may occur            | may occur   |
| Repeatable Read  | don't occur  | don't occur | don't occur          | may occur   |
| Serializable     | don't occur  | don't occur | don't occur          | don't occur |

数据库遵循的是两段锁协议，将事务分成两个阶段，加锁阶段和解锁阶段（所以叫两段锁）。

加锁阶段：在该阶段可以进行加锁操作。在对任何数据进行读操作之前要申请并获得S锁（共享锁，其它事务可以继续加共享锁，但不能加排它锁），在进行写操作之前要申请并获得X锁（排它锁，其它事务不能再获得任何锁）。加锁不成功，则事务进入等待状态，直到加锁成功才继续执行。

解锁阶段：当事务释放了一个封锁以后，事务进入解锁阶段，在该阶段只能进行解锁操作不能再进行加锁操作。

在可重复读的隔离级别中，Innodb使用MVCC解决幻读，原理是：Innodb为每一行添加3个额外的字段，分别是6字节的DB_TRX_ID（记录插入或者更新该行的事务ID）；1字节的删除标记位Deleted（当执行Deleted时，将该标记置为1）；7字节的DB_ROLL_PTR（用来记录修改前的行指针），当一个事物读取某一行时，mysql内部使用[ReadView](https://github.com/stltqhs/mysql-server/blob/5.7/storage/innobase/include/read0types.h)结构记录读取行的事务号，而且读取行的DB_TRX_ID不会大于本事务的事务ID。当其他事务修改该行时，会创建新的记录用来存储更改的数据，而由于readview的关系，当前事务不会读取到修改的数据。当执行delete操作时，mysql不会立即删除该行，而是将该行标记为Deleted。对于旧的行数据，mysql内部会检查是否存在其他事务正在读取或者已经读取该行，如果没有任何事务读取该行，则该行将在*purge*线程中清除。可以使用锁的方式使得事务总是读取最新行，这种方式称为“当前读”，与之对应的就称为“快照读”。对于普通的Select查询操作都是“快照读”，而Update，Delete，Select ... lock in share mode和Select ... for update则都是“当前读”。

参考：[MySQL（二）｜深入理解MySQL的四种隔离级别及加锁实现原理](https://www.jianshu.com/p/dab1c0ecbac0)，[数据库事务](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1)，[Isolation](https://en.wikipedia.org/wiki/Isolation_%28database_systems%29)，[InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html)

# Innodb索引

参考：[MySQL源码：索引相关的数据结构(前篇)](http://www.orczhou.com/index.php/2012/11/mysql-source-code-data-structure-about-index/)，[MySQL源码：索引相关的数据结构(后篇)](http://www.orczhou.com/index.php/2012/11/mysql-source-code-how-mysql-find-usable-index/)

# Innodb锁

锁是用来实现事务一致性和隔离性的常用技术，Innodb存储引擎实现了两种行级锁：

* 共享锁（S Lock），允许事物读一行事物；
* 排他锁（X Lock），允许事物删除或者更新一行数据；

Innodb存储引擎也支持允许表锁和行锁共存的多粒度意向锁（即表级锁）：

* 意向共享锁（IS Lock），事务想要获取一张表中某几行的共享锁；
* 意向排他锁（IX Lock），事务想要获得一张表中某几行的排他锁；

因为Innodb存储引擎支持的是行级别的锁，因此意向锁不会阻塞除全表扫描（如`LOCK TABLES ... WRITE`）以外的锁。

在Mysql源码的lock0priv.h定义了几种锁的兼容性，代码如下：

```c
static const byte lock_compatibility_matrix[5][5] = {
 /**         IS     IX       S     X       AI */
 /* IS */ {  TRUE,  TRUE,  TRUE,  FALSE,  TRUE},
 /* IX */ {  TRUE,  TRUE,  FALSE, FALSE,  TRUE},
 /* S  */ {  TRUE,  FALSE, TRUE,  FALSE,  FALSE},
 /* X  */ {  FALSE, FALSE, FALSE, FALSE,  FALSE},
 /* AI */ {  TRUE,  TRUE,  FALSE, FALSE,  FALSE}
};
```

> AI是Auto-Increment锁，即自增锁，是表级锁

Innodb存储引擎支持三种行锁的算法：

* record lock：单个索引记录上的锁，即记录锁；
* gap lock：间隙锁，锁定一个范围，但不包含记录本身；
* next-key lock：gap lock + record lock，锁定一个范围，并且锁定记录本身；

Innodb表级锁数据结构定义如下：

```c
// lock0priv.h
/** A table lock */
struct lock_table_t {
	dict_table_t*	table;		/*!< database table in dictionary
					cache */
	UT_LIST_NODE_T(lock_t)
			locks;		/*!< list of locks on the same
					table */
};
```

记录锁数据结构定义如下：

```c
// lock0priv.h
/** Record lock for a page */
struct lock_rec_t {
	ib_uint32_t	space;		/*!< space id */
	ib_uint32_t	page_no;	/*!< page number */
	ib_uint32_t	n_bits;		/*!< number of bits in the lock
					bitmap; NOTE: the lock bitmap is
					placed immediately after the
					lock struct */
};
```

`lock_rec_t`结构说明记录锁是根据页的组织形式来管理的（Innodb文件结构可阅读[Innodb File Space](https://dev.mysql.com/doc/refman/5.6/en/innodb-file-space.html)，更详细的说明可阅读Jeremy Cole的文章[Innodb](https://blog.jcole.us/innodb/)），若要知道页中某一条记录是否已经有锁，则通过位图的方式，即位图中值为1表示该记录已经持有锁，位图中索引与记录的`heap_no`（页上每行数据紧接着存放，内部使用一个 `heap_no`来表示是第几行数据。因此`[space, page_no, heap_no]`可以唯一确定一行）一一对应。

由于锁是被事务持有，因此还需要`lock_t`数据结构，定义如下：

```c
// lock0priv.h
/** Lock struct; protected by lock_sys->mutex */
struct lock_t {
	trx_t*		trx;		/*!< transaction owning the
					lock */
	UT_LIST_NODE_T(lock_t)
			trx_locks;	/*!< list of the locks of the
					transaction */

	dict_index_t*	index;		/*!< index for a record lock */

	lock_t*		hash;		/*!< hash chain node for a record
					lock. The link node in a singly linked
					list, used during hashing. */

	union {
		lock_table_t	tab_lock;/*!< table lock */
		lock_rec_t	rec_lock;/*!< record lock */
	} un_member;			/*!< lock details */

	ib_uint32_t	type_mode;	/*!< lock type, mode, LOCK_GAP or
					LOCK_REC_NOT_GAP,
					LOCK_INSERT_INTENTION,
					wait flag, ORed */
};
```

由于一个事务可能在不同页上有多个行锁，因此需要使用`trx_locks`将一个事务的所有锁信息进行链接。`type_mode`中的lock type有：

```c
// lock0lock.h
#define LOCK_TABLE	16	/*!< table lock */
#define	LOCK_REC	32	/*!< record lock */
```

lock mode有：

```c
// lock0types.h
/* Basic lock modes */
enum lock_mode {
	LOCK_IS = 0,	/* intention shared */
	LOCK_IX,	/* intention exclusive */
	LOCK_S,		/* shared */
	LOCK_X,		/* exclusive */
	LOCK_AUTO_INC,	/* locks the auto-inc counter of a table
			in an exclusive mode */
	LOCK_NONE,	/* this is used elsewhere to note consistent read */
	LOCK_NUM = LOCK_NONE, /* number of lock modes */
	LOCK_NONE_UNSET = 255
};
```

表示Innodb整个事务锁系统的数据结构是`lock_sys_t`，定义如下：

```c
// lock0lock.h
/** The lock system struct */
struct lock_sys_t{
	char		pad1[CACHE_LINE_SIZE];	/*!< padding to prevent other
						memory update hotspots from
						residing on the same memory
						cache line */
	LockMutex	mutex;			/*!< Mutex protecting the
						locks */
	hash_table_t*	rec_hash;		/*!< hash table of the record
						locks */
	hash_table_t*	prdt_hash;		/*!< hash table of the predicate
						lock */
	hash_table_t*	prdt_page_hash;		/*!< hash table of the page
						lock */

	char		pad2[CACHE_LINE_SIZE];	/*!< Padding */
	LockMutex	wait_mutex;		/*!< Mutex protecting the
						next two fields */
	srv_slot_t*	waiting_threads;	/*!< Array  of user threads
						suspended while waiting for
						locks within InnoDB, protected
						by the lock_sys->wait_mutex */
	srv_slot_t*	last_slot;		/*!< highest slot ever used
						in the waiting_threads array,
						protected by
						lock_sys->wait_mutex */
	ibool		rollback_complete;
						/*!< TRUE if rollback of all
						recovered transactions is
						complete. Protected by
						lock_sys->mutex */

	ulint		n_lock_max_wait_time;	/*!< Max wait time */

	os_event_t	timeout_event;		/*!< Set to the event that is
						created in the lock wait monitor
						thread. A value of 0 means the
						thread is not active */

	bool		timeout_thread_active;	/*!< True if the timeout thread
						is running */
};
```

`lock_sys_t`中的`rec_hash`哈希表的键通过调用`lock_rec_fold(space, page_no)`生成。因此若需要查询某一行记录是否有锁，首先是根据行所在的页进行哈希查询，然后根据查询得到的`lock_rec_t`，扫描 lock bitmap 才能最终得到该行记录是否有锁。

对表加锁通过函数`lock_table`完成，表加锁是根据事务和表来进行的。函数`lock_table`中会调用函数`lock_table_create`来完成对表锁对象`lock_t`的初始化。产生的表锁对象`lock_t`加入到表对象`lock_table_t`的`locks`链表中。`lock_table`函数在[lock0lock.cc](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/lock/lock0lock.cc)文件中实现，通过搜索该函数的调用层级，可以知道哪些SQL操作会调用`lock_table`。创建索引时`lock_table`的`mode`参数是`LOCK_S`，当执行DELETE和UPDATE语句时`mode`参数是`LOCK_IX`，当执行SELECT语句时`mode`参数是`LOCK_IS`。根据锁兼容性定义`lock_compatibility_matrix`，`LOCK_S`和`LOCK_IX`不能兼容，故索引创建索引时，执行DELETE和UPDATE会阻塞，而SELECT不会阻塞。

对记录加锁通过函数`lock_rec_lock`完成，行记录加锁根据事务和页来进行。函数`lock_rec_lock`中会调用函数`lock_rec_create`创建一个`lock_t`对象，并加入到等待队列。由于行锁使用位图实现，因此如果多个记录的page_no和锁模式相同时会重用`lock_t`对象，标记对应head_no位为1。

一般来说，加锁过程可简述为：

* 通过主键进行加锁的语句，仅对聚集索引记录进行加锁；
* 通过二级索引记录进行加锁的语句，首先对二级索引记录加锁，再对聚集索引记录进行加锁；
* 通过二级索引记录进行加锁的语句，可能还需要对下一个记录进行加锁。

需要对二级索引记录的下一条记录加锁是为了避免幻读问题。例如下面的语句，其不仅需要锁住二级索引记录xxx，还需要锁住xxx的下一条记录。这样就可以避免其他事务插入同样值为xxx的记录，从而避免幻读问题：

```sql
SELECT * FROM table WHERE key_column = xxx FOR UPDATE;
```

若key_column包含唯一约束，那么就不需要锁定下一个二级索引记录。但这仅对等值查询有效，对于非等值条件，不管二级索引是否包含唯一约束都需要锁定下一个索引记录，从而避免幻读问题的产生，如下面的语句：

```sql
SELECT * FROM table WHERE key_column <= xxx FOR UPDATE;
```



参考：[InnoDB Locking](https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html)，[MySQL内核：InnoDB存储引擎 卷1](https://book.douban.com/subject/25872763/)，[MySQL · 引擎特性 · Innodb 锁子系统浅析](http://mysql.taobao.org/monthly/2017/12/02/)

# 存储引擎

# 优化

# 容量扩展

[MaxScale](https://github.com/mariadb-corporation/MaxScale/tree/1.4.3)，[Galara](https://github.com/codership/galera),https://severalnines.com/resources/tutorials/galera-cluster-mysql-tutorial

参考：[设计数据密集型应用](https://vonng.gitbooks.io/ddia-cn/content/part-i.html)

# Nosql

参考：[MongoDB – Internals & Performance](https://iuliantabara.com/2016/01/mongodb-internals-performance/)