title: MySQL 相关知识汇总
categories: []
tags:
  - MySQL
date: '2023-04-04T13:00:07.258Z'
---
# 1. 简介

MySQL 的基本架构示意图，从中你可以清楚地看到 SQL 语句在 MySQL 的各个功能模块中的执行过程![image.png](https://blog-image.lianglianglee.com/assets/image-20221008195359-0baltm2.png)​

大体来说，MySQL 可以分为 Server 层和存储引擎层两部分。
1. Server 层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
2. 而存储引擎层负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎。

也就是说，你执行 create table 建表的时候，如果不指定引擎类型，默认使用的就是 InnoDB。不过，你也可以通过指定存储引擎的类型来选择别的引擎，比如在 create table 语句中使用 engine=memory, 来指定使用内存引擎创建表。不同存储引擎的表数据存取方式不同，支持的功能也不同。

## 1.1 引擎

* MyISAM
* InnoDB
* Memory

### 1.1.1 **MyISAM**

MyISAM 引擎是 MySQL5.5 版本（不含）之前的数据库所默认的数据表引擎。每一个采用 MyISAM 引擎的数据表在实际存储中都是由三个文件组成，分别是 frm 文件，MYD 文件和 MYI 文件，文件后缀为上述三个，文件名与数据表名相同。一个典型的 MyISAM 类型的数据表如下图（红框部分）所示：

​![image](https://blog-image.lianglianglee.com/assets/image-20221013142251-odk822t.png)​

frm 文件保存表的结构

MYD 保存表的数据

MYI 保存表的索引文件

MYD 和 MYI 与 MyISAM 引擎有很深的关联。
**特点**

1. 不支持事务。
2. 表级锁定。 即发生数据更新时，会锁定整个表，以防止其他会话对该表中数据的同时修改所导致的混乱。这样做可以使得操作简单，但是会减少并发量。
3. 读写互相堵塞。 在 MyISM 类型的表中，既不可以在向数据表中写入数据的同时另一个会话也向该表中写入数据，也不允许其他的会话读取该表中的数据。只允许多个会话同时读取该数据表中的数据。
4. 只会缓存索引，不会缓存数据。 所谓缓存，就是指数据库在访问磁盘数据时，将更多的数据读取进入内存，这样可以使得当访问这些数据时，直接从内存中读取而不是再次访问硬盘。MyISAM 可以通过 key_buffer_size 缓存索引，以减少磁盘 I/O，提升访问性能。但是 MyISAM 数据表并不会缓存数据。
5. 读取速度较快，占用资源较少。
6. 不支持外键约束。
7. 支持全文索引。

**适用场景**

1. 不需要事务支持的场景。
2. 读取操作比较多，写入和修改操作比较少的场景。
3. 数据并发量较低的场景。
4. 硬件条件比较差的场景。
5. 在配置数据库读写分离场景下，MySQL 从库可以使用 MyISAM 索引。

### 1.1.2 Innodb

略

### 1.1.3 Memory

Memory 存储引擎是 MySQL 中的一类特殊的存储引擎。其使用存储在内存中的内容来创建表，而且所有数据也放在内存中。这些特性都与 InnoDB，MyISAM 存储引擎不同。

每个基于 Memory 存储引擎的表实际对应一个磁盘文件，该文件的文件名与表名相同，类型为 frm 类型。该文件只存储表的结构，而其数据文件，都是存储在内存中的，这样有利于对数据的快速的处理，提高整个表的处理效率。

**特点**

1. 支持的数据类型有限制，比如：不支持 TEXT 和 BLOB 类型，对于字符串类型的数据，只支持固定长度的行，VARCHAR 会被自动存储为 CHAR 类型；
2. 支持的锁粒度为表级锁。所以，在访问量比较大时，表级锁会成为 MEMORY 存储引擎的瓶颈；
3. 由于数据是存放在内存中，一旦服务器出现故障，数据都会丢失；
4. 查询的时候，如果有用到临时表，而且临时表中有 BLOB，TEXT 类型的字段，那么这个临时表就会转化为 MyISAM 类型的表，性能会急剧降低；
5. 默认使用 hash 索引。
6. 如果一个内部表很大，会转化为磁盘表。

MEMORY 存储引擎拥有极高的插入，更新和查询效率

# 2. 索引

## 2.1 聚簇索引

聚簇索引是索引的叶子节点上存储的数据。一张表只能有一个。因为 Innodb 引擎必须有主键，因为数据在聚簇索引上，及时不设置主键，Innodb 也会生成一个主键【DB_ROW_ID】。Innodb 中的除主键外的索引都是非聚簇索引(二级索引)。非聚簇索引上存放的是主键的数据，从而根据主键查找实际的数据（回表）。

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008195619-a1o1jlv.png)​

## 2.2 非聚簇索引

非聚簇索引的叶子节点存放的是数据指针。一张表可以有多个。Innodb 中除了主键索引外，其他都是非聚簇索引。Myisam 不需要主键，因为引擎会单独存放数据

## 2.3 索引分类

### 2.3.1 按照存储结构分

1. BTree 索引(B-Tree，B+Tree)
2. Hash 索引
3. Full-index 索引
4. R-Tree 索引

### 2.3.2 应用层次分

1. 普通索引

一个索引只有一列，一个表可以有多个

2. 唯一索引

索引列组合必须唯一，允许有多列，但允许有 Null

3. 复合索引

索引包含多个索引列

4. 全文索引

全文索引的类型是 FullText，全文索引可以在 varchar,char,text 类型上创建，可以通过 Alter Table，或者 Create Index 命令创建。全文索引不支持中文，需要借助插件。

## 2.4 复合索引

### 2.4.1 原理

Innodb 的每个表会创建一个聚簇索引，如果有其他索引，会创建二级索引，聚簇索引以主键建立。聚簇索引和二级索引都是 B+ 树。B+ 树的特点是按照顺序将叶子节点和每页的数据相连。一般情况下使用索引查询时，先查询二级索引的 B+ 树，查到数据中的主键后，回表到聚簇索引中查询数据

### 2.4.2 使用原则

1. 最左前缀原则

在复合索引中想要命中索引，需要按照建立的顺序挨个使用，否则无法命中索引。且会一直向右匹配，直至遇到范围查询（>,<,between,like）就停止后续的匹配

2. 覆盖原则

如果索引中包含了所有需要查询的数据，则只需要在当前索引中取数据，不会从聚簇索引中再查一次。可用作极限情况下多条数据查询的性能优化。

3. 索引下推 `（Index Condition Pushdown (ICP) ）`

在不使用 ICP 的情况下，在使用非主键索引（又叫普通索引或者二级索引）进行查询时，存储引擎通过索引检索到数据，然后返回给 MySQL 服务器，服务器然后判断数据是否符合条件 。

在使用 ICP 的情况下，如果存在某些被索引的列的判断条件时，MySQL 服务器将这一部分判断条件传递给存储引擎，然后由存储引擎通过判断索引是否符合 MySQL 服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给 MySQL 服务器 。

**索引条件下推优化可以减少存储引擎查询基础表的次数，也可以减少 MySQL 服务器从存储引擎接收数据的次数。**

## 2.5 相关文档

[MySQL InnoDB 中 B+ 树索引，哈希索引，全文索引 详解](https://juejin.cn/post/6844904101839372296)

[一文了解数据库索引：哈希、B-Tree 与 LSM](https://juejin.cn/post/6844903810356215822)

[用 16 张图就给你讲明白 MySQL 为什么要用 B+ 树做索引！](https://mp.weixin.qq.com/s/muOwXKNTvPjXjrLsFRveIw)

[一文搞懂 mysql 索引底层逻辑，干货满满！ - 萨科拉 - 博客园](https://www.cnblogs.com/sakela/p/16668635.html)

# 3. 日志

## 3.1 bin log

用于记录数据库执行的写操作信息，以二进制保存到磁盘中。任何引擎都会记录 bin log。bin log 是通过追加的方式写入的，可以通过 max_bin_log_size 参数设置每个 bin log 文件大小，当达到执行大小，会生成新文件保存日志。

bin log 可以用于主从复制和数据恢复。现代的场景中，还可用于同步数据到其他辅助型数据库，甚至参与业务中。

MySQL 通过 sync_binlog 参数控制刷盘时机，取值范围是 0-N。为 0 时代表交给系统自行判断。为 1 时，代表每次 commit 都会刷盘一次。N 是每 N 次事务，才刷盘。

bin log 有三种格式。

1. STATEMENT
2. ROW
3. MIXED

STATEMENT 下，是基于 SQL 的复制，会将 SQL 记录到 Binlog 中。优点是减少了日志量，且更利于人眼查看。缺点是容易出现主从不一致。

ROW 下，是基于行的复制，不记录 SQL，只记录哪条数据被改了。优点是不会出现数据不一致问题。缺点是会产生大量日志，特别是 alter table 时。

MIXED 下，是 STATEMENT，ROW 两个模式的混合。一般使用 STATEMENT，当无法处理时用 ROW

## 3.2 redo log

redo log 包含两部分

1. 内存中的日志缓冲（redo log buffer）
2. 磁盘上的日志文件（redo log file）

MySQL 每次执行的 DML 语句会记录到 redo log buffer 中，后续某个时间点在一次性写入到 redo log file 中。这种先写日志再写磁盘的技术就是 WAL（Write-Ahead Logging）

MySQL 可以通过 innodb_flush_log_at_trx_commit 参数配置。其有 3 个值 0，1，2。

0 延迟写。事务提交后不会立即将 redo log buffer 写入到 os buffer 中，而是每秒写入一次，并调用 fsync()写入到 redo log file 中。也就是说每秒写入一次磁盘，如果系统崩溃，会丢失一秒的数据

1 每次事务提交都会将 redo log buffer 写入到 os buffer 中，并调用 fsync()刷到 redo log file 中。即使系统崩溃也不会丢失数据。但是 IO 比较差

2 每次提交只写入 os buffer，每秒调用一次 fsync()将 os buffer 中的数据写入到 redo log file 中。

## 3.3 undo log

事务的原子性是通过 undo log 实现的。比如一条 Insert 语句，对应一条的 delete 语句。每个 Update 语句，对应一条将数据还原的 Update 语句。如果发生错误，会将数据恢复到事务之前的状态。

undo log 也是 MVCC（多版本并发控制）实现的关键

## 3.4 总结

1. bin log 大小固定，可以通过 max_binlog_size 调整大小。
2. redo log 是 Innodb 引擎层的实现，并不是所有引擎都有的
3. bin log 是 server 层的实现。所以引擎都有 bin log 日志
4. redo log 采用循环写的模式，当写到结尾，会再次从头开始写
5. bin log 用追加的方式实现，当达到指定大小会生成新文件。
6. redo log 用于崩溃后的恢复，bin log 用于主从复制和数据恢复。
7. bin log 数据归档日志。
8. 事务原子性是通过 undo log 实现的，且是引擎层的实现

## 3.5 相关文档

[深入解析 MySQL binlog](https://cloud.tencent.com/developer/article/1032755)

[MySQL · 引擎特性 · InnoDB redo log 漫游](http://mysql.taobao.org/monthly/2015/05/01/)

[MySQL · 引擎特性 · InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)

# 4. SQL 优化

## 4.1 各种简单手段

1. 若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了
2. 变长字段存储空间小，可以节省存储空间
3. 当索引列大量重复数据时，可以把索引删除掉
4. 尽量避免在 where 子句中使用!=或 <> 操作符
5. 尽量避免在 where 子句中使用 or 来连接条件
6. 任何查询也不要出现 select *
7. 避免在 where 子句中对字段进行 null 值判断
8. 对作为查询条件和 order by 的字段建立索引
9. 避免建立过多的索引，多使用组合索引

## 4.1 相关文档

[【得物技术】MySQL 深分页优化](https://juejin.cn/post/6985478936683610149)

[京东云开发者-【案例回顾】春节一次较波折的 MySQL 调优](https://my.oschina.net/u/4090830/blog/5571935)

[记录一次数据库 CPU 被打满的排查过程 - 京东云开发者 - 博客园](https://www.cnblogs.com/Jcloud/p/16642188.html)

[一次较波折的 MySQL 调优 - 京东云开发者 - 博客园](https://www.cnblogs.com/Jcloud/p/16646031.html)

# 5. 隔离级别

|隔离级别|脏读|不可重复读|幻读|概念|
| ---------------| ----| ----------| ----| -----------------------------------------------------------------------------------------------------------------------------------|
|READ UNCOMMITED|√|√|√|事务能够看到其他事务没有提交的修改，当另一个事务又回滚了修改后的情况，又被称为脏读 dirty read|
|READ COMMITTED|×|√|√|事务能够看到其他事务提交后的修改，这时会出现一个事务内两次读取数据可能因为其他事务提交的修改导致不一致的情况，称为不可重复读|
|REPEATABLE READ|×|×|√|事务在两次读取时读取到的数据的状态是一致的|
|SERIALIZABLE|×|×|×|可重复读中可能出现第二次读读到第一次没有读到的数据，也就是被其他事务插入的数据，这种情况称为幻读 phantom read, 该级别中不能出现幻读|

# 6. 多版本并行控制

## 6.1 MVCC

**DB_TRX_ID：**

事务 id，6byte，记录最新一次更新的事务的 ID

每行数据是有多个版本的，每次事务更新数据时都会生成一个新的数据版本，并且把 transaction id 赋值给这个数据行的 DB_TRX_ID

**DB_ROLL_PTR：**

回滚指针，7byte，指向当前记录的 `ROLLBACK SEGMENT` 的 undo log 记录，通过这个指针获得之前版本的数据。该行记录上所有旧版本在 `undo log` 中都通过链表的形式组织。

---

读不加锁，读写不冲突。在读多写少的 OLTP 应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能，这也是为什么现阶段，几乎所有的 RDBMS，都支持了 MVCC。

于 delete 操作，innodb 是通过先将要删除的那一行标记为删除，而不是马上清除这一行，因为 Innodb 实现了 MVCC，这些 undo 段用来实现 MVCC 多版本机制。锁不阻塞读，读也不阻塞写，这样大大提高了并发性。那么在一致性读的时候，怎么才能找到和事务开始的那个版本呢？

**主键索引，每个行都有一个事务 ID 和一个 undo ID，这个 undo ID 指向了这行的先前版本的位置。**[DB_TRX_ID,DB_ROLL_PTR]

InnoDB 的行记录格式中有 6 字节事务 ID 的和 7 字节的回滚指针，通过为每一行记录添加这两个额外的隐藏值来实现 MVCC，这两个值一个记录这行数据何时被创建，另外一个记录这行数据何时过期（或者被删除）。但是 InnoDB 并不存储这些事件发生时的实际时间，相反它只存储这些事件发生时的系统版本号。这是一个随着事务的创建而不断增长的数字。每个事务在事务开始时会记录它自己的系统版本号。每个查询必须去检查每行数据的版本号与事务的版本号是否相同。让我们来看看当隔离级别是 REPEATABLE READ 时这种策略是如何应用到特定的操作的。

**SELECT：**

当隔离级别是 REPEATABLE READ 时 `select` 操作，InnoDB 必须每行数据来保证它符合两个条件：

1、InnoDB 必须找到一个行的版本，它至少要和事务的版本一样老(也即它的版本号不大于事务的版本号)。这保证了不管是事务开始之前，或者事务创建时，或者修改了这行数据的时候，这行数据是存在的。

2、这行数据的删除版本必须是未定义的或者比事务版本要大。这可以保证在事务开始之前这行数据没有被删除。

符合这两个条件的行可能会被当作查询结果而返回。

**INSERT：**

InnoDB 为这个新行记录当前的系统版本号。

**DELETE：**

InnoDB 将当前的系统版本号设置为这一行的删除 ID。

**UPDATE：**

InnoDB 会写一个这行数据的新拷贝，这个拷贝的版本为当前的系统版本号。它同时也会将这个版本号写到旧行的删除版本里。

---

这种额外的记录所带来的结果就是对于大多数查询来说根本就不需要获得一个锁。他们只是简单地以最快的速度来读取数据，确保只选择符合条件的行。这个方案的缺点在于存储引擎必须为每一行存储更多的数据，<br> 做更多的检查工作，处理更多的善后操作。

MVCC 只工作在 REPEATABLE READ 和 READ COMMITED 隔离级别下。READ UNCOMMITED 不是 MVCC 兼容的，因为查询不能找到适合他们事务版本的行版本；它们每次都只能读到最新的版本。<br>SERIABLABLE 也不与 MVCC 兼容，因为读操作会锁定他们返回的每一行数据。

## 6.2 ReadView

InnoDB 支持 MVCC 多版本，其中 RC（Read Committed）和 RR（Repeatable Read）隔离级别是利用 consistent read view（一致读视图）方式支持的。 所谓 consistent read view 就是在某一时刻给事务系统 trx_sys 打 snapshot（快照），把当时 trx_sys 状态（包括活跃读写事务数组）记下来，之后的所有读操作根据其事务 ID（即 trx_id）与 snapshot 中的 trx_sys 的状态作比较，以此判断 read view 对于事务的可见性。

‍

```shell
private:
  trx_id_t m_low_limit_id;      /* 大于等于这个 ID 的事务均不可见 */

  trx_id_t m_up_limit_id;       /* 小于这个 ID 的事务均可见 */

  trx_id_t m_creator_trx_id;    /* 创建该 Read View 的事务ID */

  trx_id_t m_low_limit_no;      /* 事务 Number, 小于该 Number 的 Undo Logs 均可以被 Purge */

  ids_t m_ids;                  /* 创建 Read View 时的活跃事务列表 */

  m_closed;                     /* 标记 Read View 是否 close */
}
```

Read view 中保存的 trx_sys 状态主要包括

* low_limit_id：high water mark，大于等于 view->low_limit_id 的事务对于 view 都是不可见的
* up_limit_id：low water mark，小于 view->up_limit_id 的事务对于 view 一定是可见的
* low_limit_no：trx_no 小于 view->low_limit_no 的 undo log 对于 view 是可以 purge 的
* rw_trx_ids：读写事务数组

RR 隔离级别（除了 Gap 锁之外）和 RC 隔离级别的差别是创建 snapshot 时机不同。 RR 隔离级别是在第一个读操作创建 read view 的；RC 隔离级别是在事务开始的时刻创建 read view 的。

下次创建 read view，判断如果是只读事务并且系统的读写事务状态没有发生变化，即 trx_sys 的 max_trx_id 没有向前推进，而且没有新的读写事务产生，就可以重用上次的 read view。

除此之外可以通过 high/low water mark 快速判断：

* trx_id < view->up_limit_id 的记录对于当前 read view 是一定可见的；
* trx_id >= view->low_limit_id 的记录对于当前 read view 是一定不可见的；

如果 trx_id 落在[up_limit_id, low_limit_id)，需要在活跃读写事务数组查找 trx_id 是否存在，如果存在，记录对于当前 read view 是不可见的。

## 6.3 **补充**

**LBCC**：Lock-Based Concurrency Control，基于锁的并发控制。

**MVCC**：Multi-Version Concurrency Control，基于多版本的并发控制协议。纯粹基于锁的并发机制并发量低，MVCC 是在基于锁的并发控制上的改进，主要是在读操作上提高了并发量。

在 MVCC 并发控制中，读操作可以分成两类：

1）快照读 (snapshot read)：读取的是记录的可见版本 (有可能是历史版本)，不用加锁（共享读锁 s 锁也不加，所以不会阻塞其他事务的写）。

2）当前读 (current read)：读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

## 6.4 相关文档

[MySQL · 源码分析 · InnoDB 的 read view，回滚段和 purge 过程简介](http://mysql.taobao.org/monthly/2018/03/01/ "MySQL · 源码分析 · InnoDB的read view，回滚段和purge过程简介")

[MySQL 到底有没有解决幻读问题？这篇文章彻底给你解答](https://www.cnblogs.com/yidengjiagou/p/16688967.html)

[详解 MySQL 隔离级别 ](https://www.cnblogs.com/jeremylai7/p/16623780.html)

# 7. 主从复制

## 7.1 过程

1. 从节点 在从节点上执行 start slave 命令开启主从复制开关，开始进行主从复制。从节点上的 I/O 进程连接主节点，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容
2. 主节点接收到来自从节点的 I/O 请求后，通过负责复制的 I/O 进程（binlog dump 线程）根据请求信息读取指定日志指定位置之后的日志信息，返回给从节点。返回信息中除了日志所包含的信息之外，还包括本次返回的信息的 binlog file 的以及 binlog position（binlog 中的下一个指定更新位置）
3. 从节点的 I/O 进程接收到主节点发送过来的日志内容、日志文件及位置点后，将接收到的日志内容更新到本机的 relay-log（中继日志）的文件（Mysql-relay-bin.xxx）的最末端，并将读取到的 binary log（bin-log）文件名和位置保存到 master-info 文件中，以便在下一次读取的时候能够清楚的告诉 Master"我需要从某个 binlog 的哪个位置开始往后的日志内容，请发给我"
4. Slave 的 SQL 线程检测到 relay-log 中新增加了内容后，会将 relay-log 的内容解析成在主节点上实际执行过 SQL 语句，然后在本数据库中按照解析出来的顺序执行，并在 relay-log.info 中记录当前应用中继日志的文件名和位置点

## 7.2 主从复制模型

1. 异步复制（默认）
2. 半同步复制
3. 组复制
4. 全同步复制
5. GTID 复制
6. 多线程复制

### 7.2.1 异步复制

主节点不会主动推送 bin-log 到从节点，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主节点如果崩溃掉了，此时主节点上已经提交的事务可能并没有传到从节点上

### 7.2.2 半同步复制

介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到 relay-log 中才返回成功信息给客户端（只能保证主库的 bin-log 至少传输到了一个从节点上，但并不能保证从节点将此事务执行更新到 db 中），否则需要等待直到超时时间然后切换成异步模式再提交。相对于异步复制，半同步复制提高了数据的安全性，一定程度的保证了数据能成功备份到从库，同时它也造成了一定程度的延迟，但是比全同步模式延迟要低，这个延迟最少是一个 TCP/IP 往返的时间。所以，半同步复制最好在低延时的网络中使用

### 7.2.3 组复制

基于 Paxos 协议实现的组复制，保证数据一致性

### 7.2.4 全同步复制

指当主库执行完一个事务，然后所有的从库都复制了该事务并成功执行完才返回成功信息给客户端。因为需要等待所有从库执行完该事务才能返回成功信息，所以全同步复制的性能必然会收到严重的影响

### 7.2.5 GTID 复制

基于 GTID 的复制是 MySQL 5.6 后新增的复制方式。GTID (global transaction identifier) 即全局事务 ID, 保证了在每个在主库上提交的事务在集群中有一个唯一的 ID

在传统的复制里面，当发生故障，需要主从切换，需要找到 binlog 和 pos 点（指从库更新到了主库 binlog 的哪个位置，这个位置之前都已经更显完毕，这个位置之后未更新），然后将主节点指向新的主节点，相对来说比较麻烦，也容易出错。在 MySQL 5.6 里面，不用再找 binlog 和 pos 点，我们只需要知道主节点的 ip，端口，以及账号密码就行，因为复制是自动的，MySQL 会通过内部机制 GTID 自动找点同步

**原理**

在原来基于日志的复制中, 从库需要告知主库要从哪个偏移量进行增量同步, 如果指定错误会造成数据的遗漏, 从而造成数据的不一致。而基于 GTID 的复制中, 从库会告知主库已经执行的事务的 GTID 的值, 然后主库会将所有未执行的事务的 GTID 的列表返回给从库. 并且可以保证同一个事务只在指定的从库执行一次.通过全局的事务 ID 确定从库要执行的事务的方式代替了以前需要用 binlog 和 pos 点确定从库要执行的事务的方式

GTID 是由 server_uuid 和事务 id 组成，格式为：GTID=server_uuid:transaction_id。server_uuid 是在数据库启动过程中自动生成，每台机器的 server-uuid 不一样。uuid 存放在数据目录的 auto.conf 文件中，而 transaction_id 就是事务提交时系统顺序分配的一个不会重复的序列号

主节点更新数据时，会在事务前产生 GTID，一起记录到 binlog 日志中。从节点的 I/O 线程将变更的 binlog，写入到本地的 relay-log 中。SQL 线程从 relay-log 中获取 GTID，然后对比本地 binlog 是否有记录（所以 MySQL 从节点必须要开启 binary log）。如果有记录，说明该 GTID 的事务已经执行，从节点会忽略。如果没有记录，从节点就会从 relay-log 中执行该 GTID 的事务，并记录到 binlog。在解析过程中会判断是否有主键，如果没有就用二级索引，如果有就用全部扫描

**好处**

GTID 使用 master_auto_position=1 代替了 binlog 和 position 号的主从复制搭建方式，相比 binlog 和 position 方式更容易搭建主从复制

GTID 方便实现主从之间的 failover（主从切换），不用一步一步的去查找 position 和 binlog 文件

**局限**

不能使用 create table table_name select * from table_name 模式的语句

在一个事务中既包含事务表的操作又包含非事务表

不支持 CREATE TEMPORARY TABLE or DROP TEMPORARY TABLE 语句操作

使用 GTID 复制从库跳过错误时，不支持 sql_slave_skip_counter 参数的语法

### 7.2.6 多线程复制

多线程复制（基于库），在 MySQL 5.6 以前的版本，slave 的复制是单线程的，而 master 是并发写入的，所以延时是避免不了的。唯一有效的方法是把多个库放在多台 slave，这样又有点浪费服务器。在 MySQL 5.6 里面，我们可以把多个表放在多个库，这样就可以使用多线程复制。但 5.6 中的每个线程只能处理一个数据库，所以如果只有一个数据库，或者绝大多数写操作都是集中在某一个数据库的，那么这个“多线程复制”就不能充分发挥作用了

### 7.2.7 相关文档

[深入了解 MySQL 主从复制的原理](https://segmentfault.com/a/1190000038967218)

[深度探索 MySQL 主从复制原理](https://zhuanlan.zhihu.com/p/50597960)

# 8. 其他

## 8.1 锁

|锁类型|锁级别|提供方|语法|备注|
| ------| ------| ------| -------------------------------------------------------| --------------------------------------------------------------|
|全局锁|实例级|Server|flush tables with read lock||
|表锁|表级|Server|lock tables t1 read;<br />lock tables t2 write;<br />unlock tables;<br />|需要主动释放锁|
|MDL|表级|Server|无|select 会自动加上 MDL 读锁。当有 Alter 语句是，升级为 MDL 写锁|
|行锁|行级|引擎|||

### 8.1.1 全局锁（FTWRL）

顾名思义，全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是 Flush tables withread lock (FTWRL)。当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。全局锁的典型使用场景是，做全库逻辑备份。也就是把整库每个表都 select 出来存成文本。

### 8.1.2 表级锁

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

**表锁的语法是 lock tables … read/write**。与 FTWRL 类似，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

在还没有出现更细粒度的锁的时候，表锁是最常用的处理并发的方式。而对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。

**另一类表级的锁是 MDL（metadata lock)。**MDL 不需要显式使用，在访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，在 MySQL 5.5 版本中引入了 MDL，当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。

* 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
* 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

虽然 MDL 锁是系统默认会加的，但却是你不能忽略的一个机制。经常看到有人掉到这个坑里：给一个小表加个字段，导致整个库挂了。

**MDL 锁没有超时限制，只要事务没提交就会一直锁。**

### 8.1.3 行级锁

MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM 引擎就不支持行锁。不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB 是支持行锁的，这也是 MyISAM 被 InnoDB 替代的重要原因之一。

顾名思义，行锁就是针对数据表中行记录的锁。这很好理解，比如事务 A 更新了一行，而这时候事务 B 也要更新同一行，则必须等事务 A 的操作完成后才能进行更新。

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008194341-ol5iakt.png)​

**在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。**

### 8.1.4 死锁和死锁检测

**核心参数**

innodb_deadlock_detect：死锁检测参数，默认为 on，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行

innodb_lock_wait_timeout：死锁等待的超时时间，默认为 50s，意味着如果不开启死锁检测，则在发生死锁之后，会等待 50s，直到超时。

---

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。这里我用数据库中的行锁举个例子

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008194533-hogldh7.png)​

这时候，事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。当出现死锁以后，有两种策略：

* 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。
* 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

在 InnoDB 中，innodb_lock_wait_timeout 的默认值是 50s，意味着如果采用第一个策略，当出现死锁以后，第一个被锁住的线程要过 50s 才会超时退出，然后其他线程才有可能继续执行。对于在线服务来说，这个等待时间往往是无法接受的。

但是，我们又不可能直接把这个时间设置成一个很小的值，比如 1s。这样当出现死锁的时候，确实很快就可以解开，但如果不是死锁，而是简单的锁等待呢？所以，超时时间设置太短的话，会出现很多误伤。所以，正常情况下我们还是要采用第二种策略，即：主动死锁检测，而且 innodb_deadlock_detect 的默认值本身就是 on。主动死锁检测在发生死锁的时候，是能够快速发现并进行处理的，但是它也是有额外负担的。

### 8.1.5 其他

* **意向排他锁（IX 锁）：** 事务计划给记录加行排他锁，事务在给一行记录加排他锁前，必须先取得该表的 IX 锁
* **意向共享锁（IS 锁）：** 事务计划给记录加行共享锁，事务在给一行记录加共享锁前，必须先取得该表的 IS 锁
* **共享锁（S 锁）：**共享锁允许一个事务读数据，不允许修改数据，如果其他事务要再对该行加锁，只能加共享锁；
* **排它锁（X 锁）：**排他锁是修改数据时加的锁，可以读取和修改数据，一旦一个事务对该行数据加锁，其他事务将不能再对该数据加任务锁。

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008201113-yjyu78x.png)​

### 8.1.6 相关文档

[十分钟了解一下 MySql 锁机制](https://juejin.cn/post/7130423416607375373)

[MySQL 进阶系列：不同隔离级别下加锁情况](https://juejin.cn/post/7129344329830629389)

## 8.2 Online DDL

MySQL Online DDL 这个新特性是在 **MySQL5.6.7** 开始支持的，更早期版本的 MySQL 进行 DDL 对于 DBA 来说是非常痛苦的。现在主流版本都集中在 5.6 与 5.7。

```sql
alter table sbtest1 add column h int(11) default null,algorithm=inplace;
```

目前 MySQL 自带的策略

* copy：是指 DDL 时，会生成（临时）新表，将原表数据逐行拷贝到新表中，在此期间会阻塞 DML
* inplace：无需拷贝全表数据到新表，但可能还是需要 IN-PLACE 方式（原地，无需生成新的临时表）重建整表。这种情况下，在 DDL 的初始准备和最后结束两个阶段时通常需要加排他 MDL 锁（metadata lock，元数据锁），除此外，DDL 期间不会阻塞 DML
* instant：只需修改数据字典中的元数据，无需拷贝数据也无需重建整表，同样，也无需加排他 MDL 锁，原表数据也不受影响。整个 DDL 过程几乎是瞬间完成的，也不会阻塞 DML。这个新特性是 8.0.12 引入的(腾讯提交的补丁)。

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008192941-c13f3bz.png)​

​![image.png](https://blog-image.lianglianglee.com/assets/image-20221008192924-dkgljj2.png)​

> MySQL 会自己决策使用哪种方式，一般不需要指定

[https://www.cnblogs.com/dbabd/p/10381942.html](https://www.cnblogs.com/dbabd/p/10381942.html)

[https://juejin.cn/post/6854573213167386637](https://juejin.cn/post/6854573213167386637)

[https://blog.csdn.net/shenchaohao12321/article/details/120564592](https://blog.csdn.net/shenchaohao12321/article/details/120564592)

[https://mydbops.wordpress.com/2020/03/04/an-overview-of-ddl-algorithms-in-mysql-covers-mysql-8/](https://mydbops.wordpress.com/2020/03/04/an-overview-of-ddl-algorithms-in-mysql-covers-mysql-8/)

[https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html)

## 8.3 change buffer

当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

虽然名字叫作 change buffer，实际上它是可以持久化的数据。也就是说，change buffer 在内存中有拷贝，也会被写入到磁盘上。

将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge。除了访问这个数据页会触发 merge 外，系统有后台线程会定期 merge。在数据库正常关闭（shutdown）的过程中，也会执行 merge 操作。

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入 (4,400) 这个记录，就要先判断现在表中是否已经存在 k=4 的记录，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。
因此，**唯一索引的更新就不能使用 change buffer**，**实际上也只有普通索引可以使用**。

change buffer 用的是 buffer pool 里的内存，因此不能无限增大。change buffer 的大小，可以通过参数 **innodb_change_buffer_max_size** 来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。反过来，假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在changebuffer，但之后由于马上要访问这个数据页，会立即触发 merge 过程。这样随机访问 IO 的次数不会减少，反而增加了 change buffer 的维护代价。所以，对于这种业务模式来说，change buffer 反而起到了副作用。

## 8.4 索引选择异常的处理

* 采用 force index 强行选择一个索引。
* 可以考虑修改语句，引导 MySQL 使用我们期望的索引。

## 8.5 索引优化的奇淫技巧

* 使用倒序存储。如果你存储身份证号的时候把它倒过来存。
* 使用 Hash 索引。如果只有精确查询，使用 Hash 索引可提高性能。
* 覆盖索引。如果需要做大量的 In 查询，且查询出的 2-3 个字段，则可以将查询的字段都加入索引，避免回表。
* 前缀索引。如果前缀索引区分度足够，则创建前缀索引可有效降低索引数据大小。但不能用覆盖索引了。

# 9. 文档

[参数调优建议](https://help.aliyun.com/document_detail/63255.htm?spm=a2c4g.11186623.0.0.67e1799cm9sXx1#concept-8056 "参数调优建议")

[调整实例 Buffer Pool 大小](https://help.aliyun.com/document_detail/162326.html "调整实例Buffer Pool大小")

[慢 SQL 治理分享](https://zhuanlan.zhihu.com/p/395493156 "慢SQL治理分享")

[RDS MySQL 慢 SQL 问题](https://help.aliyun.com/document_detail/202152.html)

[RDS MySQL 内存使用问题](https://help.aliyun.com/document_detail/202149.html)

[MySQL 索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)

[MySQL 常见的七种锁详细介绍](https://blog.csdn.net/Saintyyu/article/details/91269087)

‍

‍
