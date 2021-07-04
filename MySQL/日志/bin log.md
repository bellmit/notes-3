# 概述

它的主要使用场景：1、**主从复制**，2、**数据恢复**。

# 一、与 redo log 的差别是什么？

1. redo log 是 InnoDB 引擎特有的；binlog 是 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是 “在某个数据页做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如 “给 id=2 这一行的 c 字段加 1”。
3. redo log 是循环写的，空间固定；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

# 二、为什么需要两个日志（redo log 和 binlog）

因为最开始 MySQL 里并没有 InnoDB 引擎。MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。

# 三、配置参数

## 1、开启big_log

要开启 bin_log，必须配置一个 server-id，然后配置 big_log 的文件名，如：log-bin=mysql-bin。

## 2、sync_binlog

控制 MySQL 将二进制同步到磁·盘的频率。

- sync_binlog=0：刷盘的频率依赖于操作系统，MySQL不进行控制。
- sync_binlog=1：事务提交后会立即进行刷盘，默认值。
- sync_binlog=N：设置为非0和1时，意思是在N次事务提交时进行刷盘。

配置汇总：

```sql
[mysqld]
server-id=1
log-bin=mysql-bin
sync_binlog=1
```

配置单个 big_log 文件的最大大小，默认是 1G，最小是 4096B，最大是 1G：`max_binlog_size=1073741824`

# 四、binlog日志格式

有三种日志记录格式：

1. STATEMENT：基于语句的日志记录。
2. ROW：基于行的日志记录。
3. MIXED：混合日志记录。

**设置命令**：1、启动时加入参数 `--binlog-format=STATEMENT|ROW|MIXED`。2、运行时设置 `set global|session  binlog_format=STATEMENT|ROW|MIXED`。

在 my.cnf 中配置：

```sql
[mysqld]
binlog-format=STATEMENT
```

如何查看呢？使用如下命令进行查看：

`mysqlbinlog -v /usr/local/var/mysql/bin_log.000018`

# 五、binlog 和 redo log 的两段提交

因为 binlog 是 Server 层的，redo log 是 InnoDB 层的，事务提交后写入数据时，先写 redo log，然后写 binlog，最后执行 redo log 的 commit 操作，要保证 redo log 和 binlog 的数据一致性。

- 先写 redo log 后写 binlog，如果写完 redo log 后MySQL挂掉了，binlog 中就没有这条数据，在备份的时候就会丢失。

- 先写 binlog 后写 redo log，如果写完 binlog 后MySQL挂掉了，重启后，由于 redo log 中没有数据，所以本机中就丢了数据。