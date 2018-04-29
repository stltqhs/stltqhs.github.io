---
title: mysql-server源码阅读01 - mysql-server模块与架构
date: 2018-04-29 18:46:59
tags: mysql
---

# 前言

[mysql-server](https://github.com/mysql/mysql-server)是当前非常流行的开源`RDBMS`，在其基础上还有多个开源版本，如`Percona`、`Maria DB`等，其中以`Percona`最为常用，不仅包括对`mysql-server`的很多优化，还包括很多实用工具，而且开发社区很活跃。阅读`mysql-server`源码需要参考[MySQL Internals Manual](https://dev.mysql.com/doc/internals/en/)，还可以阅读书籍，如[Understanding Mysql Internals](http://shop.oreilly.com/product/9780596009571.do)。关于`mysql-server`的编译与调试可以参考[mysql-server编译与调试](https://stltqhs.github.io/2018/03/31/mysql-compile/)。

# mysql-server模块

![mysql-server模块](/images/mysql-modules.jpg "mysql-server模块")

### 1.服务器初始化模块

`mysqld`是`mysql-server`服务器的二进制文件，可以启动整个`mysql-server`程序，自然的`mysqld`就包含一个`main`方法，是整个程序的入口，该`main`方法位于`sql/mysqld.cc`。比较重要的方法有以下几个：

* Init_command_variables()
* Init_thread_environment()
* Init_server_components()
* sql/sql_acl.cc中的grant_init()
* sql/slave.cc中的init_slave()
* get_options()

### 2.连接器管理

监听来自客户端的链接，然后将请求派发给线程管理器，入口方法是`sql/mysqld.cc`中的`handle_connections_sockets()`。

### 3.线程管理器

线程管理器负责跟踪线程，确保分配线程，以处理来自客户端的链接。入口方法是`sql/mysqld.cc`中的create_new_thread()，还有一个特别的方法是`start_cached_thread()`。

### 4.连接线程

在一个已建立的连接上处理客户端请求，入口方法是`sql/sql_parse.cc中`的`handle_one_connection()`。

### 5.用户验证

验证所连接的用户，并对包含该用户层权限信息的结构和变量进行初始化，入口是`sql/sql_parse.cc`中的`check_connection()`，有关函数有：

* `sql/sql_acl.ccc`中的`all_check_host()`和`all_getroot()`
* `sql/password.cc`中的`create_random_string()`
* `sql/sql_parse.cc`中的`check_user()`

### 6.访问控制

acl是Access Control List的缩写，即访问控制列表，所以该模块的代码主要在`sql/sql_acl.cc`中，有关方法如下：

* `check_access()`
* 位于`sql/sql_parse.cc`中的`check_table_access()`
* `check_grant_column()`
* `all_get()`

### 6.解析器

负责解析查询并生成解析树，入口点是`sql/sql_parse.cc`中的`mysql_parse()`，该函数负责执行某些初始化函数，然后调用`sql/sql_yacc.cc`中的`yyparse()`，由`sql/sql_yacc.yy`中的[GNU Bison](https://www.gnu.org/software/bison/)生成。有关文件如下：

* `sql/gen_lex_hash.cc`
* `sql/lex`
* `sql/lex_symbol.h`
* `sql/lex_hash.h`
* `sql/sql_lex.h`
* `sql/sql_lex.cc`
* `sql/item_*.h`
* `sql/item_*.cc`

### 7.命令调度

负责将请求转发给知道如何处理这些请求的较低层次模块，包括`sql/sql_parse.cc`中的两个函数：`do_command()`和`dispatch_command()`。

### 8.查询高速缓存

缓存查询结果并提供查询缓存功能，有关方法有`sql/sql_cache.cc`中的：

* `Query_Cache::store_query()`
* `Query_Cache::send_result_to_client()`

### 9.优化器

负责`explain`、改写sql等相关执行计划的功能。入口是`sql/sql_select.cc`中的`mysql_select()`，相关方法如下：

* `JOIN::prepare()`
* `JOIN::optimize()`
* `JOIN::exec()`
* `make_join_statistics()`
* `find_best_combination()`
* `Optimize_cond()`
* `sql/opt_range.cc`中的`SQL_SELECT::test_quick_select()`

### 10.表管理器

负责创建、读取和修改表定义文件（.frm扩展名）、维护表描述符高速缓存、表管理锁，有关方法如下：

* `sql/table.cc`中的`openfrm()`
* `sql/unireg.cc`中的`mysql_create_frm()`
* `sql/sql_base.cc`中的`open_table()`、`open_tables()`、`open_ltable()`
* `sql/lock.cc`中的`mysql_lock_table()`

### 11.表修改

负责表的创建、修改、重命名、移除、更新、插入等操作，相关方法如下：

* `sql/sql_update.cc`中的`mysql_update()`和`mysql_multi_update()`
* `sql/sql_insert.cc`中的`mysql_insert()`
* `sql/sql_table.cc`中的`mysql_create_table()`、`mysql_alter_table()`、`mysql_rm_table()`
* `sql/sql_delete.cc`中的`mysql_delete.cc`

### 12.表维护

负责表维护工作，包括检查、修复、备份、恢复、优化（碎片整理）及解析（更新键分布统计），代码位于`sql/sql_table.cc`中，核心方法是`mysql_admin_table()`，其他包括：

* `mysql_check_table()`
* `mysql_repair_table()`
* `mysql_backup_table()`
* `mysql_restore_table()`
* `mysql_optimize_table()`
* `mysql_analyze_table()`

### 13.状态报告

处理`show`命令，代码在`sql/sql_show.cc`中，包括：

* `mysql_list_processes()`
* `mysqld_show()`
* `mysqld_show_create()`
* `mysqld_show_fields()`
* `mysqld_show_open_tables()`
* `mysqld_show_warnings()`
* `sql/slave.cc`中的`show_master_info()`
* `sql/sql_repl.cc`中的`show_binlog_info()`

### 14.抽象存储引擎接口（表处理器）

本模块实际上是一个名为`handler`的抽象类和一个名为`handlerton`的结构，代码在`sql/handler.h`和`sql/handler.cc`中。

### 15.表存储引擎的实现（MyISAM、InnoDB、MEMORY）

每个存储引擎都通过扩展`handler`类提供的操作接口，相关方法如下：

* `sql/ha_myisam.h`和`sql/ha_myisam.cc`
* `sql/ha_innodb.h`和`sql/ha_innodb.cc`
* `sql/ha_heap.h`和`sql/ha_heap.cc`
* `sql/ha_ndbcluster.h`和`sql/ha_ndbcluster.cc`
* `myisam/`
* `innobase/`
* `heap/`
* `ndb/`

### 16.日志记录

负责较高层次的日志维护，代码在`sql/log.cc`和`sql/log_event.cc`（二进制日志）

### 17.主从复制

负责主从复制功能，相关代码如下：

* `sql/sql_repl.cc`中的`mysql_binlog_send()`
* `sql/slave.cc`中的`handle_slave_io()`和`handle_slave_sql()`

### 18.客户端/服务端协议API

负责创建、读取、解析和发送协议包，代码在`sql/protocol.cc`、`sql/protocol.h`、`sql/net_serv.cc`中。相关类有`Protocal`、`Protocol_simple`、`Protocol_prep`、`Protocol_cursor`，有关函数如下：

* `sql/net_serv.cc`中的`my_net_read()`和`my_net_write()`
* `sql/protocol.cc`中的`net_store_data()`和`send_ok()`和`send_error()`

### 19.低层次网络IO API

低层次网络IO API提供了低层次网络IO和SSL会话抽象，代码在`vio/`目录下，方法都是以`vio_`开头

### 20.核心API

提供了可移植的文件IO、内存管理、字符串操作、文件系统导航、格式化打印、丰富的数据结构和算法集，代码在`mysys/`和`strings/`目录下。