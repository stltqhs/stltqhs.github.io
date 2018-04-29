---
title: Mysql优化 - 查询优化
date: 2018-04-01 11:22:27
tags: mysql
---
### where子句优化

`mysql`为`where`子句优化做了：

* 去掉没用的条件
* 常量替换和移除
* 索引使用的常量只会计算一次
* `count(*)`的优化
* 联表时优先读取常量表



联表查询时，当`order by`和`group by`的字段都来自同一个表，那么该表将会先查询，如果它们的字段来自不同表，`mysql`将会建立**临时表**。

### 范围优化

`范围条件`的定义：

* 对于BTREE和HASH索引，常量比较使用__=,<=>,IN(),IS NULL,IS NOT NULL__
* 对于BTREE索引，常量比较使用__>,<,>=,<=,BTWEEN,!=,<>,LIKE(不是以通配符开头的字符常量)__
* 使用`OR`或者`AND`结合多个`范围条件`

范围查找的示例：

```sql
SELECT * FROM t1
  WHERE key_col > 1
  AND key_col < 10;

SELECT * FROM t1
  WHERE key_col = 1
  OR key_col IN (15,18,20);

SELECT * FROM t1
  WHERE key_col LIKE 'ab%'
  OR key_col BETWEEN 'bar' AND 'foo';
```

mysql执行范围优化的过程是先去掉`WHERE`子句非`范围条件`和非索引字段的过滤条件，然后合并范围条件得到最后的查询条件，查询条件的字段需要命中索引，直接从索引中过滤数据，由于这个查询条件没有`WHERE`子句的条件严格，可能会获得更多数据，此时需要使用`WHERE`子句来过滤查询到的数据，最后得到满足整个`WHERE`子句条件的数据（`using where`）。

### 索引合并优化

单表查询中合并范围查找的数据，合并时会产生`unions`,`intersections`,`unions-of-intersections`，即交集、并集。

使用了索引合并的优化示例：

```sql
SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;

SELECT * FROM tbl_name
  WHERE (key1 = 10 OR key2 = 20) AND non_key = 30;

SELECT * FROM t1, t2
  WHERE (t1.key1 IN (1,2) OR t1.key2 LIKE 'value%')
  AND t2.key1 = t1.some_col;

SELECT * FROM t1, t2
  WHERE t1.key1 = 1
  AND (t2.key1 = t1.some_col OR t2.key2 = t1.some_col2);
```

### Pushdown优化

mysql分为服务层和存储引擎层，对于查询来说，mysql首先将部分sql交给存储引擎，存储引擎将查询到的数据交给mysql服务层再次过滤数据。常见的Pushdown优化有Index Pushdown和Engine Pushdown。

Index Pushdown示例：

```sql
-- 假设创建的索引是INDEX (zipcode, lastname, firstname)
SELECT * FROM people
  WHERE zipcode='95054'
  AND lastname LIKE '%etrunia%'
  AND address LIKE '%Main Street%';
-- 由于lastname是以通配符开头的查询，当没有Index Pushdown时，存储引擎将先查询出所有的zipcode，然后将数据交给mysql服务层过滤，当存在Index Pushdown时，存储引擎自己就可以在索引中来过滤数据。
```

Engine Pushdown目前只能适用于NDB存储引擎。

### 联表优化

```C
for each row in t1 matching range {
  for each row in t2 matching reference key {
    for each row in t3 {
      if row satisfies join conditions, send to client
    }
  }

```

带join buffer时：

```c
for each row in t1 matching range {
  for each row in t2 matching reference key {
    store used columns from t1, t2 in join buffer
    if buffer is full {
      for each row in t3 {
        for each t1, t2 combination in join buffer {
          if row satisfies join conditions, send to client
        }
      }
      empty join buffer
    }
  }
}

if buffer is not empty {
  for each row in t3 {
    for each t1, t2 combination in join buffer {
      if row satisfies join conditions, send to client
    }
  }
}
```

### MRR优化

该优化本质是减少随机读，通过将存储引擎层的查询到的索引数据按照主键排序，回表查询时就可以表现出顺序读。可以参考[这里](http://mysql.taobao.org/monthly/2016/01/04/)。

### Order by优化

如果`order by`无法命中索引，mysql将使用文件排序，即filesort。所以提高order by的效率，就需要使得order by命中索引。按照btree的结构，命中索引的要求是按照最左前缀匹配，当遇到范围查询时，后续的索引字段不会再考虑使用索引，假设索引为`create index idx on t(key_col1,key_col2,key_col3)`，命中索引的排序的情况如下：

```sql
select * from t where key_col1 = 1 order by key_col2;
select * from t where key_col1 = 2 and key_col2 > 3 order by key_col2;
```

不会命中索引的排序如下：

```sql
select * from t where key_col1 = 1 order by key_col2 desc, key_col3 asc;
select * from t where key_col1 = 1 and key_col2 > 3 order by key_col3;
select * from t where key_col1 = 1 order by abs(key_col2);
select * from t where kehy_col1 = 1 order by -key_col2;
```

对于`group by`操作，mysql默认都会做排序操作，可以通过`order by null`来取消排序，如：

```sql
select a,count(*) from bar group by a order by null;
```

对于filesort的优化，如果是数据量少是，会使用sort buffer来排序，否在还是磁盘临时表排序。

### Group by优化

1.松散索引扫描（Loose Index Scan）

