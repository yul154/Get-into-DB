* 索引优化
* SQL优化
* 分库分表

-----
# 性能分析
MySQL常见性能分析手段
* 慢查询日志
* EXPLAIN 分析查询
* profiling分析
* show命令查询系统状态及系统变量
---
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

---
## 索引优化
* 选择合适的索引列
* 找散列性大的，有标识度的字段做索引
* 索引的值尽量别太大
* 覆盖索引
* 联合索引
* 索引下推： 对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数
* .索引列不能参与计算，保持列“干净
* 使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本
* =和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式
---
## SQL 优化
* 覆盖索引
* 不建议使用%前缀模糊查询
* 避免在where子句中对字段进行表达式操作
* 避免隐式类型转换
---
## 数据表结构优化
* 选择合适的数据类型
 * 使用可存下数据的最小的数据类型。
 * 使用简单地数据类型
 * 尽量少用text

* 将字段很多的表分解成多个表
* 增加中间表
* 分解关联查询

---
### 性能瓶颈定位
* 通过 show 命令查看 MySQL 状态及变量，找到系统的瓶颈
```
Mysql> show status ——显示状态信息（扩展show status like ‘XXX’）

Mysql> show variables ——显示系统变量（扩展show variables like ‘XXX’）

Mysql> show innodb status ——显示InnoDB存储引擎的状态

Mysql> show processlist ——查看当前SQL执行，包括执行状态、是否锁表等

Shell> mysqladmin variables -u username -p password——显示系统变量

Shell> mysqladmin extended-status -u username -p password——显示状态信息
```

### Explain(执行计划)
> 使用 Explain 关键字可以模拟优化器执行SQL查询语句，从而知道 MySQL 是如何处理你的 SQL 语句的。分析你的查询语句或是表结构的性能瓶颈

执行计划包含的信息（如果有分区表的话还会有partitions）

![image](https://user-images.githubusercontent.com/27160394/140641340-525b1246-9606-47e6-949f-e1449a037f42.png)

* `id`:select查询的序列号,表示查询中执行select子句或操作表的顺序
 * id相同，执行顺序从上往下
 * id全不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
 * id部分相同，执行顺序是先按照数字大的先执行，然后数字相同的按照从上往下的顺序执行
* `select_type`:查询的类型，主要是用于区分普通查询、联合查询、子查询等复杂的查询
 * SIMPLE ：简单的select查询，查询中不包含子查询或UNION
 * PRIMARY：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
 * SUBQUERY：在select或where列表中包含了子查询
 * DERIVED：在from列表中包含的子查询被标记为derived（衍生）
 *  UNION：若第二个select出现在union之后
 *  UNION RESULT：从union表获取结果的select
* type 访问类型: 从最好到最差依次排列system>const>eq_ref>ref>fulltext>ref_or_null>index_merge >unique_subquery>index_subquery>range>index>ALL
  * system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现
  * const：表示通过索引一次就找到了,const用于比较primary key 或 unique 索引
  * range：只检索给定范围的行，使用一个索引来选择行
  * index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快
  * ALL：Full Table Scan，将遍历全表找到匹配的行
* key: 实际使用的索引，如果为NULL，则没有使用索引
* key_len表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度
* ref: 显示索引的那一列被使用了，如果可能，是一个常量const

### 慢查询日志
> MySQL 提供的一种日志记录，它用来记录在 MySQL 中响应时间超过阈值的语句，具体指运行时间超过 long_query_time 值的 SQL，则会被记录到慢查询日志中

* long_query_time 的默认值为10，意思是运行10秒以上的语句
* 默认情况下，MySQL数据库没有开启慢查询日志，需要手动设置参数开启

```
SHOW VARIABLES LIKE '%slow_query_log%'
mysql> set global slow_query_log='ON';
mysql> set global slow_query_log_file='/var/lib/mysql/hostname-slow.log';
mysql> set global long_query_time=2;

```
MySQL提供了日志分析工具mysqldumpslow,通过 mysqldumpslow --help 查看操作帮助信息
```

```

### Show Profile 分析查询
> Show Profile命令查看执行状态
* Show Profile 是 MySQL 提供可以用来分析当前会话中语句执行的资源消耗情况。可以用于SQL的调优的测量
* 默认情况下，参数处于关闭状态，并保存最近15次的运行结果
* 开启功能，默认是关闭，使用前需要开启 mysql>set profiling=1;
* 运行SQL
* 查看结果
