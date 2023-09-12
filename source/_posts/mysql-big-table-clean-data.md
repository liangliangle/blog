title: MySQL超大表删除数据
tags: []
categories: []
date: 2023-09-12 16:10:34
---
# 背景

笔者在公司负责公司的OpenAPI应用，估产生了调用审计的需求。对于存储这些AccessLog，虽然业界有很合适的架构和理论，奈何我司已成本优先，且作为toB的项目，调用量并不算特别大，每天也就2G左右的AccessLog产生。业务特征又导致整个订单的周期非常长，最少要保存1年以上的记录，以备排查问题所用（扯皮甩锅）。所以使用了大磁盘的MySQL直接存储。其表结构如下：

```sql
CREATE TABLE `access_log` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `trace_id` varchar(50) DEFAULT NULL COMMENT 'traceId',
  `api_name` varchar(50) DEFAULT NULL COMMENT 'api名称',
  `api_context` longtext COMMENT '调用正文',
  `api_result` longtext COMMENT '返回正文',
  `is_success` tinyint(4) DEFAULT NULL COMMENT '是否成功',
  `time_consuming` bigint(20) DEFAULT NULL COMMENT '消耗时间(毫秒)',
  PRIMARY KEY (`id`),
  KEY `idx_trace_id` (`trace_id`),
  KEY `idx_gmt_create` (`gmt_create`,`api_name`),
  KEY `idx_api_name` (`api_name`,`gmt_create`,`is_success`)
) ENGINE=InnoDB COMMENT='流量入口-api记录'
```

而随着业务发展，需要接入的系统也越来越多，甚至有定时任务需要轮询接口，导致日志量暴增。达到了日均40G的地步。单表最大数据量在600G

> 在此期间，使用了各种手段优化写入量。忽略某些超高频又不影响业务的API。只记录某些接口错误调用的日志等等。

至于为什么不采用以月为后缀的动态表，涉及到我司DB管控问题。该方案一直无法通过。

问题拖到现在，涉及两张表：accessLog 600G， errorLog 200G。存储已经达到了物理机的上限，扩容就需要进行数据库迁移，最少需要一周时间提前做数据迁移。

# 要求

**现状：**

1. 两张超大表：accessLog 600G， errorLog 200G。
2. 近两周暴增了400G的占用
3. 整个机器的存储空间已经达到91%。剩余90G左右空间。

**要求：**

1. 线上做到写入无影响。
2. 数据库不能因此宕机。

# 技术方案

因为涉及的表过大，操作必须谨慎，不能产生临时表，表重建等隐形操作。


**流程如下：**

1. 清理errorLog表，只留存3天数据
2. 检查实际空间占用，确定重建表的空间安全
3. 重建该表，将可用空间提升到200G左右
4. 归档accessLog表
5. 清理其超高频的API日志
6. 按照日期保留3个月的日志
7. 检查实际空间占用，确定重建时空间安全。
8. 说服DBA，同意基于日期的动态表方案。


## 清理数据

清理数据相对简单，只需要加上主键排序+limit即可

```sql
delete from table_name where *** order by id limit 10000;
```

但是在清理过程中需要注意binlog文件大小，因为binlog一般配置了按天保存文件，可能导致binlog打满磁盘的情况。

**查看binlog文件大小**

```sql
show binary logs;


| Log_name         | File_size|
| mysql-bin.003312 | 15178497 |
| mysql-bin.003313 | 3841846  |
| mysql-bin.003314 | 12789083 |
| mysql-bin.003315 | 9800029  |
```

**查看正在写入的Binlog文件**

```sql
show master status;


| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| mysql-bin.003315 | 10479164 |              |                  | ****              |
```

执行过程中，需要暂停一段时间执行，方便binlog切换等操作，否则可能导致实例一直繁忙状态无法生成新的binlog文件

**清理历史binlog文件**

```sql
PURGE MASTER LOGS TO 'mysql-bin.003315';

--- 清理 mysql-bin.003315 之前的文件，并不会清理当前文件。
```

清理完成后可以再次查看binlog文件

## 检查实际空间大小

因为MySQL的物理删除本质上也是逻辑删除，所以空间并不会被释放，需要检查实际的空间占用，保证重建表时，空间在安全范围内。

```sql
SELECT CONCAT(table_schema, '.', table_name)                      AS 'Table Name',
       CONCAT(ROUND(table_rows / 1000000, 4), 'M')                AS 'Number of Rows',
       CONCAT(ROUND(data_length / (1024 * 1024 * 1024), 4), 'G')  AS 'Data Size',
       CONCAT(ROUND(index_length / (1024 * 1024 * 1024), 4), 'G') AS 'Index Size',
       CONCAT(ROUND((data_length + index_length) / (1024 * 1024 * 1024), 4), 'G')
                                                                  AS 'Total',
       CONCAT(ROUND((data_free) / (1024 * 1024 * 1024), 4), 'G')
                                                                  AS 'Free Size'
FROM information_schema.TABLES
WHERE table_schema LIKE 'database_name';
```

替换掉database_name为数据库名称，则可以看到表有效数据的占用大小，可释放空间大小等等。

如`Total`>数据库剩余空间，则重建就是安全的。

## 重建表

重建表使用一下语句：

```sql
OPTIMIZE TABLE `table_name`;
```

该命令会重建表，重建过程可以可以参考 [13 为什么表数据删掉一半，表文件大小不变？](http://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/MySQL%e5%ae%9e%e6%88%9845%e8%ae%b2/13%20%20%e4%b8%ba%e4%bb%80%e4%b9%88%e8%a1%a8%e6%95%b0%e6%8d%ae%e5%88%a0%e6%8e%89%e4%b8%80%e5%8d%8a%ef%bc%8c%e8%a1%a8%e6%96%87%e4%bb%b6%e5%a4%a7%e5%b0%8f%e4%b8%8d%e5%8f%98%ef%bc%9f.md)

# 结果

**清理数据过程中**

* 180 IOPS左右浮动
* CPU在20%左右
* 磁盘空间无明显增长

**重建过程中**

* 6000左右的 IOPS，完全吃满了磁盘性能
* CPU 40%左右浮动
* 磁盘初始新增20GB，后续断崖式下降