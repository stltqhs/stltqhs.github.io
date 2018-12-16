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

在介绍DML涉及的锁操作前，需要先介绍Innodb如何定位记录以便进行DML操作。在Innodb中定位记录的实现是index page cursor（即索引页的游标），它是用来指向记录所在位置的游标，在源码中的数据结构如下：

```c
// page0cur.h
/** Index page cursor */
struct page_cur_t{
	const dict_index_t*	index;
	rec_t*		rec;	/*!< pointer to a record on page */
	ulint*		offsets;
	buf_block_t*	block;	/*!< pointer to the block containing rec */
};
```

定位记录的函数是`page_cur_search_with_match`，声明如下：

```c
// page0cur.cc
/****************************************************************//**
Searches the right position for a page cursor. */
void
page_cur_search_with_match(
/*=======================*/
	const buf_block_t*	block,	/*!< in: buffer block */
	const dict_index_t*	index,	/*!< in/out: record descriptor */
	const dtuple_t*		tuple,	/*!< in: data tuple */
	page_cur_mode_t		mode,	/*!< in: PAGE_CUR_L,
					PAGE_CUR_LE, PAGE_CUR_G, or
					PAGE_CUR_GE */
	ulint*			iup_matched_fields,
					/*!< in/out: already matched
					fields in upper limit record */
	ulint*			ilow_matched_fields,
					/*!< in/out: already matched
					fields in lower limit record */
	page_cur_t*		cursor,	/*!< out: page cursor */
	rtr_info_t*		rtr_info)/*!< in/out: rtree search stack */
```

假如存在下面的记录：

```text
1,2,2,3,3,3,4,5,5,6,7
```

定位大于等于3或者小于3的记录的查询过程如下：

```text
初始时：
   1   2   2   3   3   3   4   5   5   6   7
   ^                   ^                   ^
   |                   |                   |
  low                 mid                 up

由于定位的记录是大于等于3或者小于3，记录都在mid的左侧，因此需要将up移动到mid位置：
   1   2   2   3   3   3   4   5   5   6   7
   ^       ^           ^
   |       |           |
  low     mid          up
  
   1   2   2   3   3   3   4   5   5   6   7
           ^   ^       ^
           |   |       |
          low mid      up
 
 直到low与up相差1:
   1   2   2   3   3   3   4   5   5   6   7
           ^   ^
           |   |
          low up
```

若查询模式为大于等于3，则page cursor定位到up所指的记录3；若查询的模式为小于3，则page cursor定位到所指向的记录2。

记录定位的方法已经介绍完，接下来介绍DML涉及的锁操作。

INSERT涉及的锁操作为：

* 首先对表加上IX表锁；
* 根据查询模式PAGE_CUR_LE定位记录的next_rec；
* 判断记录next_rec是否有锁，有的话就等待锁的释放，否则直接插入；

UPDATE/DELETE涉及的锁操作为：首先尝试对更新的记录加上X锁，若待更新的记录上存在其他锁时，则事务被阻塞，需要等待记录上的锁被释放。

[死锁(Deadleck)](https://zh.wikipedia.org/wiki/%E6%AD%BB%E9%94%81)在维基百科的定义为：当两个以上的运算单元，双方都在等待对方停止运行，以获取系统资源，但是没有一方提前退出时，就称为死锁。死锁的四个条件是：

* 禁止抢占 no preemption - 系统资源不能被强制从一个进程/线程中退出

* 持有和等待 hold and wait - 一个进程/线程可以在等待时持有系统资源

* 互斥 mutual exclusion - 只有一个进程/线程能持有一个资源

* 循环等待 circular waiting - 一系列进程/线程互相持有其他进程/线程所需要的资源

死锁只有在这四个条件同时满足时出现。预防死锁就是至少破坏这四个条件其中一项，即破坏“禁止抢占”、破坏“持有等待”、破坏“资源互斥”和破坏“循环等待”。由于MYSQL系统并发性和锁的设计，可以被破坏的规则只有“循环等待”，也就是要保证加锁的顺序是一致的。

Innodb使用等待图（wait-for graph）的死锁检测方法来检测死锁，它要求在数据库中保存以下两种信息：

* 锁的信息链表
* 事务等待链表

通过上述链表可以构造出一张图，而在这个图中若存在回路，那么就代表存在死锁。

与锁相关的注意事项有：

* 如果表数据并发量大，就不要将主键设置为自增auto_increment，因为自增锁是表锁；
* 行锁实质是索引记录锁，如果锁住的记录资源越少，并发性就越高，所以对于UPDATE/DELETE操作的WHERE条件需要能命中索引（聚集索引或者二级索引），如果不能命中，则Innodb需要将所有聚集索引记录加锁；
* 不要使用`SELECT ... LOCK IN SHARE MODE`，它容易造成死锁。

为什么使用`SELECT ... LOCK IN SHARE MODE`容易造成死锁？上面说到要避免死锁就需要保证加锁的顺序是一致的。但如果使用`SELECT ... LOCK IN SHARE MODE`，就算加锁顺序是一致的，依然会出现死锁。

假如有如下SQL代码，当有两个线程并发执行这串SQL代码时，会出现死锁。

```sql
start transaction;
select * from table where id = 'A' lock in share mode;
update table set column = 'B' where id = 'A';
commit;
```

两个线程并发执行时的逻辑顺序如下：

```sql
start transaction; -- Transaction A
start transaction; -- Transaction B
select * from table where id = 'A' lock in share mode; -- Transaction A
select * from table where id = 'A' lock in share mode; -- Transaction B
update table set column = 'B' where id = 'A'; -- Transaction A
update table set column = 'B' where id = 'A'; -- Transaction B

```

当执行完第4行代码时，事务A和事务B都持有记录A的`LOCK_S`锁，当执行第5行时，由于事务B持有记录A的`LOCK_S`锁，事务A需要等待事务B释放记录A的`LOCK_S`锁，因此事务A需要等待。当执行第6行代码时，由于事务B需要等待事务A释放记录A的`LOCK_S`锁，于是出现了互相等待（循环等待），即死锁。

`SHOW ENGINE INNODB STATUS`可以用来查看当前运行中的事务占用的锁以及上次发生死锁的锁信息。为了说明`SHOW ... STATUS`的用法，使用表`t_device`来叙述。表`t_device`的DDL如下：

```sql
CREATE TABLE `t_device` (
  `id` varchar(16) NOT NULL,
  `mac` varchar(16) DEFAULT NULL,
  `did` varchar(22) DEFAULT NULL,
  `status` tinyint(3) unsigned DEFAULT NULL,
  `create_at` datetime DEFAULT NULL,
  `update_at` datetime DEFAULT NULL,
  `product_key` varchar(45) DEFAULT NULL,
  `inter_type` smallint(6) DEFAULT '1',
  `protocol_version` smallint(6) DEFAULT '0',
  `pile_id` varchar(19) DEFAULT NULL,
  `channel_id` varchar(32) DEFAULT NULL,
  `channel_type` tinyint(3) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_mac` (`mac`),
  UNIQUE KEY `idx_did` (`did`),
  KEY `idx_channel_id` (`channel_type`,`channel_id`,`create_at`)
) ENGINE=InnoDB
```

插入数据：

```sql
INSERT INTO t_device(id,mac,did,status,create_at,update_at,product_key,inter_type,protocol_version,pile_id,channel_id,channel_type) values ('0000000000000027', '868986021449112', 'dY9VMJSNZHfaxK9QmPnRCU', 4, '2018-07-28 09:01:02', '2018-12-15 11:55:40', '696018edf68443e3b9856dc87aefe251', 48, 0, NULL, NULL, 0);
```

先检查输出锁信息的变量值是否设置为开启状态，如下：

```sql
mysql> show variables like 'innodb_status_output_locks';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_status_output_locks | ON    |
+----------------------------+-------+
1 row in set (0.00 sec)

```

如果`Value`为`ON`表示已经开启，否则需要执行SQL语句将其开启，如下：

```sql
set global innodb_status_output_locks = 1;
```

 开启输出锁信息后，可以执行更新记录和查看锁记录的操作，如下：

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)
mysql> update t_device set status = 3 where id = '0000000000000027';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
mysql> show engine innodb status\G
---TRANSACTION 48945, ACTIVE 8 sec
2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1
MySQL thread id 45, OS thread handle 0x70000c4b5000, query id 25439 localhost root init
show engine innodb status
TABLE LOCK table `charge`.`t_device` trx id 48945 lock mode IX
RECORD LOCKS space id 839 page no 8 n bits 224 index `PRIMARY` of table `charge`.`t_device` trx id 48945 lock_mode X locks rec but not gap
Record lock, heap no 154 PHYSICAL RECORD: n_fields 14; compact format; info bits 0
 0: len 16; hex 30303030303030303030303030303237; asc 0000000000000027;;
 1: len 6; hex 00000000bf31; asc      1;;
 2: len 7; hex 7100000d2a2e50; asc q   *.P;;
 3: len 15; hex 383638393836303231343439313132; asc 868986021449112;;
 4: len 22; hex 645939564d4a534e5a486661784b39516d506e524355; asc dY9VMJSNZHfaxK9QmPnRCU;;
 5: len 1; hex 03; asc  ;;
 6: len 5; hex 99a0789042; asc   x B;;
 7: len 5; hex 99a19ebde8; asc      ;;
 8: len 32; hex 3639363031386564663638343433653362393835366463383761656665323531; asc 696018edf68443e3b9856dc87aefe251; (total 32 bytes);
 9: len 2; hex 8030; asc  0;;
 10: len 2; hex 8000; asc   ;;
 11: SQL NULL;
 12: SQL NULL;
 13: len 1; hex 00; asc  ;;
```

为了输出内容精简，已经去掉其他信息。第10行表示事务48945正在执行`show engine innodb status`SQL语句。第11行表示事务48945获取了表`t_device`的意向排他锁`IX`，因为执行了更新表`t_device`记录的SQL语句。第12行表示锁记录信息，根据前面对锁记录数据结构的描述，可以知道`space id 839`、`page no 8`、`n bits 224`、`heap no 154`的意义。`lock_mode X locks rec but not gap`表示的事`LOCK_X`类型的记录锁（rec lock）但没有使用间隙锁锁住该记录之前的数据。`n_fields 14`表示该记录锁有14个字段，这些字段是表结构中聚集索引或者二级索引相关的信息。由于第3行是按照主键更新，因此需要使用聚集索引。对于聚集索引，Innodb为每一行添加3个额外的字段，分别是6字节的DB_TRX_ID（记录插入或者更新该行的事务ID）；1字节的删除标记位Deleted（当执行Deleted时，将该标记置为1）；7字节的DB_ROLL_PTR（用来记录修改前的行指针）。所以第14行就是`t_device`的id，因为更新的是id为0000000000000027的记录，该行显示该记录的长度是16，十六进制hex的内容是`30303030303030303030303030303237`，ASC码内容为`0000000000000027`，因为id列的类型是字符类型，因此ASC码的内容正是第3行更新语句中指定的id。第15行是6字节的DB_TRX_ID，由于第5行的`Change`为1，表示执行UPDATE语句更新了一条记录，更新该记录的事务号是48945，而48945的HEX格式正是`bf31`。第16行是7字节的DB_ROLL_PTR。从第17行起显示的内容都是表t_device除id列外的其他列的值信息，故第17行是mac，第18行是did，以此类推。

根据前面叙述的内容可以解释下列死锁原因：

```sql
start transaction; -- Transaction A
start transaction; -- Transaction B
update t_device set status = 4 where id = '0000000000000027'; -- Transaction A
update t_device set mac = null,did = null where mac = '868986021449112' and did = 'dY9VMJSNZHfaxK9QmPnRCU'; -- Transaction B
update t_device set mac = null,did = null where mac = '868986021449112' and did = 'dY9VMJSNZHfaxK9QmPnRCU'; -- Transaction A
```

执行到第4行时，由于记录0000000000000027的mac也是`868986021449112`，而mac是二级索引，因此需要先锁定二级索引的记录，再锁住聚集索引的记录。而因为第3行的原因，事务B需要等待事务A释放记录0000000000000027的锁，因此事务B所属的线程会等待。当执行到第5行时，由于二级索引的记录已经被事务B获取，因此事务A又等待事务B等待释放二级索引的记录锁，所以出现了循环等待，于是死锁。

参考：[InnoDB Locking](https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html)，[MySQL内核：InnoDB存储引擎 卷1](https://book.douban.com/subject/25872763/)，[MySQL · 引擎特性 · Innodb 锁子系统浅析](http://mysql.taobao.org/monthly/2017/12/02/)

# 存储引擎

# 优化

# 容量扩展

[MaxScale](https://github.com/mariadb-corporation/MaxScale/tree/1.4.3)，[Galara](https://github.com/codership/galera),https://severalnines.com/resources/tutorials/galera-cluster-mysql-tutorial

参考：[设计数据密集型应用](https://vonng.gitbooks.io/ddia-cn/content/part-i.html)

# Nosql

参考：[MongoDB – Internals & Performance](https://iuliantabara.com/2016/01/mongodb-internals-performance/)