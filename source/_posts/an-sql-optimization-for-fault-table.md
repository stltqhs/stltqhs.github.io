---
title: 记一次sql优化的过程
date: 2018-04-16 13:34:43
tags: mysql
---

### 数据信息

t_fault表用来记录系统故障数据，目前有16233661条记录，表结构如下：

```sql
CREATE TABLE `t_fault` (
  `id` varchar(36) NOT NULL COMMENT '主键ID',
  `pile_id` varchar(19) DEFAULT NULL,
  `report_type` int(2) DEFAULT '0' COMMENT '上报来源，1:蓝牙；2:gprs桩;',
  `fault_code` int(2) DEFAULT NULL COMMENT '故障代码;0:急停故障；1:电表故障；2:接触器故障；3：读卡器故障；4:内部过温故障；5:连接器故障；6:绝缘故障；7:其他；',
  `err_code` int(2) DEFAULT NULL COMMENT '错误代码；0:电流异常；1:电压异常；2:其他',
  `err_status` int(2) DEFAULT NULL COMMENT '错误状态；1:故障；2：错误',
  `solve_status` int(2) DEFAULT NULL COMMENT '解决状态；1:未解决；2：已解决',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `record_time` datetime DEFAULT NULL COMMENT '故障时间戳',
  `fault_type` int(8) DEFAULT NULL COMMENT '错误类型，1<<1:交流过压；2:交流欠压；3:交流过流；4:交流漏电；5:交流短路',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  `solve_time` datetime DEFAULT NULL COMMENT ' 故障解决时间',
  `operator_id` varchar(19) DEFAULT NULL,
  `inter_no` smallint(6) DEFAULT '0' COMMENT '充电口，0表示所有充电口'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='故障与报警';

```

索引如下：

```Sql
mysql> show index from t_fault;
'+---------+------------+------------------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table   | Non_unique | Key_name                     | Seq_in_index | Column_name  | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+---------+------------+------------------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t_fault |          0 | PRIMARY                      |            1 | id           | A         |    15241935 |     NULL | NULL   |      | BTREE      |         |               |
| t_fault |          1 | i_fault_common               |            1 | err_status   | A         |           2 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_common               |            2 | report_type  | A         |           2 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_common               |            3 | solve_status | A         |         322 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_common               |            4 | fault_code   | A         |        1932 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_common               |            5 | record_time  | A         |    15241935 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            1 | pile_id      | A         |      200551 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            2 | err_status   | A         |      362903 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            3 | report_type  | A         |      224146 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            4 | solve_status | A         |      401103 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            5 | fault_code   | A         |      435483 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_pile_common          |            6 | record_time  | A         |    15241935 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            1 | operator_id  | A         |      105116 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            2 | err_status   | A         |       90188 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            3 | report_type  | A         |       86112 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            4 | solve_status | A         |      105846 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            5 | fault_code   | A         |       95262 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_common      |            6 | record_time  | A         |    15241935 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            1 | operator_id  | A         |       91269 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            2 | pile_id      | A         |      211693 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            3 | err_status   | A         |      461876 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            4 | report_type  | A         |      311059 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            5 | solve_status | A         |      317540 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            6 | fault_code   | A         |      282258 |     NULL | NULL   | YES  | BTREE      |         |               |
| t_fault |          1 | i_fault_operator_pile_common |            7 | record_time  | A         |    15241935 |     NULL | NULL   | YES  | BTREE      |         |               |
+---------+------------+------------------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
25 rows in set (0.00 sec)
```

表状态如下：

```sql
mysql> show table status like 't_fault'\G
*************************** 1. row ***************************
           Name: t_fault
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 15241935
 Avg_row_length: 128
    Data_length: 1965031424
Max_data_length: 0
   Index_length: 7746813952
      Data_free: 6291456
 Auto_increment: NULL
    Create_time: 2018-03-26 21:02:22
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 故障与报警
1 row in set (0.01 sec)
```

### 表结构优化

t_fault表的主键是`varchar(36)`，生成的方式是随机。如果主键是随机生成，当插入一条新记录时，由于mysql的聚集索引的原因，要求主键必须是逻辑上顺序排列。我们知道，对于查询来说，顺序读是非常快的。当插入一条无序记录（指新记录的主键与表中记录的主键没有顺序关系）时，由于要保证主键逻辑有序，因此需要移动数据，使得记录物理上无序，这种情况称为[页分裂](https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/)（page split），导致数据读取性能下降。所以第一步优化是将主键设计为严格单调增，为了简便，这里将主键id设计为自增类型。其次，如果能确定某些列不会存在null的情况时就需要将该列设计为`not null`。新的表名为t_fault_new，表结构如下：

```sql
CREATE TABLE `t_fault_new` (
  `id` bigint NOT NULL primary key auto_increment COMMENT '主键ID',
  `pile_id` varchar(19) DEFAULT NULL,
  `report_type` int(2) DEFAULT '0' not null COMMENT '上报来源，1:蓝牙；2:gprs桩;',
  `fault_code` int(2) DEFAULT 20 not null COMMENT '故障代码;20:无故障;0:急停故障；1:电表故障；2:接触器故障；3：读卡器故障；4:内部过温故障；5:连接器故障；6:绝缘故障；7:其他；',
  `err_code` int(2) DEFAULT 20 not null COMMENT '错误代码；0:电流异常；1:电压异常；2:其他;20:无错误',
  `err_status` int(2) not null DEFAULT 1 COMMENT '错误状态；1:故障；2：错误',
  `solve_status` int(2) not null DEFAULT 1 COMMENT '解决状态；1:未解决；2：已解决',
  `create_time` datetime not null COMMENT '创建时间',
  `record_time` datetime not null COMMENT '故障时间戳',
  `fault_type` int(8) not null DEFAULT 0 COMMENT '错误类型，1<<1:交流过压；2:交流欠压；3:交流过流；4:交流漏电；5:交流短路',
  `update_time` datetime default null COMMENT '更新时间',
  `solve_time` datetime DEFAULT NULL COMMENT ' 故障解决时间',
  `operator_id` varchar(19) not null DEFAULT '0' COMMENT '运营商ID',
  `inter_no` smallint(6) not null DEFAULT '0' COMMENT '充电口，0表示所有充电口'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='故障与报警';
```

表状态如下：

```sql
mysql> show table status like 't_fault_new'\G
*************************** 1. row ***************************
           Name: t_fault_new
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 16157511
 Avg_row_length: 97
    Data_length: 1570766848
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: 16252681
    Create_time: 2018-04-16 11:43:02
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 故障与报警
```

可以看到Avg_row_length从127降到了97，数据量减少了，这样innodb buffer可以缓存更多数据。

将旧表的数据导入到新表中：

```sql
insert into t_fault_new(pile_id,report_type,fault_code,err_code, solve_status, create_time,record_time,fault_type,update_time,solve_time,operator_id,inter_no)
select pile_id,report_type,fault_code,err_code, solve_status, create_time,record_time,ifnull(fault_type, 0),update_time,solve_time,operator_id,inter_no from t_fault
where err_status is not null;
```

创建相关索引：

```sql
create index i_fault_common on t_fault_new(err_status,report_type,solve_status,fault_code,record_time);
create index i_fault_pile_common on t_fault_new(pile_id,err_status,report_type,solve_status,fault_code,record_time);

create index i_fault_operator_common on t_fault_new(operator_id,err_status,report_type,solve_status,fault_code,record_time);
create index i_fault_operator_pile_common on t_fault_new(operator_id,pile_id,err_status,report_type,solve_status,fault_code,record_time);
```

先查询旧表数据：

```sql
mysql> select count(*) as num,fault_code from t_fault where err_status = 1 and report_type in (1,2) and solve_status in (1,2) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----------+------------+
| num      | fault_code |
+----------+------------+
|     6202 |          0 |
|     1532 |          1 |
|     3199 |          2 |
|      559 |          3 |
|       31 |          4 |
|      115 |          5 |
|    20186 |          6 |
|     1966 |          7 |
| 16143595 |          8 |
+----------+------------+
9 rows in set (17 min 49.43 sec)
```

查询耗时17分钟，再次查询新表数据：

```sql
mysql> select count(*) as num,fault_code from t_fault_new where err_status = 1 and report_type in (1,2) and solve_status in (1,2) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----------+------------+
| num      | fault_code |
+----------+------------+
|     6202 |          0 |
|     1532 |          1 |
|     3199 |          2 |
|      559 |          3 |
|       31 |          4 |
|      115 |          5 |
|    20186 |          6 |
|     1966 |          7 |
| 16143595 |          8 |
+----------+------------+
9 rows in set (13.99 sec)
```

查询时间从17分钟降到13秒。

### 索引优化

使用`explain`检查sql的查询计划：

```sql
mysql> explain select count(*) as num,fault_code from t_fault_new where err_status = 1 and report_type in (1,2) and solve_status in (1,2) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----+-------------+-------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+-------+-----------------------------------------------------------+
| id | select_type | table       | type  | possible_keys                                                                           | key            | key_len | ref  | rows  | Extra                                                     |
+----+-------------+-------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+-------+-----------------------------------------------------------+
|  1 | SIMPLE      | t_fault_new | range | i_fault_common,i_fault_pile_common,i_fault_operator_common,i_fault_operator_pile_common | i_fault_common | 16      | NULL | 77796 | Using where; Using index; Using temporary; Using filesort |
+----+-------------+-------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+-------+-----------------------------------------------------------+
1 row in set (0.00 sec)
```

Extra列的解释：

* Using where：需要使用where条件来过滤数据；
* Using index：直接使用覆盖索引查询数据而不需要回表查询数据；
* Using temporary：使用临时表；
* Using filesort：需要文件排序，排序时需要使用临时表，临时表可能是内存临时表，也有可能是磁盘文件临时表，数据量决定是使用哪种。

由于该sql的where字段都可以命中索引，使用临时表的原因是mysql会将`in`拆成多个`=`的sql，然后再union，所以数据量是1 x 2 x 2 x 9 = 36，数据量不大，不会使用磁盘临时表。可以使用下面的方式证明：

(1)先查询当前会话临时表数据

```sql
mysql> show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 13    |
| Created_tmp_tables      | 9     |
+-------------------------+-------+
```

临时表数量是9，临时表文件数量是13。

(2)执行sql查询

```sql
mysql> select count(*) as num,fault_code from t_fault_new where err_status = 1 and report_type in (1,2) and solve_status in (1,2) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----------+------------+
| num      | fault_code |
+----------+------------+
|     6202 |          0 |
|     1532 |          1 |
|     3199 |          2 |
|      559 |          3 |
|       31 |          4 |
|      115 |          5 |
|    20186 |          6 |
|     1966 |          7 |
| 16143595 |          8 |
+----------+------------+
```

(3)再次查询会话临时表数据

```sql
mysql> show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 13    |
| Created_tmp_tables      | 10    |
+-------------------------+-------+
```

临时表数量增加了1，变为10，临时表文件数量依然是13，表示并没有使用磁盘临时表。

通过上面的执行计划可以看到，使用的索引长度16字节。由于业务规则，err_status，report_type和solve_status只能有2种状态，因此可以新建一个列，值就是它们的组合（即`((err_status - 1) << 2) | ((report_type - 1) << 1) | (solve_status - 1)`），这样索引就可以少2个字段，数据更加紧凑，查询速度会更快。新表结构如下：

```sql
CREATE TABLE `t_fault_new_compound` (
  `id` bigint NOT NULL primary key auto_increment COMMENT '主键ID',
  `pile_id` varchar(19) DEFAULT NULL,
  `report_type` int(2) DEFAULT 2 not null COMMENT '上报来源，1:蓝牙；2:gprs桩;',
  `fault_code` int(2) DEFAULT 20 not null COMMENT '故障代码;20:无故障;0:急停故障；1:电表故障；2:接触器故障；3：读卡器故障；4:内部过温故障；5:连接器故障；6:绝缘故障；7:其他；',
  `err_code` int(2) DEFAULT 20 not null COMMENT '错误代码；0:电流异常；1:电压异常；2:其他;20:无错误',
  `err_status` int(2) not null DEFAULT 1 COMMENT '错误状态；1:故障；2：错误',
  `solve_status` int(2) not null DEFAULT 1 COMMENT '解决状态；1:未解决；2：已解决',
  `create_time` datetime not null COMMENT '创建时间',
  `record_time` datetime not null COMMENT '故障时间戳',
  `fault_type` int(8) not null DEFAULT 0 COMMENT '错误类型，1<<1:交流过压；2:交流欠压；3:交流过流；4:交流漏电；5:交流短路',
  `update_time` datetime default null COMMENT '更新时间',
  `solve_time` datetime DEFAULT NULL COMMENT ' 故障解决时间',
  `operator_id` varchar(19) not null DEFAULT '0' COMMENT '运营商ID',
  `inter_no` smallint(6) not null DEFAULT '0' COMMENT '充电口，0表示所有充电口',
  `compound` int(2) not null comment 'err_status,report_type,solve_time组合，取值为 ((err_status - 1) << 2) | ((report_type - 1) << 1) | (solve_status - 1)'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='故障与报警';
```

导入数据：

```sql
insert into t_fault_new_compound(pile_id,report_type,fault_code,err_code, solve_status, create_time,record_time,fault_type,update_time,solve_time,operator_id,inter_no, compound)
select pile_id,report_type,fault_code,err_code, solve_status, create_time,record_time,ifnull(fault_type, 0),update_time,solve_time,operator_id,inter_no,((err_status - 1) << 2) | ((report_type - 1) << 1) | (solve_status - 1) from t_fault
where err_status is not null;
```

建立索引：

```sql
create index i_fault_common on t_fault_new_compound(compound,fault_code,record_time);
create index i_fault_pile_common on t_fault_new_compound(pile_id,compound,fault_code,record_time);

create index i_fault_operator_common on t_fault_new_compound(operator_id,compound,fault_code,record_time);
create index i_fault_operator_pile_common on t_fault_new_compound(operator_id,compound,fault_code,record_time);
```

查询执行计划：

```sql
mysql> explain select count(*) as num,fault_code from t_fault_new_compound where compound in (0,1,2,3,4,5,6,7) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----+-------------+----------------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+--------+-----------------------------------------------------------+
| id | select_type | table                | type  | possible_keys                                                                           | key            | key_len | ref  | rows   | Extra                                                     |
+----+-------------+----------------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+--------+-----------------------------------------------------------+
|  1 | SIMPLE      | t_fault_new_compound | range | i_fault_common,i_fault_pile_common,i_fault_operator_common,i_fault_operator_pile_common | i_fault_common | 8       | NULL | 136296 | Using where; Using index; Using temporary; Using filesort |
+----+-------------+----------------------+-------+-----------------------------------------------------------------------------------------+----------------+---------+------+--------+-----------------------------------------------------------+
```

可以看到使用的索引长度为8，少了一半。执行查询：

```sql
mysql> select count(*) as num,fault_code from t_fault_new_compound where compound in (0,1,2,3,4,5,6,7) and fault_code in (0,1,2,3,4,5,6,7,8) group by fault_code;
+----------+------------+
| num      | fault_code |
+----------+------------+
|     6202 |          0 |
|     1532 |          1 |
|     3199 |          2 |
|      559 |          3 |
|       31 |          4 |
|      115 |          5 |
|    20186 |          6 |
|     1966 |          7 |
| 16143595 |          8 |
+----------+------------+
9 rows in set (9.69 sec)
```

查询速度从13秒降到9秒，又快了4秒。

通过上面的结果可以看到，fault_code为8的结果有16143595条，改写sql，单独查询fault_code为8，然后union整个结果，如下：

```sql
mysql> select sum(num) as num,fault_code from( select count(*) as num,fault_code from t_fault_new_compound where compound in(0,1,2,3,4,5,6,7) and fault_code in (0,1,2,3,4,5,6,7) group by fault_code union select count(*) as num,fault_code from t_fault_new_compound where compound in (0,1,2,3,4,5,6,7) and fault_code = 8 ) t where fault_code is not null group by fault_code;
+----------+------------+
| num      | fault_code |
+----------+------------+
|     6202 |          0 |
|     1532 |          1 |
|     3199 |          2 |
|      559 |          3 |
|       31 |          4 |
|      115 |          5 |
|    20186 |          6 |
|     1966 |          7 |
| 16143595 |          8 |
+----------+------------+
9 rows in set (5.94 sec)
```

查询速度从9秒再次降到6秒，又快了3秒。

### count优化

对于innodb存储引擎来说，count查询不是常量查询，它需要遍历所有数据才能得到结果。对于数据量达到几千万的情况下，sql优化会达到极限，但是依然不能实现秒级查询，此时可以建立一个统计表，count时直接查询统计表即可。