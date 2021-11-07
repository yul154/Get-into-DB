* 索引优化
* SQL优化

-----
# SQL 优化

## 优化 SQL 语句的一般步骤

1. 通过 show status 命令了解各种 SQL 的执行频率:通过以上几个参数，可以很容易地了解当前数据库的应用是以插入更新为主还是以查询操作为主
2. 定位执行效率较低的SQL语句
  * 通过慢查询日志定位那些执行效率较低的 SQL语句，用log-slow-queries选项启动时 
  * 可以使用show processlist命令查看当前MySQL在进行的线程，包括线程的状态、是否锁表等，可以实时地查看 SQL 的执行情况，同时对一些锁表操作进行优化
3.通过`EXPLAIN`分析低效SQL的执行计划,获取MySQL如何执行SELECT语句的信息
  * `select_type`:表示 SELECT的类型，常见的取值有SIMPLE(简单表，即不使用表连接或者子查询),PRIMARY(主查询，即外层的查询),UNION(第二个select出现在UNION之后),SUBQUERY(子查询中的第一个SELECT)
  * `table`:输出结果集的表
  * `type`:表示表的连接类型:system(表中仅有一行),const(表示通过索引一次就找到了)range(单表中的范围查询),index(index类型只遍历索引树)、all(将遍历全表找到匹配的行)

4.确定问题并采取相应的优化措施

## 索引问题

**MySQL如何使用索引**
* 对于创建的多列索引，只要查询的条件中用到了最左边的列，索引一般就会被使用
* 索引字段越小越好
* 在where从句，group by从句，order by从句及联表on从句中出现的列
* 对于使用 like 的查询，后面如果是常量并且只有%号不在第一个字符，索引才可能会 被使用
* 离散度较大的列，或者经常用到的列适合放在联合索引的前面

**避免不走索引的场景**
* 用or分割开的条件，如果or前的条件中的列有索引,而后面的列中没有索引,那么涉及到的索引都不会被用到
* 尽量避免在字段开头模糊查询，会导致数据库引擎放弃索引进行全表扫描
* 尽量避免使用in 和not in，会导致引擎走全表扫描
* 隐式类型转换造成不使用索引
* order by 条件要与where中条件一致，否则order by不会利用索引进行排序
* 如果列类型是字符串，那么一定记得在 where 条件中把字符常量值用引号引起来，否则的话即便这个列上有索引，MySQL 也不会用到的
* 尽量避免在where条件中等号的左侧进行表达式、函数操作，会导致数据库引擎放弃索引进行全表扫描
* 尽量避免进行null值的判断，会导致数据库引擎放弃索引进行全表扫描
* 正确使用hint优化语句

查看索引使用情况
* `Handler_read_key` 的值将很高，这个值代表了一个行被索引值读的次数，很低的值表明增加索引得到的性能改善不高，因为索引并不经常使用
* `Handler_read_rnd_next`的值高则意味着查询运行低效，并且应该建立索引补救
* 

### 两个简单实用的优化方法
1. 定期分析表和检查表
  * 使用ANALYZE TABLE语句来分析表
  * 使用CHECK TABLE语句来检查表。CHECK TABLE语句能够检查InnoDB和MyISAM类型的表是否存在错误
2. 定期优化表： MySQL中使用OPTIMIZE TABLE语句来优化表，OPTILMIZE TABLE语句只能优化表中的VARCHAR、BLOB或TEXT类型的字(可以将表中的空间碎片进行合并)

## 常用 SQL 的优化

### 大批量插入数据
MyISAM 存储引擎的表，在导入大量的数据到一个非空的 MyISAM 表时
```
ALTER TABLE tbl_name DISABLE KEYS; 
loading the data
ALTER TABLE tbl_name ENABLE KEYS;
```

因为 InnoDB 类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺 序排列，可以有效地提高导入数据的效率

### 优化`INSERT`语句
* 如果同时从同一客户插入很多行，尽量使用多个值表的 INSERT 语句，这种方式将大大缩减客户端与数据库之间的连接、关闭等消耗(`insert into test values(1,2),(1,3),(1,4)...`)
* 如果从不同客户插入很多行，能通过使用`INSERT DELAYED`语句得到更高的速度.`DELAYED`的含义是让`INSERT`语句马上执行，其实数据都被放在内存的队列中，并没有真正写入磁盘
* 如果进行批量插入，可以增加 bulk_insert_buffer_size 变量值的方法来提高速度，但是，这只能对 MyISAM 表使用

### SELECT语句其他优化
* 避免出现`select *`
* 避免出现不确定结果的函数
* 多表关联查询时,小表在前,大表在后,第一张表会涉及到全表扫描，所以将小表放在前面
* 用where字句替换HAVING字句,自上而下的顺序解析where子句。根据这个原理，应将过滤数据多的条件往前放

### 优化嵌套查询

* 永远小表驱动大表（小的数据集驱动大的数据集）
* 子查询可以被更有效率的连接(JOIN)替代
* 对于含有`OR`的查询子句，如果要利用索引，则`OR`之间的每个条件列都必须用到索引; 如果没有索引，则应该考虑增加索引


### 查询条件优化

* 对于复杂的查询，可以使用中间临时表 暂存数据

#### 优化`GROUP BY`语句
group by实质是先排序后进行分组，遵照索引建的最佳左前缀

* MySQL对所有`GROUP BY col1，col2....`的字段进行排序,用户想要避免排序结果的消耗，则可以指定`ORDER BY NULL`禁止排序
* where高于having，能写在where限定的条件就不要去having限定了
* 当无法使用索引列，增大 max_length_for_sort_data 参数的设置，增大sort_buffer_size参数的设置

### 优化`ORDER BY`语句
* `order by`子句，尽量使用Index方式排序，避免使用`FileSort`方式排序
* MySQL 支持两种方式的排序，`FileSort` 和 `Index`，`Index`效率高，它指MySQL扫描索引本身完成排序，`FileSort`效率较低
* `ORDER BY`满足两种情况，会使用Index方式排序；
  1. ORDER BY语句使用索引最左前列
  2. 使用where子句与ORDER BY子句条件列组合满足索引最左前列
* 尽可能在索引列上完成排序操作，遵照索引建的最佳最前缀
* 增大sort_buffer_size参数的设置
* 增大max_lencth_for_sort_data参数的设置



## 优化表的数据类型
* 应当遵循最小原则：一般情况下，应该尽量使用可以正确存储数据的最小数据类型.简单就好：简单的数据类型通常需要更少的CPU周期,尽量使用int而不是bigint
* 尽量避免NULL：通常情况下最好指定列为NOT NULL
* 可以使用函数`PROCEDURE ANALYSE()`对当前应用的表进行分析，该函数可 以对数据表中列的数据类型提出优化建议
