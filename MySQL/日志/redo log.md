- [小酒馆的故事](#小酒馆的故事)
- [一、介绍](#一介绍)
- [二、LSN](#二lsn)
- [三、checkpoint](#三checkpoint)
- [四、配置参数](#四配置参数)
    - [1、**<u>innodb_flush_log_at_trx_commit</u>**](#1uinnodb_flush_log_at_trx_commitu)
    - [2、**<u>innodb_log_buffer_size</u>**](#2uinnodb_log_buffer_sizeu)
    - [3、**<u>innodb_log_file_size</u>**](#3uinnodb_log_file_sizeu)
    - [4、**<u>innodb_log_files_in_group</u>**](#4uinnodb_log_files_in_groupu)
    - [5、**<u>innodb_log_group_home_dir</u>**](#5uinnodb_log_group_home_diru)

# 小酒馆的故事

酒馆的老板有一个赊账的账本，用来记录酒馆所有的赊账记录，在生意红火时，老板每次记录赊账数据都去查询这个大账本的话，效率会很低，怎么办呢？老板还有一个小黑板，用来临时的记录最新的赊账数据，等到不忙的时候，再把小黑板上的数据合并到大账本中，这样效率就会提升很多，那小黑板记满了呢？有两种办法，1、先把小黑板一部分数据合并到大账本中，2、多准备一两个小黑板。

MySQL中的`redo log`就是采用这种方法提高了数据修改效率以及保证了数据修改的持久性和正确性。

# 一、介绍
`redo log`充当了小黑板的作用，执行insert、update、delete语句时，会先写入`redo log buffer`中，之后再进行刷盘。

`redo log`分为`redo log buffer`和`redo log file`，buffer中的数据何时刷到file中是可以进行配置的，由`innodb_flush_log_at_trx_commit`配置项决定，为了保证数据正确，通常都是事务提交后立马刷到file中。

因为是先写log，再进行刷盘，所以产生了一个名词叫：WAL（Write-Ahead-Log），即先写日志后写数据。

`redo log`文件的大小及个数都是可以配置的，文末有配置说明。

**作用**

1. 解决了插入、更新数据时随机访问磁盘速度慢的问题。
2. 保证了事务提交后数据的持久性和正确性。

# 二、LSN

LSN（log sequence number）：日志序列号，是一个一直递增的整形数字，占8个字节。它表示事务写入到日志的字节总量（单位是byte）。LSN主要用于系统发生崩溃时对数据进行恢复！每个数据页、重做日志、checkpoint都有LSN。

使用`show engine innodb status\G`可以看到日志中`lsn`的信息，如下：

```sql
---
LOG
---
Log sequence number 433500797 # 当前系统最大的LSN
Log flushed up to   433500797 # 当前已经写入redo日志文件的 LSN
Pages flushed up to 433500797 # 已经将更改写入脏页的LSN号
Last checkpoint at  433500788 # 系统最后一次刷新buffer pool脏中页数据到磁盘的checkpoint
0 pending log flushes, 0 pending chkp writes
289 log i/o's done, 0.00 log i/o's/second
```

# 三、checkpoint

<img src="http://snail-resources.oss-cn-beijing.aliyuncs.com/1623842103.9139662p9C85jiZv1.png" alt="6F4285F2-F3A0-43B8-8BAC-D0ED89A96BA7" style="zoom: 50%;" />

- `write pos` 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。
- `checkpoint` 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。
- `write pos` 和 `checkpoint` 之间的是“黑板”上还空着的部分，可以用来记录新的操作。如果 `write pos` 追上 `checkpoint`，表示“黑板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

# 四、配置参数

## 1、**<u>innodb_flush_log_at_trx_commit</u>**

作用：控制 redo log 刷盘策略，将 redo log buffer 刷到 redo log file 中。

- 值为 0 时：日志每秒被写入和刷新到磁盘。
- 值为 1 时：**默认值**，每次事务提交时后会写入日志并刷盘。
- 值为 2 时：在每次事务提交后写入日志，并每秒刷新一次磁盘。

![47BBB576-3C28-44A7-B0B0-CCBB1E907393](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623842179.9929678g8ewOM3175.png)

## 2、**<u>innodb_log_buffer_size</u>**

缓冲的大小，配置的值越大，就可以避免大事务在事务提交前将日志写入磁盘。

- 默认值：16777216（16M）
- 最小值：1048576（1M）
- 最大值：4294967295（4G）

## 3、**<u>innodb_log_file_size</u>**

控制每个 redo log 文件的大小。

- 默认值：50331648（48M）
- 最小值：4194304（4M）
- 最大值：https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_log_buffer_size

默认值可以设置为 1G。

## 4、**<u>innodb_log_files_in_group</u>**

设置在一个组内可以有多少个重做日志文件。

- 默认值：2 个
- 最小值：2 个
- 最大值：100 个

注意：innodb_log_file_size * innodb_log_files_in_group < 512G。

## 5、**<u>innodb_log_group_home_dir</u>**

设置 redo log file 的目录。默认在 datadir 目录下。
