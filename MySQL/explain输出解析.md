# 概述

还记得MySQL的组件吗？连接器、分析器、优化器、执行器，其中优化器的作用是告诉执行器如何执行，也就是优化器决定了索引的使用，决定了SQL会扫描多少行等。

下面我们就来了解SQL的执行计划。

# 一、先明白 explain 命令输出内容含义

英语好的同学直接看官方文档：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html

官方文档的输出格式的字段如下：

| Column                                                       | JSON Name       | Meaning                                        |
| :----------------------------------------------------------- | :-------------- | :--------------------------------------------- |
| [`id`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_id) | `select_id`     | The `SELECT` identifier                        |
| [`select_type`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_select_type) | None            | The `SELECT` type                              |
| [`table`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_table) | `table_name`    | The table for the output row                   |
| [`partitions`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_partitions) | `partitions`    | The matching partitions                        |
| [`type`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_type) | `access_type`   | The join type                                  |
| [`possible_keys`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_possible_keys) | `possible_keys` | The possible indexes to choose                 |
| [`key`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_key) | `key`           | The index actually chosen                      |
| [`key_len`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_key_len) | `key_length`    | The length of the chosen key                   |
| [`ref`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_ref) | `ref`           | The columns compared to the index              |
| [`rows`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_rows) | `rows`          | Estimate of rows to be examined                |
| [`filtered`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_filtered) | `filtered`      | Percentage of rows filtered by table condition |
| [`Extra`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_extra) | None            | Additional information                         |

## 1、id

select查询的序列号，如果行是UNION的结果集，则这个值是NULL，在这种情况下，table列的值如下：<union*`M`*,*`N`*>，M和N的值是id。

当Id相同时，按照从上到下的顺序执行查询，当id不同时，先执行id值大的查询。

## 2、select_type

查询类型，主要用来区分简单查询、复杂查询、联合查询等。它可以是下列值中的任何一个：

| `select_type` Value                                          | JSON Name                    | Meaning                                                      |
| :----------------------------------------------------------- | :--------------------------- | :----------------------------------------------------------- |
| `SIMPLE`                                                     | None                         | Simple [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) (not using [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html) or subqueries) |
| `PRIMARY`                                                    | None                         | Outermost [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) |
| [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html) | None                         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html) |
| `DEPENDENT UNION`                                            | `dependent` (`true`)         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html), dependent on outer query |
| `UNION RESULT`                                               | `union_result`               | Result of a [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html). |
| [`SUBQUERY`](https://dev.mysql.com/doc/refman/5.7/en/optimizer-hints.html#optimizer-hints-subquery) | None                         | First [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) in subquery |
| `DEPENDENT SUBQUERY`                                         | `dependent` (`true`)         | First [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) in subquery, dependent on outer query |
| `DERIVED`                                                    | None                         | Derived table                                                |
| `MATERIALIZED`                                               | `materialized_from_subquery` | Materialized subquery                                        |
| `UNCACHEABLE SUBQUERY`                                       | `cacheable` (`false`)        | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query |
| `UNCACHEABLE UNION`                                          | `cacheable` (`false`)        | The second or later select in a [`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html) that belongs to an uncacheable subquery (see `UNCACHEABLE SUBQUERY`) |

关于每一种类型的介绍，后续章节会说。

## 3、table

输出行所引用表的名称，可以是一下的值：

- <union`M`,`N`>：使用了UNION的结果集。
- <derived`N`>：使用了衍生表的结果集。
- <subquery`N`>：使用了materialized subquery的结果集。

## 4、partitions

The partitions from which records would be matched by the query. The value is `NULL` for nonpartitioned tables.

## 5、type

The join type.

## 6、possible_keys

可能使用的索引，如果这列是NULL，就应该需要优化了。

## 7、key

实际使用的索引，

可以使用 FORCE INDEX、USE INDEX 或 IGNORE INDEX 在你的查询中。

## 8、key_len

MySQL实际使用的索引的长度。

## 9、ref

显示索引的那一列被使用了，如果可能，是一个常量const。

## 10、rows

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数。

## 11、filtered

## 12、Extra

包含了MySQL如何解析查询的附加信息。

# 二、select type

## 1、SIMPLE

简单的select查询，不使用UNION或子查询。

如：`select emp_no from employees where emp_no=10003`。

## 2、PRIMARY

最外层的查询，如：`select * from employees where emp_no=(select emp_no from dept_emp where emp_no=10001)`。

执行计划如下：最外层的查询类型是PRIMARY。

| id   | select\_type | table     | partitions | type  | possible\_keys | key     | key\_len | ref   | rows | filtered | Extra       |
| :--- | :----------- | :-------- | :--------- | :---- | :------------- | :------ | :------- | :---- | :--- | :------- | :---------- |
| 1    | **PRIMARY**  | employees | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | NULL        |
| 2    | SUBQUERY     | dept\_emp | NULL       | ref   | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | Using index |

## 3、UNION

在一个 UNION 查询中，从第二个 SELECT 语句到最后一个 SELECT 语句的查询类型是 UNION。如：

```sql
explain
select emp_no from employees where emp_no=10001
union
select emp_no from employees where emp_no=10002
union
select emp_no from employees where emp_no=10003
union
select emp_no from employees where emp_no=10004
```

执行计划如下，为什么第一个查询类型是 PRIMARY？

| id   | select\_type | table                | partitions | type  | possible\_keys | key     | key\_len | ref   | rows | filtered | Extra           |
| :--- | :----------- | :------------------- | :--------- | :---- | :------------- | :------ | :------- | :---- | :--- | :------- | :-------------- |
| 1    | PRIMARY      | employees            | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | Using index     |
| 2    | **UNION**    | employees            | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | Using index     |
| 3    | **UNION**    | employees            | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | Using index     |
| 4    | **UNION**    | employees            | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | Using index     |
| NULL | UNION RESULT | &lt;union1,2,3,4&gt; | NULL       | ALL   | NULL           | NULL    | NULL     | NULL  | NULL | NULL     | Using temporary |

## 4、DEPENDENT UNION

同查询类型 UNION 类似，只不过该查询类型依赖于外部的查询，比如（此处只是示例，不要去追究SQL语句的意义）：

```sql
explain
select *
from dept_emp d
where d.dept_no='d004' and d.emp_no in (
    select emp_no from employees where emp_no=d.emp_no
    union
    select emp_no from employees where emp_no=d.emp_no
    union
    select emp_no from employees where emp_no=d.emp_no
    union
    select emp_no from employees where emp_no=d.emp_no
)
```

执行计划如下：

| id   | select\_type        | table                | partitions | type    | possible\_keys | key      | key\_len | ref                 | rows   | filtered | Extra                    |
| :--- | :------------------ | :------------------- | :--------- | :------ | :------------- | :------- | :------- | :------------------ | :----- | :------- | :----------------------- |
| 1    | PRIMARY             | d                    | NULL       | ref     | dept\_no       | dept\_no | 16       | const               | 128052 | 100      | Using where              |
| 2    | DEPENDENT SUBQUERY  | employees            | NULL       | eq\_ref | PRIMARY        | PRIMARY  | 4        | employees.d.emp\_no | 1      | 100      | Using where; Using index |
| 3    | **DEPENDENT UNION** | employees            | NULL       | eq\_ref | PRIMARY        | PRIMARY  | 4        | employees.d.emp\_no | 1      | 100      | Using where; Using index |
| 4    | **DEPENDENT UNION** | employees            | NULL       | eq\_ref | PRIMARY        | PRIMARY  | 4        | employees.d.emp\_no | 1      | 100      | Using where; Using index |
| 5    | **DEPENDENT UNION** | employees            | NULL       | eq\_ref | PRIMARY        | PRIMARY  | 4        | employees.d.emp\_no | 1      | 100      | Using where; Using index |
| NULL | UNION RESULT        | &lt;union2,3,4,5&gt; | NULL       | ALL     | NULL           | NULL     | NULL     | NULL                | NULL   | NULL     | Using temporary          |

## 5、UNION RESULT

从 UNION 的结果集中进行查询。如 <u>3、UNION</u> 中的示例，执行计划的最后一行是 UNION RESULT，输出的行是结果集中的行。

## 6、SUBQUERY

子查询类型。如 <u>2、PRIMARY</u> 中的示例，先执行子查询，然后执行外部查询。

## 7、DEPENDENT SUBQUERY

依赖于子查询的查询类型。如：`select * from employees t where t.emp_no = (select emp_no from employees where emp_no=t.emp_no)`。

执行计划如下：

| id   | select\_type           | table     | partitions | type    | possible\_keys | key     | key\_len | ref                 | rows   | filtered | Extra       |
| :--- | :--------------------- | :-------- | :--------- | :------ | :------------- | :------ | :------- | :------------------ | :----- | :------- | :---------- |
| 1    | PRIMARY                | t         | NULL       | ALL     | NULL           | NULL    | NULL     | NULL                | 298847 | 100      | Using where |
| 2    | **DEPENDENT SUBQUERY** | employees | NULL       | eq\_ref | PRIMARY        | PRIMARY | 4        | employees.t.emp\_no | 1      | 100      | Using index |

子查询的查询条件依赖于外部查询。

## 8、DERIVED

子查询位于 form 之后 where 之前，查询类型为衍生表。

MySQL5.7中引入了衍生表优化（默认开启），可以使用命令：`set optimizer_switch='derived_merge=off'` 进行关闭，关闭后执行：

```sql
explain select * from (select * from employees) as t where t.emp_no = 10002
```

执行计划如下：

| id   | select\_type | table            | partitions | type | possible\_keys     | key                | key\_len | ref   | rows   | filtered | Extra |
| :--- | :----------- | :--------------- | :--------- | :--- | :----------------- | :----------------- | :------- | :---- | :----- | :------- | :---- |
| 1    | PRIMARY      | &lt;derived2&gt; | NULL       | ref  | &lt;auto\_key0&gt; | &lt;auto\_key0&gt; | 4        | const | 10     | 100      | NULL  |
| 2    | DERIVED      | employees        | NULL       | ALL  | NULL               | NULL               | NULL     | NULL  | 298847 | 100      | NULL  |

如果开启了衍生表优化，则执行计划如下：

| id   | select\_type | table     | partitions | type  | possible\_keys | key     | key\_len | ref   | rows | filtered | Extra |
| :--- | :----------- | :-------- | :--------- | :---- | :------------- | :------ | :------- | :---- | :--- | :------- | :---- |
| 1    | SIMPLE       | employees | NULL       | const | PRIMARY        | PRIMARY | 4        | const | 1    | 100      | NULL  |

关于衍生表的优化，可以自行搜索，出现衍生表的查询语句效率会很低，查询类型中不应该出现衍生表。

## 9、MATERRIALIZED

中文：物化，使用子查询时，只对子查询查询一次，然后保存为一个临时表，后续对子查询结果集的访问直接访问临时表即可，与之相对的执行方式就是外表中的每一行数据都执行一次子查询，该查询类型就是 `DEPENDENT SUBQUERY`。

## 10、UNCACHABLE SUBQUERY

## 11、UNCACHABLE UNION

# 三、join type

## 1、system

The table has only one row (= system table). This is a special case of the [`const`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_const) join type.

## 2、const

表中有且只有一个匹配行时使用，对主键或唯一索引的查询，效率最高，将主键置于WHERE列表中，MySQL就能将该查询转换为一个const。

```sql
SELECT * FROM tbl_name WHERE primary_key=1;

SELECT * FROM tbl_name WHERE primary_key_part1=1 AND primary_key_part2=2;
```

## 3、eq_ref

唯一性索引或主键查找，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描。

```sql
SELECT * FROM ref_table,other_table WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table WHERE ref_table.key_column_part1=other_table.column AND ref_table.key_column_part2=1;
```

## 4、ref

使用非唯一性索引查找，返回匹配某个单独值的所有行。

```sql
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table WHERE ref_table.key_column_part1=other_table.column AND ref_table.key_column_part2=1;
```

## 5、fulltext

使用了全文索引。

## 6、ref_or_null

与 ref 类似，只不过这里包含了 NULL 值。

如：`SELECT * FROM *ref_table*  WHERE *key_column*=*expr* OR *key_column* IS NULL`。

## 7、index_merge

索引合并优化方法。

## 8、unique_subquery

子查询中返回唯一索引。This type replaces [`eq_ref`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_eq_ref) for some `IN` subqueries of the following form： `value IN (SELECT primary_key FROM single_table WHERE some_expr)`。

`unique_subquery` 只是一个索引查找函数，它完全替换子查询以提高效率。

## 9、index_subquery

子查询中返回了非唯一索引。This join type is similar to [`unique_subquery`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_unique_subquery). It replaces `IN` subqueries, but it works for nonunique indexes in subqueries of the following form：`value IN (SELECT key_column FROM single_table WHERE some_expr)`。

## 10、range

范围查询，在使用 <u>=</u>, <u><></u>, <u>></u>, <u>>=</u>, <u><</u>, <u><=</u>, <u>IS NULL</u>, <u><=></u>, <u>BETWEEN</u>, <u>LIKE</u>, or <u>IN()</u> 任何一个运算符时。

```sql
SELECT * FROM tbl_name WHERE key_column = 10;

SELECT * FROM tbl_name WHERE key_column BETWEEN 10 and 20;

SELECT * FROM tbl_name WHERE key_column IN (10,20,30);

SELECT * FROM tbl_name WHERE key_part1 = 10 AND key_part2 IN (10,20,30);
```

## 11、index

遍历非聚簇索引树。

## 12、ALL

遍历聚簇索引树。

# 四、Extra信息

## 1、Using where

使用 where 条件限制那些行被查询出来。

## 2、Using temporary

为了解决查询，MySQL 需要创建一个临时表来保存结果。 如果查询包含以不同方式列出列的 GROUP BY 和 ORDER BY 子句，则通常会发生这种情况。

## 3、Using filesort

使用文件排序

## 4、Using index

使用了覆盖索引，避免了回表。

## 5、Using index condition

使用了索引下推优化。

## 6、Using join buffer (Block Nested Loop)`, `Using join buffer (Batched Key Access)

使用 join buffer。

## 7、Using MRR

Tables are read using the Multi-Range Read optimization strategy.

## 8、Not exists

使用not exists优化查询。

## 9、No tables used

没有从表里查询数据。

## 10、Distinct

优化distinct操作，在找到第一个匹配的元祖后即停止找同样值得动作。
