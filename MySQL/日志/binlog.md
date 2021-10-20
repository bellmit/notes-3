- [一、与redo log的有什么区别](#一与redo-log的有什么区别)
- [二、如何开启binlog](#二如何开启binlog)
- [三、binlog的写入逻辑](#三binlog的写入逻辑)
- [四、使用场景](#四使用场景)
- [五、配置参数](#五配置参数)
    - [1、sync_binlog](#1sync_binlog)
    - [2、max_binlog_size](#2max_binlog_size)
- [六、binlog日志格式](#六binlog日志格式)

# 一、与redo log的有什么区别

1. `redo log` 是 InnoDB 引擎特有的；`binlog` 是 Server 层实现的，所有引擎都可以使用。
2. `redo log` 是物理日志，记录的是“在某个数据页做了什么修改”；`binlog` 是逻辑日志，记录的是这个语句的原始逻辑，比如`update t set a=a+1 where id=1;`。
3. `redo log` 是循环写的，空间固定；binlog 是追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

# 二、如何开启binlog

要开启binlog，必须配置一个`server-id`，然后配置binlog的文件名，如：`log-bin=binlogfile`。

然后配置`my.cnf`文件

```conf
[mysqld]
server-id=1
log-bin=binlogfile
```

# 三、binlog的写入逻辑

分析语句`update t set a=a+1 where id=1;`的执行过程：

1. 执行器先找引擎获取id=1这条记录，id是主键，引擎直接用树搜索找到这一行，如果id=1这一行所在的数据页在内存中，就直接返回给执行器，否则先从磁盘读入内存，再返回给执行器。
2. 执行器拿到数据，把 a 的值加上1，然后调用引擎的接口写入这行数据。
3. 引擎将这行数据更新到内存中，同时将这个操作写入`redo log`，此时`redo log`处于prepare状态，然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的binlog，把binlog写入磁盘。
5. 执行器调用引擎的提交事物接口，引擎把刚刚写入的`redo log`改成commit状态，更新完成。

这里将 redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。

**问题：为什么要进行两阶段提交呢？**

如果不采用两阶段提交会怎么样？

1. 先写`redo log`后写`binlog`：假设`redo log`写完，`binlog`还没写完的时候，MySQL进程异常重启，因为`redo log`写完了，所以仍然能够把数据恢复回来，但是！`binlog`没写完就crash了，此时`binlog`中就没有这个更新语句了，后面备份日志的时候，就导致数据不正确，也会导致主从同步错误。
2. 先写`binlog`后写`redo log`：如果在`binlog`写完之后crash了，由于`redo log`还没写完，崩溃恢复后，这个更新语句就等于没执行过，值还是原来的值，但是！`binlog`中是有记录的，在本分日志的时候，就导致了数据不正确，也会导致主从同步错误。


# 四、使用场景

1. 主从复制
2. 数据恢复

# 五、配置参数

## 1、sync_binlog

控制 MySQL 将二进制同步到磁盘的频率。

- sync_binlog=0：刷盘的频率依赖于操作系统，MySQL不进行控制。
- sync_binlog=1：事务提交后会立即进行刷盘，默认值。
- sync_binlog=N：设置为非0和1时，意思是在N次事务提交时进行刷盘。

## 2、max_binlog_size

配置单个 big_log 文件的最大大小，默认是 1G，最小是 4096B，最大是 1G。

# 六、binlog日志格式

有三种日志记录格式：

1. STATEMENT：基于语句的日志记录。
2. ROW：基于行的日志记录。
3. MIXED：混合日志记录。

设置方法：

1、启动时加入参数：`--binlog-format=STATEMENT|ROW|MIXED`。

2、运行时设置：`set global|session  binlog_format=STATEMENT|ROW|MIXED`。

3、在 my.cnf 中配置
```conf
[mysqld]
binlog-format=STATEMENT
```

查看binlog文件：`mysqlbinlog -v /usr/local/var/mysql/bin_log.000018`
