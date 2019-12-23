---
layout:     post
title:      "PostgreSQL分表实践"
subtitle:   "两种分表策略的使用"
date:       2019-12-20
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - PostgreSQL
    - Database
    - Translation
---

> "PostgreSQL分表实践"

### 概述

分表指的是将一个大表分割成主表加上若干小的表，主表不存储数据，是逻辑表。子表是存储数据的物理表。通常在数据库表数据膨胀到一定程度的时候就需要考虑分表，尤其是经常访问的数据库表，分表可以带来以下好处：

- 一般情况下查询效率会提升，特别是经常访问的表被拆分成小的表之后，而且小的表索引也会变小，查询会更快；
- 索引变小，会提升写入，更新和删除效率，因为大表的索引通常会非常大，写入，更新和删除导致的索引更新会比较耗时；
- 不常用的数据可以转移到廉价的存储媒介中，热点数据可以放在性能比较好的存储媒介中。

分表的时机取决于应用程序的实际情况，一般是单表非常大的时候采取分表。

PostgreSQL在PostgreSQL 10引用分表功能，PostgreSQL10支持的分表策略是

- 区域分区(Range Partition)：根据某一列或者多列的值划分范围，
- 列表分区(List Partition)：表按照一个给定的列表来分区

### 软件版本

- PostgreSQL 10

### 如何分区

根据某一列或者多列来划分表的列叫做分区键(partition key)。分表之后，有一个主表，但是主表不存储数据，向表格中插入数据的时候看起来是向主表写数据，但是PostgreSQL会根据分区策略将数据路由到真正要写入数据的表。

#### Range Partition

现在我们需要构造一张日志表，记录应用的状态和时间等信息，随着时间积累表里面的数据越来越大，而且历史日志对于我们来说也没有意义，我们只想保留3个月之内的数据，所以一开始需要手动清理旧数据。

- 表的结构如下

```sql
CREATE TABLE log (
    msg             varchar(512) not null,
    logdate         date not null,
    instance_id     int not null,
    app_endpoint    varchar(256) not null
);
```

- 将表格声明成分区表
在建表语句后面加上partition by声明这是一个分区表，然后加上一个分区方法range以及分区键(logdate)。

```sql
CREATE TABLE log (
    msg             varchar(512) not null,
    logdate         date not null,
    instance_id     int not null,
    app_endpoint    varchar(256) not null
)PARTITION BY RANGE (logdate);
```

- 创建子表
子表需要声明隶属于哪个分区表以及分区键对应的区间，注意区间不能重合否则会报错。

```sql
create table logy2019m10 PARTITION of log for values from ('2019-10-01') to ('2019-11-01');
create table logy2019m11 PARTITION of log for values from ('2019-11-01') to ('2019-12-01');
create table logy2019m12 PARTITION of log for values from ('2019-12-01') to ('2020-01-01');
```

- 在子表创建索引
子表上可以分别创建不同的索引，也可以在主表创建索引，主表创建的索引会级联更新到所有子表。

```sql
create index on logy2019m10 (logdate);
create index on logy2019m11 (logdate);
create index on logy2019m12 (logdate);
```

- 确保constraint_exclusion配置是打开状态。这个配置在postgresql.conf中修改
- 创建完成之后，在postgre命令行执行\d+ log来查看表的详细信息

```sql
postgres=# \d+ log
                                               Table "public.log"
    Column    |          Type          | Collation | Nullable | Default | Storage  | Stats target | Description
--------------+------------------------+-----------+----------+---------+----------+--------------+-------------
 msg          | character varying(512) |           | not null |         | extended |              |
 logdate      | date                   |           | not null |         | plain    |              |
 instance_id  | integer                |           | not null |         | plain    |              |
 app_endpoint | character varying(256) |           | not null |         | extended |              |
Partition key: RANGE (logdate)
Partitions: logy2019m10 FOR VALUES FROM ('2019-10-01') TO ('2019-11-01'),
            logy2019m11 FOR VALUES FROM ('2019-11-01') TO ('2019-12-01'),
            logy2019m12 FOR VALUES FROM ('2019-12-01') TO ('2020-01-01')
```

- 写入一些数据

```sql
insert into log values('access api', '2019-10-11 00:00:00', 12, '/api/some');
insert into log values('access api', '2019-11-11 00:00:00', 21, '/api/some');
insert into log values('access api', '2019-12-11 00:00:00', 4, '/api/some');
insert into log values('access api', '2019-10-11 00:00:00', 464, '/api/some');
insert into log values('access api', '2019-11-11 00:00:00', 24, '/api/some');
insert into log values('access api', '2019-10-11 00:00:00', 8, '/api/some');
insert into log values('access api', '2019-10-11 00:00:00', 9, '/api/some');
```

- 确认主表里面没有数据

```sql
postgres=# select count(*) from only log;
 count
-------
     0
(1 row)

postgres=# select count(*) from log;
 count
-------
     7
(1 row)
```

我们如果带上only关键字查询主表的数据是没有数据的，不带上only关键字会查询所有子表的数量。

- 确认子表的数据

```sql
postgres=# select * from logy2019m10;
    msg     |  logdate   | instance_id | app_endpoint
------------+------------+-------------+--------------
 access api | 2019-10-11 |          12 | /api/some
 access api | 2019-10-11 |         464 | /api/some
 access api | 2019-10-11 |           8 | /api/some
 access api | 2019-10-11 |           9 | /api/some
(4 rows)

postgres=# select * from logy2019m11;
    msg     |  logdate   | instance_id | app_endpoint
------------+------------+-------------+--------------
 access api | 2019-11-11 |          21 | /api/some
 access api | 2019-11-11 |          24 | /api/some
(2 rows)

postgres=# select * from logy2019m12;
    msg     |  logdate   | instance_id | app_endpoint
------------+------------+-------------+--------------
 access api | 2019-12-11 |           4 | /api/some
(1 row)
```

### List Partition

现在我们想要创建一张日志表，按照类型把数据存储到不同的表，比如info级别的日志存储到log_info表

```sql
--定义主表
CREATE TABLE log_by_type (
    msg             varchar(512) not null,
    logdate         date not null,
    instance_id     int not null,
    log_type    varchar(256) not null
)PARTITION BY LIST (log_type);
```

查看主表信息可以看到

```sql
postgres=# \d+ log_by_type
                                          Table "public.log_by_type"
   Column    |          Type          | Collation | Nullable | Default | Storage  | Stats target | Description
-------------+------------------------+-----------+----------+---------+----------+--------------+-------------
 msg         | character varying(512) |           | not null |         | extended |              |
 logdate     | date                   |           | not null |         | plain    |              |
 instance_id | integer                |           | not null |         | plain    |              |
 log_type    | character varying(256) |           | not null |         | extended |              |
Partition key: LIST (log_type)
Number of partitions: 0
```

现在我们创建一些子表，主表的log_type字段假设会填充的类型是(INFO, WARNING, ERROR)

```sql
create table log_info partition of log_by_type for values in ('Info');
create table log_warning partition of log_by_type for values in ('Warning');
create table log_error partition of log_by_type for values in ('Error');
```

现在查看log_by_type信息

```sql
postgres=# \d+ log_by_type
                                          Table "public.log_by_type"
   Column    |          Type          | Collation | Nullable | Default | Storage  | Stats target | Description
-------------+------------------------+-----------+----------+---------+----------+--------------+-------------
 msg         | character varying(512) |           | not null |         | extended |              |
 logdate     | date                   |           | not null |         | plain    |              |
 instance_id | integer                |           | not null |         | plain    |              |
 log_type    | character varying(256) |           | not null |         | extended |              |
Partition key: LIST (log_type)
Partitions: log_error FOR VALUES IN ('Error'),
            log_info FOR VALUES IN ('Info'),
            log_warning FOR VALUES IN ('Warning')
```

写入一些数据然后测试

```sql
insert into log_by_type values('start application', '2019-12-24 10:00:00', '12', 'Info');
insert into log_by_type values('init plugin', '2019-12-24 10:00:02', '12', 'Info');
insert into log_by_type values('start application in 5 seconds', '2019-12-24 10:00:05', '12', 'Info');
insert into log_by_type values('warn, method is deprecated;', '2019-12-24 10:00:10', '12', 'Warning');
insert into log_by_type values('fail to connected to database', '2019-12-24 10:01:00', '12', 'Error');
```

```sql
postgres=# select * from only log_info;
              msg               |  logdate   | instance_id | log_type
--------------------------------+------------+-------------+----------
 start application              | 2019-12-24 |          12 | Info
 init plugin                    | 2019-12-24 |          12 | Info
 start application in 5 seconds | 2019-12-24 |          12 | Info
(3 rows)

postgres=# select * from only log_warning;
             msg             |  logdate   | instance_id | log_type
-----------------------------+------------+-------------+----------
 warn, method is deprecated; | 2019-12-24 |          12 | Warning
(1 row)

postgres=# select * from only log_error;
              msg              |  logdate   | instance_id | log_type
-------------------------------+------------+-------------+----------
 fail to connected to database | 2019-12-24 |          12 | Error
(1 row)
```

### 移除分区

有两种方式来移除分区

- 删除表

直接删除子表可以移除分区表和主表的关联关系，以及删掉子表所有的数据，这种移除分区的方式会在主表加 *ACCESS EXCLUSIVE* 的锁。

```sql
drop table log_warning;
```

- 删除分区关系

可以使用detach的功能移除主表和子表的关联关系，这样子可以保留子表，方便做后续的处理，一般推荐用这一种方式来删除分区关系。

```sql
ALTER TABLE log_by_type DETACH PARTITION log_error;
```

### 注意

- 分区表的主表和普通表不能相互转换，但是可以在分区表建好之后添加新的分区表和删除已有的分区表。分区表的子表可以转换成普通的数据库表。
- 普通的表无法继承分区表，反之亦然。

### 引用

- PostgreSQL [官方文档](https://www.postgresql.org/docs/10/ddl-partitioning.html#DDL-PARTITIONING-OVERVIEW)
- [Hash partition介绍](https://blog.dbi-services.com/hash-partitioning-in-postgresql-11/)
