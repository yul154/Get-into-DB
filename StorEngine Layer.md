# 存储引擎
> 负责MySQL中的数据的存储和提取,是与文件打交道的子系统.它是根据MySQL提供的文件访问层抽象接口定制的一种文件访问机制

MySQL 默认支持多种存储引擎，以适用于不同领域 的数据库应用需要，用户可以根据应用的需要选择如 何存储和索引数据、是否使用事务等。
* 支持的存储引擎包括 MyISAM,InnoDB,BD,MEMORY,MERGE,EXAMPLE,NDB Cluster、ARCHIVE、CSV、BLACKHOLE、FEDERATED 等，
* 其中 InnoDB 和 BDB 提供事务安全表，其他存储引擎都是非事务安全表

默认情况下，创建新表不指定表的存储引擎，则新表是默认存储引擎的，如果需要修改默认的存储引擎
* 在创建新表的时候，可以通过增加 ENGINE 关键字设置新建表的存储引擎
```
CREATE TABLE ai (i bigint(20) NOT NULL AUTO_INCREMENT, PRIMARY KEY (i)) ENGINE=MyISAM DEFAULT CHARSET=gbk;
-- 修改存储引擎
ALTER TABLE t ENGINE = InnoDB;
-- 修改默认存储引擎，也可以在配置文件my.cnf中修改默认引擎
SET default_storage_engine=NDBCLUSTER;
```

查看存储引擎
```
-- 查看支持的存储引擎
SHOW ENGINES

-- 查看默认存储引擎
SHOW VARIABLES LIKE 'storage_engine'

--查看具体某一个表所使用的存储引擎，这个默认存储引擎被修改了！
show create table tablename

--准确查看某个数据库中的某一表所使用的存储引擎
show table status like 'tablename'
show table status from database where name="tablename"
```

## 各种存储引擎的特性

### MyISAM

* 不支持事务
* 表级锁定，数据更新时锁定整个表：其锁定机制是表级锁定，这虽然可让锁定的实现成本很小可是也同时大大下降了其并发性能。
* 读写互相阻塞：不只会在写入的时候阻塞读取，myisam还会在读取的时候阻塞写入，但读自己并不会阻塞另外的读
* 读取速度较快，占用资源相对少. (MyIsam 则非聚集型索引)
* 不支持外键约束，但支持全文索引
* 对事务完整性没有要求或者以SELECT,INSERT为主的应用基本上都可以使用这个引擎来创建表
* 每个MyISAM类型的表都有一个AUTO_INCREMENT的内部列，当INSERT和UPDATE操作的时候该列被更新，同时AUTO_INCREMENT列将被刷新。所以说，MyISAM类型表的AUTO_INCREMENT列更新比InnoDB类型的AUTO_INCREMENT更快
* MyISAM格式的一个重要缺陷就是不能在表损坏后恢复数据。


使用场景
1. 不须要事务支持的业务（例如转帐就不行）
2. myisam没有事务支持，它的连续的插入和查询速度都比Innodb快很多
3. 做很多count 的计算
4. 插入不频繁，查询非常频繁。
5. 以读为主的业务，例如：数据库系统表、www， blog ，图片信息数据库，用户数据库，商品库等业务。
6. 对数据一致性要求不是很是高的业务（不支持事务）。
7. 硬件资源比较差的机器能够用 MyiSAM （占用资源少）

每个 MyISAM 在磁盘上存储成 3 个文件，其文件名都和表名相同
* `.frm`(存储表定义);
* `.MYD`(MYData，存储数据);
* `.MYI` (MYIndex，存储索引)。

MyISAM 类型的表可能会损坏，原因可能是多种多样的，损坏后的表可能不能访问，会提示需要修复或者访问后返回错误的结果。
* MyISAM 类型的表提供修复的工具，可以用`CHECK TABLE`语句来检查 MyISAM 表的健康，
* 用`REPAIR TABLE`语句修复一个损坏的 MyISAM 表。

|MyISAM存储格式|特点|
|-------------|---|
|静态(固定长度)表|默认的存储格式。静态表中的字段都是非变长字段，这种存储方式的优点是存储非常迅速，容易缓存，出现故障容易恢复;缺点是占用的空间通常比动态表多|
|动态表|包含变长字段,记录不是固定长度的,这样存储的优点是占用的空间相对较少,但是频繁地更新删除记录会产生碎片，需要定期执行 OPTIMIZE TABLE 语句命令来改善性能，|
|压缩表|由 myisampack 工具创建，占据非常小的磁盘空间。因为每个记录是被单独压缩的，所以只有非常小的访问开支|



### InnoDB
* 支持事务，此外 InnoDB 还实现了4 种隔离级别，使得对事务的支持更加灵活
* 提供了具有提交,回滚和崩溃恢复能力的事务安全
* 支持行级锁,行级锁可以最大程度的支持并发
* InnoDB 支持外键，为了保持数据的完整性
* InnoDB提供了专门的缓冲池，实现了缓冲管理，不仅能缓冲索引也能缓冲数据，常用的数据可以直接从内存中处理，比从磁盘获取数据处理速度要快

使用场景
* 可靠性要求比较高，或者要求事务
表更新和查询都相当的频繁，并且行锁定的机会比较大的情况。

自动增长列
* InnoDB表的自动增长列可以手工插入,但是插入的值如果是空或者0,则实际插入的将\是自动增长后的值
* 可以通过“ALTER TABLE *** AUTO_INCREMENT = n;”语句强制设置自动增长列的初识值
* 可以使用 LAST_INSERT_ID()查询当前线程最后插入记录使用的值
* 对于 InnoDB 表，自动增长列必须是索引。如果是组合索引，也必须是组合索引的第一列，对于MyISAM表,自动增长列可以是组合索引的其他列

外键约束
* MySQL 支持外键的存储引擎只有 InnoDB
* 在创建外键的时候，要求父表必须有对应的索引.
* 子表在创建外键的时候也会自动创建对应的索引.
* RESTRICT和NO ACTION相同，是指限制在子表有关联记录的情况下父表不能更新;
* CASCADE 表示父表在更新或者删除时，更新或者删除子表对应记录
* SET NULL 则表示父表在更新或者删除的时候，子表的对应字段被 SET NULL

InnoDB 物理文件结构为
* `.frm` 文件:与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
* `.ibd` 文件或 `.ibdata `文件：这两种文件都是存放InnoDB数据的文件，之所以有两种文件形式存放InnoDB的数据，是因为InnoDB的数据存储方式能够通过配置来决定是使用共享表空间存放存储数据，还是用独享表空间存放存储数据.

InnoDB 存储表和索引有以下两种方式
* 使用共享表空间存储,这种方式创建的表的表结构保存在`.frm`文件中,数据和索引保存在 `innodb_data_home_dir` 和 `innodb_data_file_path`定义的表空间中，可以是多个文件
* 使用多表空间存储，这种方式创建的表的表结构仍然保存在`.frm`文件中，但是每个表的数据和索引单独保存在`.ibd` 中。如果是个分区表,则每个分区对应单独的`.ibd` 文件，文件名是`表名+分区名`

使用多表空间特性的表,可以比较方便地进行单表备份和恢复操作,但是直接复制`.ibd` 文件是不行的，因为没有共享表空间的数据字典信息，直接复制的.ibd 文件和.frm 文 件恢复时是不能被正确识别的
```
 ALTER TABLE tbl_name DISCARD TABLESPACE; 
 ALTER TABLE tbl_name IMPORT TABLESPACE;
```

### MEMORY
> 将表中的数据存储到内存中

* 每个 MEMORY表只实际对应一个磁盘文件
* 格式是`.frm`。MEMORY 类型的表访问非常得快，因为它的数据是放在内存中的
* 并且默认使用HASH索引，但是一旦服务关闭，表中的数据就会丢失掉。
* MEMORY不支持BLOB或TEXT列

使用场景
* MEMORY类型的存储引擎主要用在那些内容变化不频繁的代码表，或者作为统计操作的中间结果表便于高效地对中间结果进行分析并得到最终的统计结果
* 对 MEMORY存储引擎的表进行更新操作要谨慎,因为数据并没有实际写入到磁盘中,所以一定要对下次重新 启动服务后如何获得这些修改后的数据有所考虑

### MERGE

MERGE 存储引擎是一组 MyISAM 表的组合
* 这些 MyISAM表必须结构完全相同，
* MERGE表本身并没有数据，
* 对MERGE类型的表可以进行查询、更新、删除的操作，这些操作实际上是对内部的实际的 MyISAM表进行的
* 对于MERGE类型表的插入操作，是通过 `INSERT_METHOD` 子句定义插入的表，可以有3个不同的值，使用`FIRST`或 `LAST` 值使得插入操作被相应地作用在第一或最后一个表上

MERGE 表在磁盘上保留两个文件
* 文件名以表的名字开始，一个`.frm`文件存储表定义
* 另一个`.MRG`文件包含组合表的信息，包括MERGE表由哪些表组成、插入新的数据时的依据

特点
* MERGE表和分区表的区别，MERGE表并不能智能地将记录写到对应的表中，而分区表是可以的
* 通常我们使用MERGE表来透明地对多个表进行查询和更新操作,而对这种按照时间记录的操作日志表则可以透明地进行插入操作
----

## 如何选择合适的存储引擎

MyISAM
* 如果应用是以读操作和插入操作为主只有很少的更新和删除操作 
* 并且对事务的完整性,并发性要求不是很高，那么选择这个存储引擎是非常适合的。
* MyISAM 是在 Web，数据仓储和其他应用环境下最常使用的存储引擎

InnoDB
* 用于事务处理应用程序，支持外键
* 如果应用对事务的完整性有比较高的要求
* 在并发条件下要求数据的一致性,数据操作除了插入和查询以外，还包括很多的更新,删除操作
* innoDB 存储引擎有效地降低由于删除和更新导致的锁定
* 需要频繁的更新、删除操作的数据库，也可以选择InnoDB还可以确保事务的完整提交(Commit)和回滚(Rollback)
* 对于类似计费系统或者财务系统等对数据准确性要求比较高的系统

MEMORY
* 将所有数据保存在RAM中,在需要快速定位记录和其他类似数据的环境下,可提供极快的访问。
* MEMORY的缺陷是对表的大小有限制，太大的表无法CACHE 在内存中，其次是要确保表的数据可以恢复，数据库异常终止后表中的数据是可以恢复的。 
* MEMORY表通常用于更新不太频繁的小表,用以快速得到访问结果。

MERGE
* 用于将一系列等同的MyISAM表以逻辑方式组合在一起,并作为一个对象引用它们。
* MERGE表的优点在于可以突破对单个MyISAM表大小的限制，并且通过将不同的表分布在多个磁盘上，可以有效地改善MERGE表的访问效率。
* 这对于诸如数据仓储等 VLDB 环境十分适合。

## 面试问题

>MyISAM 和 InnoDB 区别

|方面|MyISAM|InnoDb|
|----|-----|-------|
|事务和外键|InnoDB支持事务和外键，具有安全性和完整性，适合大量insert或update操作|MyISAM不支持事务和外键，它提供高速存储和检索，适合大量的select查询操作|
|索引结构|InnoDB 是聚簇索引,索引和记录在一起存储，既缓存索引，也缓存记录。|MyISAM 是非聚簇索引,索引和记录分开|
|并发处理能力|InnoDB 最小的锁粒度是行锁,读写阻塞可以与隔离级别有关，可以采用多版本并发控制（MVCC）来支持高并发|MyISAM最小的锁粒度是表锁,会导致写操作并发率低，读之间并不阻塞，读写阻塞|
|存储文件|InnoDB表对应两个文件，一个.frm表结构文件，一个.ibd数据文件。InnoDB表最大支持64TB|MyISAM表对应三个文件，一个.frm表结构文件，一个MYD表数据文件，一个.MYI索引文件|

> 适用场景
MyISAM
 * 不需要事务支持（不支持）
 * 并发相对较低（锁定机制问题）数据修改相对较少，以读为主
 * 数据一致性要求不高
 * 它提供高速存储和检索，以及全文搜索能力。如果应用中需要执行大量的SELECT查询
 
InnoDB
* 需要事务支持（具有较好的事务特性）
* 行级锁定对高并发有很好的适应能力
* 数据更新较为频繁的场景数据一致性
* 要求较高硬件设备内存较大，可以利用InnoDB较好的缓存能力来提高内存利用率，减少磁盘IO

* 索引结构:InnoDB 是聚簇索引，MyISAM 是非聚簇索引。
   * 聚簇索引的文件存放在主键索引的叶子节点上，因此InnoDB必须要有主键，通过主键索引效率很高。
   * 但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。
   * 主键不应该过大，因为主键太大，其他索引也都会很大。
   * MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。

* 是否需要事务？有，InnoDB
* 是否存在并发修改？有，InnoDB
* 是否追求快速查询，且数据修改少？是，MyISAM
* 在绝大多数情况下，推荐使用InnoDB

> 一张表，里面有ID自增主键，当insert了17条记录之后，删除了第15,16,17条记录，再把Mysql重启，再insert一条记录，这条记录的ID是18还是15 ？'
* 如果表的类型是MyISAM,那么是18.因为MyISAM表会把自增主键的最大ID 记录到数据文件中
* 如果表的类型是InnoDB，那么是15。因为InnoDB 表只是把自增主键的最大ID记录到内存中，所以重启数据库或对表进行OPTION操作，都会导致最大ID丢失

> 哪个存储引擎执行`select count(*)`更快，为什么?
* MyISAM更快，因为MyISAM内部维护了一个计数器，可以直接调取。在 MyISAM 存储引擎中，把表的总行数存储在磁盘上，当执行`select count(*) from t`时,直接返回总数据。
* 在InnoDB存储引擎中，跟 MyISAM 不一样，没有将总行数存储在磁盘上，当执行 `select count(*) from t`时，会先把数据读出来，一行一行的累加，最后返回总数量。
* 为什么InnoDB引擎不像 MyISAM 引擎一样，将总行数存储到磁盘上？这跟 InnoDB 的事务特性有关，由于多版本并发控制（MVCC）的原因，InnoDB 表“应该返回多少行”也是不确定的。


> 为什么MyISAM会比Innodb 的查询速度快。

1. 数据块，INNODB要缓存，MYISAM只缓存索引块，  这中间还有换进换出的减少； 
2. innodb寻址要映射到块，再到行，MYISAM 记录的直接是文件的OFFSET，定位比INNODB要快
3. INNODB还需要维护MVCC一致；虽然你的场景没有，但他还是需要去检查和维护