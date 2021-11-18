# 事务

> SQL事务隔离级别实现原理

标准SQL事务隔离级别的实现是依赖锁的

|事务隔离级别|实现方式 |
|----------|--------|
|未提交读（RU）|事务对当前被读取的数据不加锁；事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加行级共享锁，直到事务结束才释放|
|提交读（RC）|事务对当前被读取的数据加行级共享锁（当读到时才加锁），一旦读完该行，立即释放该行级共享锁，事务在更新某数据的瞬间（就是发生更新的瞬间),必须先对其加行级排他锁，直到事务结束才释放|
|可重复读（RR）|事务在读取某数据的瞬间(就是开始读取的瞬间)，必须先对其加行级共享锁，直到事务结束才释放;事务在更新某数据的瞬间（就是发生更新的瞬间),必须先对其加行级排他锁，直到事务结束才释放|
|序列化读(S)|事务在读取数据时，必须先对其加表级共享锁 ，直到事务结束才释放；事务在更新数据时，必须先对其加表级排他锁 ，直到事务结束才释放。|

> InnoDB事务隔离级别实现原理
在只使用锁来实现隔离级别的控制的时候,需要频繁的加锁解锁,而且很容易发生读写的冲突(例如在RC级别下，事务A更新了数据行1，事务B则在事务A提交前读取数据行1都要等待事务A提交并释放锁)
* 为了不加锁解决读写冲突的问题，MySQL引入了MVCC机制


* 锁定读：在一个事务中，主动给读加锁，如SELECT ... LOCK IN SHARE MODE 和 SELECT ... FOR UPDATE。分别加上了行共享锁和行排他锁
* 一致性非锁定读:InnoDB使用MVCC向事务的查询提供某个时间点的数据库快照.查询会看到在该时间点之前提交的事务所做的更改,而不会看到稍后或未提交的事务所做的更改(在开始了事务之后，事务看到的数据就都是事务开启那一刻的数据了，其他事务的后续修改不会在本次事务中可见)
  * Consistent read是InnoDB在RC和RR隔离级别处理SELECT语句的默认模式。
  * 一致性非锁定读不会对其访问的表设置任何锁

当前读:它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁
* 特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁， 读取的是最新数据

快照读:读取的是快照版本，也就是历史版本
  * 快照读的前提是隔离级别不是未提交读和序列化读级别，因为未提交读总是读取最新的数据行，而不是符合当前事务版本的数据行，而序列化读则会对表加锁
  * 简单的select操作，属于快照读，不加锁

|事务隔离级别|实现方式 |
|----------|--------|
|未提交读(RU)|事务对当前被读取的数据不加锁，都是当前读;事务在更新某数据的瞬间(就是发生更新的瞬间),必须先对其加行级共享锁，直到事务结束才释放.|
|提交读(RC)|事务对当前被读取的数据不加锁，且是快照读；事务在更新某数据的瞬间(就是发生更新的瞬间)，必须先对其加行级排他锁（Record），直到事务结束才释放.|
|可重复读(RR)|事务对当前被读取的数据不加锁，且是快照读;事务在更新某数据的瞬间（就是发生更新的瞬间),必须先对其加行级排他锁（Record，GAP，Next-Key），直到事务结束才释放。通过间隙锁，在这个级别MySQL就解决了幻读的问题通过快照，在这个级别MySQL就解决了不可重复读的问题|
|序列化读(S)|事务在读取数据时，必须先对其加表级共享锁 ，直到事务结束才释放，都是当前读;事务在更新数据时，必须先对其加表级排他锁 ，直到事务结束才释放。|


>  InnoDB 是怎么加锁的
隐式锁定:InnoDB在事务执行过程中，使用两阶段锁协议(不主动进行显示锁定的情况)
1. 随时都可以执行锁定，InnoDB会根据隔离级别在需要的时候自动加锁
2. 锁只有在执行commit或者rollback的时候才会释放，并且所有的锁都是在同一时刻被释放

显式锁定
```
select ... lock in share mode //共享锁
select ... for update //排他锁
lock table
unlock table
```

> 幻读到底包不包括了delete的情况?

标准SQL的RR级别是会对查到的数据行加行共享锁，所以这时候其他事务想删除这些数据行其实是做不到的，所以在RR下，不会出现因delete而出现幻读现象，也就是幻读不包含delete的情况


> MYSQL 怎么解决脏读
提高隔离级别 到 RC以上
* 事务对当前被读取的数据加行级共享锁（当读到时才加锁），一旦读完该行，立即释放该行级共享锁，事务在更新某数据的瞬间（就是发生更新的瞬间),必须先对其加行级排他锁，直到事务结束才释放
```
：set [作用域] transaction isolation level [事务隔离级别]，
set global transaction isolation level read committed; 
```

MVCC 快照读
* 当我们读取某个数据库在时间点的快照时，只能看到时间点之前提交更新的结果，不能看到时间点之后事务提交的更新结果。这样就避免了脏读问题

> MYSQL 怎么解决幻读

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁(Gap Lock)。

InnoDB 是采用 Next-key的行锁机制，解决幻读。
* RR隔离级别下间隙锁才有效，RC隔离级别下没有间隙锁；


> MVCC能解决了幻读问题?

when the same query produces different sets of rows at different times.

一般意义上，“幻象（phantom）”可被定义为：对于相同的区间查询，插入和删除操作使得对相同的区间查询操作返回不同的结果。如果这么定义幻象异常，那么MVCC下的可重复读（RR）是可以避免幻象的。

但是对于当前读的操作依然会出现幻读,RR级别产生幻读,往往都是直接或间接执行了「当前读」,导致ReadView更新,所以会读到新的提交

* 问题的本质是因为 update语句的查找阶段相当于select ... for update,这会更新事务A的 ReadView, 从而可以读到「其他事务已提交的修改」.
* 在快照读情况下，MySQL通过mvcc来避免幻读。
* 在当前读情况下，MySQL通过next-key来避免幻读


> MVCC 原理
在使用READ COMMITTD、REPEATABLE READ这两种隔离级别的事务在执行普通的SEELCT操作时访问记录的版本链的过程，这样就可以使不同事务的读-写、写-读操作并发执行，从而提升系统性能

Multi-Version Concurrency Control)的核心就是 版本链+ Read view，

* “MV”就是通过 Undo log来保存数据的历史版本，实现记录多版本，
* “CC”是通过 Read view来实现管理，通过 Read view原则来决定数据是否显示
* 同时针对不同的隔离级别,Read view的生成策略不同，也就实现了不同的隔离级别

UNDO LOG
* 数据表中的一行记录实际上有多个版本，每个版本有自己的事务 ID(row_trx_id)每次需要读取那个版本数据的时候，
* roll_pointer： 每次对某条记录进行改动时，这个隐藏列会存一个指针，可以通过这个指针找到该记录修改前的信息
* 我们每一次对数据记录的改动，MySQL都会记录一条日志，我们把它称作undo日志，每一条undo日志对应着也都有一个roll_pointer属性，rooll_pointer就指向undo log 
* 行记录快照是保存在 Undo Log 中，并且行记录快照是通过链被roll_pointer属性连接成一个链表串联起来，每个快照都保存了 trx_id (事务 ID)，如果要找到历史快照，只需要遍历回滚指针进行查找

ReadView 
* 中保存了当前活跃的事务列表。通过比较事务版本，可以判断当前行数据版本是不是对当前事务可见
* 一个事务开启，这个事务要查询数据，需要读取哪个版本的行记录呢？Read View 是可以帮助我们解决可见性问题的
* 主要包含当前MySQL中还有哪些活跃的读写事务，把它们的事务id放到一个列表中，我们把这个列表命名为为m_ids(一个数组）
* 如果当前行记录版本是 trx_id < 活跃的最小事务 ID (up_limit_id),说明行记录在这些活跃的事务创建前就已经提交，这个行记录对当前事务是可见的
* 如果当前行记录事务版本 trx_id > 活跃的最大事务 ID (low_limit_id) 说明，这个行记录是在事务后创建的，这个行记录对当前事务不可见
* 如果 up_limit_id < trx_id < low_limit_id，则有如下情况：
  *  若 row trx_id 在数组 trx_ids 中，表示这个版本是由还没提交的事务生成的，不可见。
  *  若 row trx_id 不在数组 trx_ids 中，表示这个版本是已经提交了的事务生成的，可见。

* READ COMMITTD在每一次进行普通SELECT操作前都会生成一个ReadView，而REPEATABLE READ只在第一次进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复这个ReadView就好了


> 事务的实现就是如何实现ACID特性。
* 事务的原子性是通过 undo log 来实现的
* 事务的持久性性是通过 redo log 来实现的
* 事务的隔离性是通过 (读写锁+MVCC)来实现的
* 而事务的一致性是通过原子性，持久性，隔离性来实现的

> mysql 怎么实现 事务的原子性

* 用于记录数据被修改前的信息,保存了事务发生之前的数据的一个版本，可以用于回滚
* undo log记录了数据在每个操作前的状态，用于记录数据被修改前的信息
* 如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作
* undo log不redo log不一样，它属于逻辑日志。它对SQL语句执行相关的信息进行记录
* Undo记录的是已部分完成并且写入硬盘的未完成的事务

> undo log 有什么作用
* 用于保障未提交事务的原子性:记录事务修改之前版本的数据信息，因此假如由于系统错误或者rollback操作而回滚的话可以根据undo log的信息将数据从逻辑上恢复至事务之前的状态
* 多个行版本控制(MVCC): 当读取的某一行被其他事务锁定时，它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息

> mysql 怎么实现 事务的持久性
* redo log是innodb层产生的
* 用来记录事务操作引起数据的变化，记录的是数据页的物理修改

> binlog(二进制日志)的作用

二进制日志binlog是服务层的日志
binlog主要记录数据库的变化情况，内容包括数据库所有的更新操作
因此有了binlog可以很方便的对数据进行复制和备份
binlog是基于point-in-time recovery，保证服务器可以基于时间点对数据进行恢复，或者对数据进行备份

1. 主从复制：在主库中开启Binlog功能，这样主库就可以把Binlog传递给从库，从库拿到Binlog后实现数据恢复达到主从数据一致性。
2. 数据恢复：用于数据库的基于时间点的还原,通过mysqlbinlog工具来恢复数据。


> Mysql怎么保证隔离性的？ 
利用的是锁和MVCC机制

> MySQL中的锁

按读写
* 独占锁：又称排它锁、X锁、写锁。X锁不能和其他锁兼容，只要有事务对数据上加了任何锁，其他事务就不能对这些数据再放置X了，同时某个事务放置了X锁之后，其他事务就不能再加其他任何锁了，只有获取排他锁的事务是可以对数据进行读取和修改。
* 共享锁：又称读锁、S锁。S锁与S锁兼容，可以同时放置。其他事务只能再对A加S锁，而不能X锁，其他事务可以读,但不能写,会阻塞其他事务修改表数据


按粒度
* 表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低。数据库引擎总是一次性同时获取所有需要的锁以及总是按相同的顺序获取表锁从而避免死锁。
* 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。行锁总是逐步获得的，因此会出现死锁。
* 页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。

行锁兼容矩阵
* 记录锁(Record Locks)：会去锁住索引记录,锁住一行记录.它封锁的是该行的索引记录。其他事务不能修改和删除加锁项
* 间隙锁(Gap Lock)：只锁间隙，前开后开区间(a,b)，对索引的间隙加锁，防止其他事务插入数据
* 临键锁(Next-Key Lock)：同时锁住记录和间隙，前开后闭区间(a,b]

> InnoDB引擎什么时候使用表锁？？
表锁与锁表的误区
* 只有正确通过索引条件检索数据（没有索引失效的情况），InnoDB才会使用行级锁，
* 否则InnoDB对表中的所有记录加锁，也就是将锁住整个表。
* 注意，这里说的是锁住整个表，但是Innodb并不是使用表锁来锁住表的，而是使用了下面介绍的Next-Key Lock来锁住整个表

> select for update有什么含义，会锁表还是锁行还是其他
* for update 仅适用于InnoDB，且必须在事务块(BEGIN/COMMIT)中才能生效。
* 在进行事务操作时，通过“for update”语句，MySQL会对查询结果集中每行数据都添加排他锁，其他线程对该记录的更新与删除操作都会阻塞。排他锁包含行锁、表锁


> 意向锁
为了允许行锁和表锁共存，解决表级锁和行级锁之间的冲突,实现多粒度锁机制，InnoDB 还有两种内部使用的意向锁(Intention Locks),这两种意向锁都是表锁
* 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。
* 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁
* 事务在申请行锁前，必须先申请表的意向锁，成功后再申请行锁

> 主从复制用过吗
> Mysql 日志的作用
> Mysql 引擎支持的锁
> 慢查询了解吗
* 设置一个阈值，将运行时间超过该值的所有SQL语句都记录到慢查询的日志文件中
* 使用 Explain 关键字可以模拟优化器执行SQL查询语句，从而知道 MySQL 是如何处理你的 SQL 语句的
```
slow_query_log	是否开启慢查询日志，ON表示开启，OFF表示关闭
slow-query-log-file	慢查询日志存储路径
long_query_time	当查询时间多于设定的阈值时，记录日志
log_queries_not_using_indexes	未使用索引的查询也被记录到慢查询日志中（可选项）
log_output	FILE 或者 TABLE，分别表示 记录到文件 或者 mysql.slow_log表中。也可同时设置，定义log_output='FILE,TABLE'
```
```
select_type
SIMPLE：简单的SELECT，不实用UNION或者子查询。
2） PRIMARY：最外层SELECT。
3） UNION：第二层，在SELECT之后使用了UNION。
4） DEPENDENT UNION：UNION语句中的第二个SELECT，依赖于外部子查询。
5） UNION RESULT：UNION的结果。

table
type： 
5） index：全索引扫描。
6） index_merge：查询中同时使用两个（或更多）索引，然后对索引结果进行合并（merge），再读取表数据。
7） index_subquery：子查询中的返回结果字段组合是一个索引（或索引组合），但不是一个主键或唯一索引。
8） rang：索引范围扫描。
9） ref：Join语句中被驱动表索引引用的查询。
10） ref_or_null：与ref的唯一区别就是在使用索引引用的查询之外再增加一个空值的查询。
11） system：系统表，表中只有一行数据
```


> Innodb 为啥用递增主键
* InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B+Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，这样就会形成一个紧凑的索引结构，近似顺序填满。由于每次插入时也不需要移动已有数据，因此效率很高
* 如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置：此时 MySQL 不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过OPTIMIZE TABLE 来重建表并优化填充页面