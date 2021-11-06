# Mysql basic knowledge
# SQL
# Data type
# 存储引擎
# 索引
# Transactions
# 锁机制
# 分区分库
# 性能优化
----
# Mysql 框架
![image](https://user-images.githubusercontent.com/27160394/140594674-19156356-f2ba-42d7-9401-b8cab5168324.png)

## 1. 网络连接层
客户端和连接服务。主要完成一些类似于连接处理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。

### 组件
* Connectors:负责与其他编程语言的sql语言进行交互

## 2.服务层
>负责查询处理和其他系统任务。
主要完成大部分的核心服务功能， 包括查询解析、分析、优化、缓存、以及所有的内置函数，所有跨存储

### 组件
* 连接池组建 (Connection Pool): MySQL会为每一个连接绑定一个线程,之后这个连接上的所有查询都在这个线程中执行.MySQL通常会缓存线程或者使用线程池,从而避免频繁的创建和销毁线程。
* 管理服务和工具组建 (Enterprise Management Services & Utilities)
* SQL接口组件（SQL Interface）: MySQL支持DML（数据操作语言）、DDL（数据定义语言）、存储过程、视图、触发器、自定义函数等多种SQL语言接口。
* 查询分析器（Parser): MySQL会解析SQL查询,并为其创建语法树,并根据数据字典丰富查询语法树,会验证该客户端是否具有执行该查询的权限.创建好语法树后,MySQL还会对SQl查询进行语法上的优化，进行查询重写
* 优化器组件（Optimizer）:语法解析和查询重写之后,MySQL会根据语法树和数据的统计信息对SQL进行优化,包括决定表的读取顺序、选择合适的索引等，最终生成SQL的具体执行步骤。这些具体的执行步骤里真正的数据操作都是通过预先定义好的存储引擎API来进行的，与具体的存储引擎实现无关
* 缓冲组件（Cache & Buffer）: MySQL内部维持着一些Cache和Buffer,比如Query Cache用来缓存一条Select语句的执行结果,如果能够在其中找到对应的查询结果,那么就不必再进行查询解析、优化和执行的整个过程了

## 3. 引擎层
> 存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信

MySQL可以动态安装或移除存储引擎，可以有多种存储引擎同时存在，可以为每个Table设置不同的存储引擎。存储引擎负责在文件系统之上，管理表的数据、索引的实际内容，同时也会管理运行时的Cache、Buffer、事务、Log等数据和功能

### 组件
* Pluggable Storage Engine:负责与具体的文件系统交互
  * MySQL可以动态安装或移除存储引擎，可以有多种存储引擎同时存在，可以为每个Table设置不同的存储引擎。
  * 存储引擎负责在文件系统之上，管理表的数据、索引的实际内容，同时也会管理运行时的Cache、Buffer、事务、Log等数据和功能 

### 存储引擎类别
* InnoDB 存储引擎：Mysql 5.5版本后默认的存储引擎，优点是支持事务，行级锁，外键约束，支持崩溃后的安全恢复；
* MyISAM 存储引擎：不支持事务和外键，支持全文索引（但只对英文有效），特点是查询速度快；
* Memory 存储引擎：数据放在内存当中（类似memcache）以便得到更快的响应速度，但是崩掉的话数据会丢失；
* NDB 存储引擎：主要用于Mysql Cluster分布式集群；
* Archive 存储引擎：有很好的压缩机制，用于文件归档，写入时会进行压缩；


## 4.存储层

是将数据存储在运行于该设备的文件系统之上，并完成与存储引擎的交互

### 组件
* 物理文件(File System,Files & Logs): 所有的数据，数据库、表的定义，表的每一行的内容，索引，都是存在文件系统上，以文件的方式存在的


## MySQL 的查询流程具体是？or 一条SQL语句在MySQL中如何执行的

1. 客户端请求
2. 连接器（验证用户身份，给予权限）
3. 查询缓存（存在缓存则直接返回，不存在则执行后续操作）
4. 分析器（对SQL进行词法分析和语法分析操作）
5. 优化器（主要对执行的sql优化选择最优的执行方案方法）
6. 执行器（执行时会先看用户是否有执行权限，有才去使用这个引擎提供的接口）
7. 去引擎层获取数据返回（如果开启查询缓存则会缓存查询结果）


## 一条SQL更新语句是如何执行的

1. 先连接数据库（连接器）
2. 在一个表上更新时，与该表有关的查询缓存会失效，所以该语句会将表T上所有的缓存结果都清空；
3. 将SQL语句进行词法分析，并检测SQL语法(分析器)
4. 然后优化对应的查询操作(优化器)
5. 执行器负责具体执行，找到这一行，进行更新

查询语句只需要返回查询结果即可，但是更新语句需要去真的修改数据库中的数据，所以更新语句相对来讲要复杂一些.更新流程还涉及两个重要的日志模块 redolog(重做日志)和 binlog(归档日志）

每一次的更新操作都需要写进磁盘,然后磁盘也要找到对应的那条记录,然后再更新,整个过程IO成本,查找成本都很高.MySQL的设计者Write-Ahead Logging，它的关键点就是先写日志，再写磁盘.
* 执行一条更新语句，InnoDB就会先把记录写到redo log里面，然后更新到内存，等到系统比较空闲的时候再写入磁盘。redo log的文件大小是固定的，是通过循环写的 实现的

*重做日志(redo log)和归档日志(binlog)*
Redo log
> 为了解决crash-safe问题而产生的，是一种物理日志
redo log是InnoDB引擎层的一种日志，是用来记录这个页"做了什么改动"
![image](https://user-images.githubusercontent.com/27160394/140596469-31603a55-eb39-48cb-8ff7-b7faa87a6b8b.png)
* redo log的文件大小是固定的，会循环写入文件,一边写一边后移，写到第3号文件末尾后就回到0号文件开头,checkpoint也是往后推移并且循环的,擦除记录前要把记录更新到数据文件
* write pos和checkpoint之间还空着的部分，可以用来记录新的操作
* 如果write pos追上checkpoin，表示redo log满了，这时候不能再执行新的更新，得停下来先擦掉一些记录

Binlog
> 一种逻辑日志，是Server层的一种日志，记录了所有的sql语句，主要是用来配合备份来恢复数据库的
只要我们有最近一次的备份和这期间完整的binlog就能够恢复数据库了
* binlog是追加写，写到一定大小后会切换到下一个，并不会覆盖以前的日志

两阶段提交
1. UPDATE语句的结果写入内存，同时将这个操作写入redo log，此时redo log处于prepare状态，并告知执行器随时可以提交事物
2. 执行器生成这个操作的binlog，并写入binlog日志. 
3. 执行器通知将之前处于prepare状态改为commit状态，更新完成。

两个阶段提交保证了redo log和binlog的一致性
* 先写redo log后写binlog,redo log会恢复crash的语句，但是如果用这产生时的binlog去恢复数据库就会丢失这条记录，此时两个日志恢复的数据库数据就产生了差异
* 先写binlog后写redo log,redo log中还没写,此时异常重启后这个事务是无效的,所以无法恢复,但是binlog中有这条数据,当用此时的binlog文件去恢复数据库的时候,就会比当前的数据库数据多一条记录。

执行器和InnoDB引擎在执行这个简单的update语句时的内部流程:
1. 执行器先找引擎取ID=2这一行，ID是主键，引擎直接用树搜索找到这一行，如果ID=2这一行所在的数据页本就在内存中，就直接返回给执行器，否则需要从磁盘读入再返回；
2. 执行器拿到引擎给的行数据后，将值加1，原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据；
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务
4. 执行器生成这个操作的binlog，并把binlog写入磁盘
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交（commit）状态，更新完成
6. Redo log的写入拆成了两个步骤：prepare和commit，这就是"两阶段提交"。
7. 
![image](https://user-images.githubusercontent.com/27160394/140596830-ff8ff742-f9cc-4106-b40a-a4c76dbbec67.png)

Note
* Redo log不是记录数据页“更新之后的状态”，而是记录这个页 “做了什么改动”。
* Binlog有两种模式，statement 格式的话是记sql语句,row格式会记录行的内容,记两条,更新前和更新后都有。

---
# Basic SQL
## SQL 分类
* DDL（Data Definition languages):定义了不同的数据段,数据库,表,列,索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter 等。
* DML(Data Manipulation Language)语句:CRUD=，并检查数据完整性，常用的语句关键字主要包括 insert、delete、udpate 和 select 等。
* DCL(Data Control Language)语句:用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别.主要的语句关键字包括 grant、revoke等。

## DDL
它和DML语言的最大区别是DML只是对表内部数据的操作,而不涉及到表的定义,结构的修改,更不会涉及到其他对象.DDL语句更多的被数据库管理员(DBA)所使用

<img width="420" alt="Screen Shot 2021-11-06 at 12 08 29 PM" src="https://user-images.githubusercontent.com/27160394/140597425-7305b30e-d1df-48eb-84c4-5a858b9581af.png">

* information_schema:主要存储了系统中的一些数据库对象信息。比如用户表信息、列信 息、权限信息、字符集信息、分区信息等。
* cluster:存储了系统的集群信息。
* mysql:存储了系统的用户权限信息。
* test:系统自动创建的测试数据库，任何用户都可以使用。

```
create database test1;
show databases;
uses test1;
show tables;
drop test1;
create table emp(ename varchar(10),hiredate date,sal decimal(10,2),deptno int(2));
desc emp;//需要查看一下表的定义
show create table emp 
drop table emp;
alter table emp modify ename varchar(20);
alter table emp add column age int(3);
alter table emp drop column age;
alter table emp change age age1 int(4); //can rename,but modify can't
alter table emp add birth date after ename;
alter table emp modify age int(3) first;
```
## DML
DML 操作是指对数据库中表记录的操作,是开发人员日常使用最频繁的操作
```
insert into emp (ename,hiredate,sal,deptno) values('zzx1','2000-01-01','2000',1);
insert into emp values('lisa','2003-02-01','3000',2);//没写的字段可以自动设置为 NULL、 默认值、自增的下一个数字
insert into dept values(5,'dept5'),(6,'dept6');
update emp set sal=4000 where ename='lisa';
delete from emp where ename='dony';
select * from emp;
select distinct deptno from emp;
select * from emp where deptno = 1;
select * from emp where deptno=1 and sal<3000;

// 排序和分页
select * from emp order by sal;//关键字默认是升序排列,
select * from emp order by deptno,sal desc;//如果排序字段的值一样，则值相同的字段按照第二个排序字段进行排序
select * from emp order by sal limit 3;
select * from emp order by sal limit 1,3;//limit 经常和 order by 一起配合使用来进行记录的分页显示

// 聚合: sum,count,max,mni(聚合函数)， GROUP BY（分类聚合),WHTH ROLLUP 对分类聚合后的结果进行再汇总,HAVING对分类后的结果再进行条件的过滤
select deptno,count(1) from emp group by deptno with rollup;// 既要统计各部门人数，又要统计总人数
select deptno,count(1) from emp group by deptno having count(1)>1;// having 和 where 的区别在于 having 是对聚合后的结果进行条件的过滤，而 where是在聚合前就对记录进行过滤

//表连接: 内连接和外连接,它们之间的最主要区别是內连接仅选出两张表中互相匹配的记录,而外连接会选出其他不匹配的记录
select ename,deptname from emp,dept where emp.deptno=dept.deptno;
select ename,deptname from emp left join dept on emp.deptno=dept.deptno;//外连接有分为左连接和右连接:包含所有的左边表中的记录甚至是右边表中没有和它匹配的记录

//子查询
select * from emp where deptno in(select deptno from dept);//如果子查询记录数唯一，还可以用=代替 in
select emp.* from emp ,dept where emp.deptno=dept.deptno;// 表连接在很多情况下用于优化子查询

// 记录联合(union): join 是两张表根据条件相同的部分合并生成一个记录集。union是产生的两个记录集(字段要一样的)并在一起，成为一个新的记录集 。
SELECT * FROM t1 UNION|UNION ALL SELECT * FROM t2 // UNION 是将 UNION ALL 后的结果进行一次 DISTINCT，去除重复记录后的结果。
```
##  DCL
主要是 DBA 用来管理系统中的对象权限时所使用
```
grant select,insert on sakila.* to 'z1'@'localhost' identified by '123
```
完整的顺序是：
1、FROM子句组装数据（包括JOIN）
2、WHERE子句进行条件筛选
3、GROUP BY分组
4、使用聚集函数进行计算；
5、HAVING筛选分组；
6、计算所有的表达式；
7、SELECT 的字段；
8、ORDER BY排序
9、LIMIT筛选
---
# MySQL 支持的数据类型
> MySQL 提供了多种数据类型，主要包括数值型、字符串类型、日期和时间类型
## 数值类型
MySQL 支持所有标准 SQL 中的数值类型
* 严格数值类型(`INTEGER、SMALLINT、DECIMAL 和 NUMERIC`)
* 近似数值数据类型(`FLOAT、REAL 和 DOUBLE PRECISION`)
* 增加了 `TINYINT、MEDIUMINT 和 BIGINT` 这 3 种长度不同的整型，
* 并增加了 `BIT` 类型，用来存放位数据

<img width="470" alt="Screen Shot 2021-11-06 at 12 57 46 PM" src="https://user-images.githubusercontent.com/27160394/140598537-ba4b44d5-d665-44c4-b7b5-c238a04e6b3c.png">

### 整形数据
对于整型数据,MySQL 还支持在类型名称后面的小括号内指定显示宽度
* 例如 int(5)表 示当数值宽度小于5位的时候在数字前面填满宽度，如果不显示指定宽度则默认为 int(11)
* 一般配合 zerofill 使用，顾名思义，zerofill 就是用“0”填充的意思，也就是在数字位数不够 的空间用字符“0”填满

所有的整数类型都有一个可选属性 UNSIGNED(无符号)，如果需要在字段里面保存非负数或者需要较大的上限值时，可以用此选项，
* 它的取值范围是正常值的下限取 0，上限取 原值的 2 倍
* tinyint 有符号范围是-128~+127，而无符号范围是 0~255。
* 如果一个列 指定为 zerofill，则 MySQL 自动为该列添加 UNSIGNED 属性

整数类型还有一个属性:AUTO_INCREMENT
* 在需要产生唯一标识符或顺序值时,可利用此属性
* 一个表中最多只能有一个AUTO_INCREMENT列。
* 对于任何想要使用AUTO_INCREMENT 的 列，应该定义为 NOT NULL
* 并定义为 PRIMARY KEY 或定义为 UNIQUE 键
```
CREATE TABLE AI(ID INT AUTO_INCREMENT NOT NULL ,PRIMARY KEY(ID));
```
###  小数数据
对于小数的表示，MySQL 分为两种方式:浮点数和定点数
* 浮点数包括 float(单精度) 和 double(双精度)
* 而定点数则只有 decimal 一种表示: 定点数在 MySQL 内部以字符串形 式存放，比浮点数更精确，适合用来表示货币等精度高的数据。

浮点数和定点数都可以用类型名称后加“(M,D)”的方式来进行表示，“(M,D)”表示该值一共显示M位数字(整数位+小数位)，其中D位位于小数点后面
* float 和 double 在不指定精度时，默认会按照实际的精度(由实际的硬件和操作系统决定
* 如果有精度和标度，则会自动将四舍五入后的结果插入,系统不会报错
```
float(7,4)列内插入999.00009 --> 999.0001//MySQL 保存值时进行四舍五入
```
`decimal`在不指定精度时，默认的整数位为 10，默认的小数位为 0。
 * 默认字段的小数位被截断
 * 并且如果数据超越了精度和标度值，系统则会报错

`BIT` 用于存放位字段值，BIT(M)可以用来存放多位二进制数，M 范围从 1~ 64，如果不写则默认为 1 位.
* 对于位字段，直接使用 SELECT 命令将不会看到结果
* 可以用 `bin()`(显示为二进制格式)或者`hex()`(显示为十六进制格式)函数进行读取

## 日期时间类型
<img width="493" alt="Screen Shot 2021-11-06 at 1 14 22 PM" src="https://user-images.githubusercontent.com/27160394/140598856-0e9fc08e-f730-478e-98f0-9edd97066f11.png">

|Data type| 使用场景|
|---------|---------|
|`DATE`|如果要用来表示年月日|
|`DATETIME`|如果要用来表示年月日时分秒|
|`TIME`|只用来表示时分秒|
|`TIMESTAMP`|需要经常插入或者更新日期为当前系统时间, 值返回后显示为“YYYY-MM-DD HH:MM:SS”格式的字符串，显示宽度固定为19个字符如果想要获得数字值,应在TIMESTAMP列添加+0|
|`YEAR`|表示年份,YEAR有2位或4位格式的年。默认是4位格式|

## 字符串类型
