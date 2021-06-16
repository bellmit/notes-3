# 一、概念

Multi-Version Concurrency Control，即多版本并发控制，它可以保存一行记录的多个历史版本，这些历史版本信息保存在 system tablespaces 或 undo tablespaces 中，统一叫做 rollback segment。用这些信息来支持事物的回滚操作和一致性读（可重复读）。

引入 MVCC 之后新增了两个读的概念：**快照读**和**当前读**。

- 快照读：快照读的实现是基于MVCC，它读取的数据可能是历史数据。
- 当前读：当前读即读取的是最新的数据，会对读取的记录进行加排它锁，保证读取时其它事务不能修改当前记录。

MVCC 就是为了实现读-写冲突不加锁，这个读就是快照读，非当前读。主要解决了不可重复读的问题，也解决了幻读的问题。

MVCC 的核心主要是：

1. 每行数据会有三个隐藏字段：DB_TRX_ID、DB_ROLL_PTR 和 DB_ROW_ID
2. Undo log
3. 视图（read view）

通过这三个功能来实现一致性读（可重复读）。

# 二、隐藏字段

实际上，InnoDB 会给每一行记录增加三个隐藏字段：

1. 6个字节的 **DB_TRX_ID**：存储修改（插入、更新和删除）这行数据的最后一个事务的id。此外删除操作在内部视为更新操作，将该行中的特殊bit位标记为已删除。
2. 7个字节的 **DB_ROLL_PTR**：存储着回滚指针。指向 rollback segment 中的一个回滚日志记录，如果一行被更新了，则回滚日志中记录了如何还原的信息。
3. 6个字节的 **DB_ROW_ID**：存储着自增的 row ID，如果使用了聚簇索引，则里面存储着 row ID 的值，否则该字段不会出现在任何索引中。

我理解的行记录如下图：

![1A970F0E-64D8-4154-8AB2-279AE1FDC6B4](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623843014.3690763uOw72aAVb.png)

**注意**：不能给一个表添加字段名为 DB_TRX_ID、DB_ROLL_PTR 及 DB_ROW_ID 的字段（MySQL不区分大小写）。

# 三、undo log

## 1、概念

undo log 有两个作用：回滚事务和支持MVCC。

Undo log 中记录的是逻辑日志，比如执行一条 delete 语句时，就会在 undo log 中记录一条 insert 语句，反之亦然。

undo log 在 rollback segment 中，分为了插入日志和更新日志：

1. 插入日志：只有在回滚时需要，事务提交后，该日志很快就被删除。
2. 更新日志：用来进行一致性读，当没有事务需要时会被删除。

## 2、存储位置

Undo log 存在于 undo log segment 中，

undo log segment 存在于 rollback segment，

rollback segment 存在于 system tablespace、undo tablespaces 和 temporary tablespaces。

![48E93A09-BF71-452A-97C8-45DC537E41E9](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623843045.575447wS7b81ifPy.png)

## 3、配置

### a、**innodb_undo_directory**

配置 undo log 日志文件的存放目录。默认为 . 即在 datadir 中。

### b、**innodb_undo_logs**

定义 rollback segments 的数量，每个回滚段中有 1024 个 undo log segment。

默认值：128

最小值：1

最大值：128

### c、**innodb_max_undo_log_size**

定义 undo tablespaces 的最大大小。

默认值：1024MB

最小值：10MB

最大值：2^64 - 1

# 4、解释

Undo log 可以理解为一个链表，由 DB_ROLL_PTR 连接起来，每执行一次插入或更新操作，就会在链表中增加一个节点，并且 purge 线程在分析到某个历史节点不再被使用时就会清除该节点（删除日志）。

注意：不能使用长事务，因为长事务会一直保留 undo log，这样日志文件会越来越大。

![E4972D34-200D-46F2-A332-0705B922EBAD](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623843085.233679XmOR9Ct2Bk.png)

# 四、视图（read view）

也就是某个时间点的数据库快照，它的作用是定义事务执行期间”我能看见什么数据“。

在 MySQL 里，有两个视图的概念：

- 一个是虚拟表，使用 create view 创建的。
- 一个是 InnoDB 在实现 MVCC 时用到的一致性视图，用于支持 RC 和 RR 隔离级别的实现。

视图的创建时间：

1. 使用 BEGIN 或 START TRANSACTION 开启事务，然后会在第一个 SELECT 语句中进行创建。
2. 使用 START TRANSACTION WITH CONSISTENT SNAPSHOT 直接开启事物并创建快照。

InnoDB 里面每个事务有一个唯一的事务ID，叫做 trx_id，它是在事务开始的时候向 InnoDB 的事务系统申请的，是按照申请顺序严格递增的。

每行数据都是有多个版本的，每次事务更新的时候，都会生成一个新的版本，并且把当前事务的 trx id 赋值给这行数据的 DB_TRX_ID 字段，同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息可以直接拿到它（即使用 DB_ROLL_PTR 指针进行指向）。

按照可重复读的定义，一个事务启动时，能够看到所有已经提交的事务的修改，但是之后，这个事务执行期间，其他事务的更新对它不可见。

在实现上，InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间当前正在“活跃”的所有事务 ID。“活跃”指的就是，启动了但还没提交。数组里面事务 ID 的最小值记为低水位（说明比这个 ID 还小的事务都已经提交了），当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位（说明比这个 ID 还大的事务是未来的事务）。

这个视图数组和高低水位，就组成了当前事务的一致性视图（read-view）。而数据版本的可见性规则，就是基于数据的 DB_TRX_ID 和这个一致性视图的对比结果得到的。

![AF50E145-1DA4-4B5A-80F6-77B9A5315B94](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623843103.150505y92kQGJR1e.png)

这样，对于当前事务的启动瞬间来说，一个数据版本的 DB_TRX_ID，有以下几种可能：

1. 如果落在绿色部分，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的；
2. 如果落在红色部分，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
3. 如果落在黄色部分，那就包括两种情况
4. 1. 若 DB_TRX_ID 在数组中，表示这个版本是由还没提交的事务生成的，不可见；
    2. 若 DB_TRX_ID 不在数组中，表示这个版本是已经提交了的事务生成的，可见。（重点理解）

一个数据版本，对于一个事务视图来说，除了自己的更新总是可见以外，有三种情况：

1. 版本未提交，不可见；
2. 版本已提交，但是是在视图创建后提交的，不可见；
3. 版本已提交，而且是在视图创建前提交的，可见。



**Q1**、为什么事务A中看不到事务B中插入的新记录？

答：因为新插入的记录的 DB_TRX_ID 比事务A的 TRX_ID 大，属于不可见范围。

**Q2**、为什么事务A中依旧能看到事务B中删除的记录？

答：删除后只是把记录的删除标记更新为 true 了，并不是真正的物理删除，MySQL会启动一个线程：purge线程对删除标记为 true 的记录进行删除。标记为删除后，该行的 DB_TRX_ID 也被更新了，也属于不可见范围。



