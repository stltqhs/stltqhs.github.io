---
title: mysql_performace_analyze
date: 2019-04-06 19:50:34
tags:
---

# 1.查询锁等待的sql

```
select requesting_trx_id,requested_lock_id,t1.trx_query as request_trx_query,blocking_trx_id,blocking_lock_id,t2.trx_query as blocking_trx_query from innodb_lock_waits left join innodb_trx t1 on requesting_trx_id=t1.trx_id left join innodb_trx t2 on blocking_trx_id = t2.trx_id\G
```

# 2.profile

```
show profiles;
set profiling=1;// 默认关闭
show profile [source] for query 2;
show profile block io,cpu for query 2;
```

还可以查询PROFILING视图

# 3.查看临时表

```
> show global status like '%tmp%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Created_tmp_disk_tables | 5155228 |
| Created_tmp_files       | 8261    |
| Created_tmp_tables      | 7046409 |
+-------------------------+---------+
Created_tmp_disk_tables / Created_tmp_tables * 100% <= 25%（该值不清楚）
```

# 4.查看innodb状态

```
show engine innodb status\G
```

可以看到IO情况，事务加锁的情况等。

# 5.查看表状态

```
show table status like 't_charging'\G
*************************** 1. row ***************************
           Name: t_charging
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 5926799
 Avg_row_length: 323
    Data_length: 1917845504
Max_data_length: 0
   Index_length: 1267597312
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2017-09-07 10:49:00
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options:
        Comment: 充电记录
1 row in set (0.00 sec)
```

# 6.开启optimizer_trace

```
set optimizer_trace="enabled=on";
select sum(length(val)) from t where j = 2 and i between 100 and 200;
select * from information_schema.optimizer_trace;
```

