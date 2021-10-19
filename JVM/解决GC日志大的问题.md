# 概述

随着程序一直稳定的运行，GC日志文件会越来越大的，可不可以像 log4j 的日志一样对日志进行切片呢？

当然可以，而且还是 JVM 提供的，只需要配置几个参数就可以了。

```
-Xloggc:gc-log.log // 指定日志文件
-XX:+UseGCLogFileRotation // 开启日志滚动
-XX:GCLogFileSize=512K // 设置每个日志的大小
-XX:NumberOfGCLogFiles=3 // 可以有多少个日志文件
```

如下是写完 3 个日志文件后，清空第一个文件，循环写的情况：

```
gc-log.log.0.current
gc-log.log.1
gc-log.log.2
```

但是！输出 GC 日志是通过 outputStream 进行输出的，在多线程情况下，会在安全点进行检查日志是否需要滚动，在安全点进行滚动是安全的，否则就得需要获取锁来进行滚动日志，例如：tty_lock。

下面是官方的回复：[https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6941923](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6941923)

> introduced three flags:
>
> 1) -XX:+UseGCLogRotation                       must be used with -Xloggc:<filename>
>
> 2) -XX:NumberOfGClogFiles=<number of files>    must be >=1, default is one
>
> 3) -XX:GCLogFileSize=<number>M (or K)          default will be set to 512K
>
> if UseGCLogRotation set and -Xloggc gives file name, do Log rotation, depends on other flag settings.
>
> if NumberOfGClogFiles = 1, using same file, rotate log when reaches file capicity (set by GCLogFileSize) in <filename>.0
>
> if NumberOfGClogFiles > 1, rotate between files <filename>.0, <filename>.1, ..., <filename>.n-1
>
> GC output through outputStream, we have multiple threads, vmThread, CMS threads, GC worker threads. Check if need rotation at every safepoint. When stop world (at safepoint), changing file handle is safe, else need grab lock to do the change. i.e. tty_lock.
