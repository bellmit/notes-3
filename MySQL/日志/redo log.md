# 一、有什么作用？

1. 为了保证事务提交后数据的正确性。
2. 解决了插入、更新数据时随机访问磁盘速度慢的问题。

# 二、WAL技术

当执行插入或更新操作时，数据先被写入 redo log，然后 InnoDB 引擎在合适的时候会把数据写入磁盘（数据页）。

**注意点**：

1. 写入 redo log 是顺序写入的，即使机械硬盘速度也很快。
2. redo log 分为 redo log buffer 和 redo log file，数据什么时候从 buffer 刷到 file 中是可以配置的。

由此产生了一个名词 Write-Ahead-Log(WAL) 即：先写日志再写数据。

举例：菜馆老板记账的故事。

# 三、LSN

LSN（log sequence number）：日志序列号，是一个一直递增的整形数字，在 MySQL5.6.3 版本后占8个字节。它表示事务写入到日志的字节总量。LSN主要用于发生 crash 时对数据进行 recovery ！每个数据页、重做日志、checkpoint都有LSN。

使用 show engine innodb status\G 可以看到日志中 lsn 的信息，如下：

```sql
---
LOG
---
Log sequence number 433500797
Log flushed up to   433500797
Pages flushed up to 433500797
Last checkpoint at  433500788
0 pending log flushes, 0 pending chkp writes
289 log i/o's done, 0.00 log i/o's/second
```

| 属性                    | 说明                                                    |
| ----------------------- | ------------------------------------------------------- |
| **Log sequence number** | 当前系统最大的LSN号                                     |
| **log flushed up to**   | 当前已经写入redo日志文件的LSN                           |
| **pages flushed up to** | 已经将更改写入脏页的LSN号                               |
| **Last checkpoint at**  | 系统最后一次刷新buffer pool脏中页数据到磁盘的checkpoint |

若将已经写入到 redo log 中的 lsn 记为 redo_lsn，将已经刷回到磁盘最新页的 lsn 记为 checkpoint_lsn，则可定义：

`checkpoint_age = redo_lsn - checkpoint_lsn`

再定义一下变量:

```
Async_water_mark = 75% * total_redo_log_file_size
Sync_water_mark = 90% * total_redo_log_file_size
```

当 `async_water_mark < checkpoint_age < sync_water_mark` 时就得进行刷盘了。

从这里可以发现，lsn 就是数据量的大小，单位是 byte。

# 四、checkpoint

<img src="http://snail-resources.oss-cn-beijing.aliyuncs.com/1623842103.9139662p9C85jiZv1.png" alt="6F4285F2-F3A0-43B8-8BAC-D0ED89A96BA7" style="zoom: 50%;" />

write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。

checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。

write pos 和 checkpoint 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

# 五、配置参数

## 1、**<u>innodb_flush_log_at_trx_commit</u>**

作用：控制 redo log 刷盘策略，将 redo log buffer 刷到 redo log file 中。

取值范围 0,1,2：

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



