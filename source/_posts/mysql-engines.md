---
title: MySQL引擎介绍
tags: []
categories: []
abbrlink: fcfd18c1
date: 2022-04-04 22:44:48
---
# 引擎是什么？
MySQL中的数据用各种不同的技术存储在文件(或者内存)中。这些技术中的每一种技术都使用不同的存储机制、索引技巧、锁定水平并且最终提供广泛的不同的功能和能力。通过选择不同的技术，你能够获得额外的速度或者功能，从而改善你的应用的整体功能。
在文件系统中，MySQL将每个数据库（也可以称之为schema）保存为数据目录下的一个子目录。创建表时，MySQL会在数据库子目录下创建一个和表同名的.frm文件保存表的定义。例如创建一个名为 MyTable的表，MySQL会在MyTable.frm文件中保存该表的定义。因为MySQL使用文件系统的目录和文件来保存数据库和表的定义，大小写敏感性和具体的平台密切相关。在Windows中，大小写是不敏感的；而在类Unix中则是敏感的。不同的存储引擎保存数据和索引的方式是不同的，但表的定义则是在MySQL服务层统一处理的。

## 查看支持的引擎

```sql
show engines;
```

![](https://static.lianglianglee.com/assets/tih4p92cmigv3qr02q3avl3src.png)

## 查看已有表信息

：show table status like 'user' \G

![](https://static.lianglianglee.com/assets/0aa6rm8psaif1pjt86nud1ealp.png)

 **Name**:表名 

**Engine**:存储引擎

**Row_format**:行格式

**Rows**：行数，MyISAM是准确的，InnoDB是估计值

**Avg_row_length**：平混每行字节数

**Data_length**：表数据大小

**Max_data_length**：表数据最大容量，与存储引擎有关。

**Index_length**：索引大小（字节）

**Data_free**：已分配但未使用空间

**Auto_increment**: 下一个Auto_increment值,自增主键是下一个主键的值

**Create_time**: 表创建时间
**Update_time**: 表数据最后更新时间

**Check_time**: 使用ckeck table或者myisamchk工具最后检查时间

**Collation**: 默认字符集和字符排序规则

**Checksum**: 如果启用，保存整个表的实时校验和

**Create_options**: 创建指定的其他选项

**Comment**: 其他额外信息，一般用作表备注

# InnoDB

InnoDB是MySQL默认的事务型引擎，也是最重要、最广泛的存储引擎。它的设计是用来处理大量短期事务，短期事务大部分是正常提交的，很少回滚。InnoDB的性能和自动崩溃恢复特性，使得它在非事务型存储的需求中，也很流行。除了非常特别的原因需要使用其他引擎，InnoDB也是非常好值得花时间研究的对象。

InnoDB的数据存储在表空间中，表空间是由InnoDB管理的黑盒文件系统，由一系列系统文件组成。InnoDB可以将每个表的数据和索引存放在单独的文件中。InnoDB也可以使用裸设备作为表空间存储介质。

InnoDB通过间隙锁（next-key locking）防止幻读的出现。InnoDB是基于聚簇索引建立，与其他存储引擎有很大的区别，聚簇索引对主键查询有很高的性能，不过它的二级索引（secondary index，非主键索引）必须包含主键列。所以如果主键列很大的话，索引会很大。

# MyISAM

在5.1之前，MyISAM是默认的引擎，MyISAM有大量的特心态，包括全文索引、压缩、空间函数。但是MyISAM不支持事务和行级锁，而且在崩溃后无法安全恢复。即使后续版本中MyISAM支持了事务，但是很多人的概念中依然是不支持事务的引擎。

MyISAM并不是无所事处。对于一些只读数据，或者表空间较小，可以忍受恢复操作，可以使用MyISAM。MyISAM会将表存储在两个文件中：数据文件、索引文件。分别是.MYD、.MYI扩展名。MyISAM表可以包含动态或者静态行。MySQL会根据表定义选择那种行格式。MyISAM表的行记录数，取决于磁盘空间和操作系统中的单个文件最大尺寸。

在MySQL中，默认配置只能存储256TB的数据。因为指向数据记录的指针长度是6字节。需要修改可以修改表的MAX_ROWS和AVG_ROW_LENGTH选项。两个相乘是最大的大小。会导致重建索引。

MyISAM是对整个表加锁，而不是行锁，读取的时候对表加共享锁，写入的时候加排他锁。但是在表有读取查询的同时，也可以往表内写入记录。

对于MyISAM，即使是Blob，Text等等长字段，也可以基于前500字符创建索引，MyISAM支持全文索引，这是一个基于分词创建的索引，也可以支持复杂的查询。

MyISAM可以选择延迟更新索引键，在创建表的时候指定delay_key_write选项，在每次修改执行完成时，不会立刻将修改的索引数据写入磁盘，而是写到缓存区，只有在清理缓存区或者关闭表的时候才会将索引写入磁盘。这可以极大的提升写入性能，但是在主机崩溃时会造成索引损坏，需要执行修复操作。

MyISAM另一个特性是支持压缩表。如果数据在写入后不会修改，那么这个表适合MyISAM压缩表。可以使用myisampack对MyISAM表进行打包，压缩表是不可以修改数据的。压缩表可以极大的减少磁盘占用，因此可以减少磁盘IO，提升性能，压缩表也支持索引，但是索引也是只读的。

整体来说MyISAM并没有那么不堪，但是由于没有行锁机制，所以在海量写入的时候，会导致所有查询处于Locked状态。

# 其他存储引擎

MySQL还有一些其他特殊用途的引擎，有些可能不再支持，具体支持情况参考数据库支持引擎。

## Archive

Archive引擎支持是Insert，Select操作，现在支持索引，Archive引擎会缓存所有的写，并利用zlib对写入行进行压缩，所以比MyISAM表的磁盘IO更少。但是在每次Select查询都需要执行全表扫描。所以在Archive适合日志和数据采集应用。这类应用在分析时往往需要全表扫描忙活着更快的Insert操作场景中也可以使用。

Archive引擎支持行级锁和专用的缓存区，所以可以实现高并发写入，在查询开始到返回表存在的所有行数之前，Archive会阻止其他Select执行，用来实现一致性读。另外也实现了批量写入结束前批量写入数据对读操作不可见，这种机制模仿了事务和MVCC的特性，但是Archive不是一个事务型引擎，而是针对高写入压缩做了优化的简单引擎。

## Blackhole

Blackhole没有实现任何存储机制，它会舍弃所有写入数据，不做任何保存，但是服务器会记录Blackhole表的日志，用于复制数据到备库，或者只是简单的记录到日志，这种特殊的存储引擎可以在一些特俗的复制架构和日志审核时发挥作用。但是不推荐。

## CSV

CSV引擎可以将普通的CSV文件作为MySQL表来处理，但是这种表不支持索引，CSV可以在数据库运行时拷贝或者拷出文件，可以将Excel等电子表格中的数据存储未CSV文件，然后复制到MySQL中，就能在MySQL中打开使用。同样，如果将数据写入到一个CSV引擎表，其他外部程序也可以从表的数据文件中读取CSV的数据。因此CSV可以作为数据交换机制。非常好用。

## Federated

Federated引擎是访问其他MySQL服务器的一个代理，它会创建一个到远程MySQL服务器的客户端连接，并将查询传输到远程服务器执行，然后提取或者发送需要的数据。最初设计该存储引擎是为了和企业级数据库如MicrosoftSQLServer和Oracle的类似特性竞争的，可以说更多的是一种市场行为。尽管该引擎看起来提供了一种很好的跨服务器的灵活性，但也经常带来问题，因此默认是禁用的。

## Memroy

如果需要快速地访问数据，并且这些数据不会被修改，重启以后丢失也没有关系，那么使用Memory表（以前也叫做HEAP表）是非常有用的。Memory表至少比MyISAM表要快一个数量级，因为所有的数据都保存在内存中，不需要进行磁盘I/O。Memory表的结构在重启以后还会保留，但数据会丢失。

Memroy表在很多场景可以发挥好的作用：

1. 用于查找（lookup）或者映射（mapping）表，例如将邮编和州名映射的表。
2. 用于缓存周期性聚合数据（periodicallyaggregateddata）的结果。
3. 用于保存数据分析中产生的中间数据。

Memory表支持Hash索引，因此查找操作非常快。虽然Memory表的速度非常快，但还是无法取代传统的基于磁盘的表。Memroy表是表级锁，因此并发写入的性能较低。它不支持BLOB或TEXT类型的列，并且每行的长度是固定的，所以即使指定了VARCHAR列，实际存储时也会转换成CHAR，这可能导致部分内存的浪费。如果MySQL在执行查询的过程中需要使用临时表来保存中间结果，内部使用的临时表就是Memory表。如果中间结果太大超出了Memory表的限制，或者含有BLOB或TEXT字段，则临时表会转换成MyISAM表。

## Merge

Merge引擎是MyISAM引擎的一个变种。Merge表是由多个MyISAM表合并而来的虚拟表。如果将MySQL用于日志或者数据仓库类应用，该引擎可以发挥作用。但是引入分区功能后，该引擎已经被放弃

## NDB 集群 引擎

NDB集群存储引擎，作为SQL和NDB原生协议之间的接口。MySQL服务器、NDB集群存储引擎，以及分布式的、share-nothing的、容灾的、高可用的NDB数据库的组合，被称为MySQL集群（MySQLCluster）。

# 特殊

## PERFORMANCE_SCHEMA

MySQL 5.5新增一个存储引擎：命名**PERFORMANCE_SCHEMA**  ,主要用于收集数据库服务器性能参数。MySQL用户是不能创建存储引擎为PERFORMANCE_SCHEMA的表。

performance_schema提供以下功能：

1. 提供进程等待的详细信息，包括锁、互斥变量、文件信息；
2. 保存历史的事件汇总信息，为提供MySQL服务器性能做出详细的判断；
3. 对于新增和删除监控事件点都非常容易，并可以随意改变mysql服务器的监控周期，例如（CYCLE、MICROSECOND）、

**performance_schema功能开启和部分表功能**

Performance的开启很简单，在my.cnf中[mysqld]加入performanc_schema，检查性能数据库是否启动的命令:

```
SHOW VARIABLES LIKE 'performance_schema';
```

&gt; 若是返回的 值为ON，则说明性能数据库正常开启状态。