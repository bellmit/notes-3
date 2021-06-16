先创建两个表：

```sql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

Tips: 使用 `straight_join` 可以避免 MySQL 优化，改变驱动表，如：`select * from t1 straight_join t2 on t1.a=t2.a;`

# 一、join 算法

## 1、Index Nested-Loop Join（NLJ）

先看一下语句 `select * from t1 straight_join t2 on (t1.a=t2.a);` 的执行计划：

![4011BDB4-C385-4556-AC95-5950C5B4451E](http://snail-resources.oss-cn-beijing.aliyuncs.com/1623823318.5350049WTflHj6yUg.png)

驱动表 t2 的字段 a 上有索引，join 过程用上了这个索引，因此这个语句的执行流程如下：

1. 从表 t1 中读取一条数据 R。
2. 从数据 R 中取出字段 a 到表 t2 里查找。
3. 取出表 t2 满足条件的行，跟 R 组成一行，作为结果集的一部分。
4. 重复执行 1 到 3，直到表 t1 扫描结束。

在这个流程里：

1. 对驱动表 t1 做了全表扫描，这个过程需要扫描 100 行；

1. 而对于每一行 R，根据 a 字段去表 t2 查找，走的是树搜索过程。由于我们构造的数据都是一一对应的，因此每次的搜索过程都只扫描一行，也是总共扫描 100 行；
2. 所以，整个执行流程，总扫描行数是 200。

在这里，**驱动表走全表扫描**，**被驱动表走树搜索**。

到这里小结一下：

1. 使用 join 语句，性能比强行拆成多个单表执行 SQL 语句的性能要好；
2. 如果使用 join 语句的话，需要让小表做驱动表。

以上的结论都以 “可以使用被驱动表的索引” 为前提。

## 2、Simple Nested-Loop Join

**当被驱动表没有索引可以使用时**，如：`select * from t1 straight_join t2 on (t1.a=t2.b);` 由于表 t2 的字段 b 上没有索引，因此再用图 2 的执行流程时，每次到 t2 去匹配的时候，就要做一次全表扫描。

这里就是笛卡尔积了，这个 SQL 请求就要扫描表 t2 多达 100 次，总共扫描 100*1000=10 万行。

当然 MySQL 不会采用这种方法的。

## 3、Block Nested-Loop Join

这时候，被驱动表上没有可用的索引，算法的流程是这样的：

1. 把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 select *，因此是把整个表 t1 放入了内存；
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。

虽然和 Simple Nested-Loop Join 一样的逻辑，都是扫描 10万行，但是该算法每次都是在内存里判断，速度上取得优势。

问题：join_buffer 的大小是由参数 **join_buffer_size** 设定的，默认值是 256k。如果放不下表 t1 的所有数据话，策略很简单，就是分段放。

执行过程就变成了：

1. 扫描表 t1，顺序读取数据行放入 join_buffer 中，放完第 88 行 join_buffer 满了，继续第 2 步；
2. 扫描表 t2，把 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回；
3. 清空 join_buffer；
4. 继续扫描表 t1，顺序读取最后的 12 行数据放入 join_buffer 中，继续执行第 2 步。

这个流程才体现出了这个算法名字中“Block”的由来，表示“分块去 join”，这里要注意，当你分成 N 段时，就要对被驱动表进行 N 次全表扫描，所以如果你的 join 语句很慢，就把 join_buffer_size 改大，减小 N 的值。

# 二、如何选择驱动表

驱动表要选择小表，小表的意思是：**驱动表的数据占用的内存小**。

**例1**：

```sql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```

如果是用第二个语句的话，join_buffer 只需要放入 t2 的前 50 行，显然是更好的。所以这里，“t2 的前 50 行”是那个相对小的表，也就是“小表”。

**例2**：

```sql
select t1.b,t2.* from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```

表 t1 和 t2 都是只有 100 行参加 join。但是，这两条语句每次查询放入 join_buffer 中的数据是不一样的：

1. 表 t1 只查字段 b，因此如果把 t1 放到 join_buffer 中，则 join_buffer 中只需要放入 b 的值；
2. 表 t2 需要查所有的字段，因此如果把表 t2 放到 join_buffer 中的话，就需要放入三个字段 id、a 和 b。

这里，我们应该选择表 t1 作为驱动表。也就是说在这个例子里，“只需要一列参与 join 的表 t1”是那个相对小的表。

所以，更准确地说，在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。

# 三、总结

1. 执行计划中如果出现了 Block Nested-Loop Join，即 Extra 字段里面有没有出现“Block Nested Loop”字样，就可以使用 join。
2. 如果你的 join 语句很慢，就把 join_buffer_size 改大。









