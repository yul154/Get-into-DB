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
### MySQL查询
> `count(*)`和`count(1)`和`count(列名)`区别
* 如果该表只有一个主键索引,没有任何二级索引的情况下,那么`COUNT(*)`和`COUNT(1)`都是通过通过主键索引来统计行数的。
* 如果该表有二级索引，则`COUNT(1)`和`COUNT(*)`都会通过占用空间最小的字段的二级索引进行统计,也就是说虽然COUNT(1)指定了第一列（此处表达有误，详见文章结尾）但是innodb不会真的去统计主键索引

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
> MySQL 提供了多种数据类型,主要包括数值型、字符串类型、日期和时间类型
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

#### 选择场景
* 浮点型插入数据的精度超过该列定义的实际精度，则插入值会被四舍五入到实际定义的精度值
* 定点数不同于浮点数,定点数实际上是以字符串形式存放的,所以定点数可以更加精确的保存数据
* 编程中应尽量避免浮点数的比较，如果非要使用浮点数比较则 最好使用范围比较而不要使用“==”比较


## 日期时间类型
<img width="493" alt="Screen Shot 2021-11-06 at 1 14 22 PM" src="https://user-images.githubusercontent.com/27160394/140598856-0e9fc08e-f730-478e-98f0-9edd97066f11.png">

|Data type| 使用场景|
|---------|---------|
|`DATE`|如果要用来表示年月日|
|`DATETIME`|如果要用来表示年月日时分秒|
|`TIME`|只用来表示时分秒|
|`TIMESTAMP`|需要经常插入或者更新日期为当前系统时间, 值返回后显示为“YYYY-MM-DD HH:MM:SS”格式的字符串，显示宽度固定为19个字符如果想要获得数字值,应在TIMESTAMP列添加+0|
|`YEAR`|表示年份,YEAR有2位或4位格式的年。默认是4位格式|


### 选择优化
* 根据实际需要选择能够满足应用的最小存储的日期类型.如果应用只需要记录“年份”,那么用1个字节来存储的`YEAR`类型完全可以满足,而不需要用`4`个字节来 存储的 DATE 类型。
* 如果要记录年月日时分秒，并且记录的年份比较久远，那么最好使用`DATETIME`， 而不要使用`TIMESTAM`。因为`TIMESTAMP`表示的日期范围比`DATETIME`要短得多
* 如果记录的日期需要让不同时区的用户使用，那么最好使用`TIMESTAMP`，因为日期类型中只有它能够和实际时区相对应


## 字符串类型
<img width="467" alt="Screen Shot 2021-11-06 at 1 27 17 PM" src="https://user-images.githubusercontent.com/27160394/140599097-e7de57aa-a86f-42b9-b525-dbac68e10cf6.png">

### CHAR 和 VARCHAR 类型
> 用来保存 MySQL 中较短的字符串
二者的主要区别在于存储方式的不同:
* `CHAR` 列的长度固定为创建表时声明的长度，长度可以为从 0~255 的任何值;
* `VARCHAR`列中的值为可变长字符串，长度可以指定为0~255(5 5.0.3以前)或者6553(5 5.0.3 以后)之间的值
* 在检索的时候，CHAR 列删除了尾部的空格，而 VARCHAR 则保留这些空格

#### 选择合适的数据类型
* 由于CHAR是固定长度的,所以它的处理速度比VARCHAR快得多，但是其缺点是浪费存储空间,程序需要对行尾空格进行处理,所以对于那些长度变化不大并且对查询速度有较高要求的数据可以考虑使用CHAR类型来存储
* MyISAM 存储引擎:建议使用固定长度的数据列代替可变长度的数据列
* MEMORY 存储引擎:目前都使用固定长度的数据行存储,两者都是作为 CHAR 类型处理
* InnoDB 存储引擎:建议使用`VARCHAR`类型。对于InnoDB数据表,内部的行存储格式没有区分固定长度和可变长度列,使用固定长度的 CHAR 列不一定比使用可变长度VARCHAR列性能要好


### BINARY 和 VARBINARY
类似于 CHAR 和 VARCHAR,它们包含二进制字符串 而不包含非二进制字符串


### TEXT 与 BLOB
> 在保存较大文本时， 通常会选择使用 TEXT 或者 BLOB，二者之间的主要差别是 BLOB 能用来保存二进制数据，比如照片;而TEXT只能保存字符数据,比如一篇文章或者日记

性能提升
* 删除操作会在数据表中留下很大的“空洞”，以后填入这些“空洞”的记录在插入的性能上会有影响。为了提高性能,建议定期使用`OPTIMIZE TABLE`功能对这类表进行碎片整理
* 可以使用合成的(Synthetic)索引来提高大文本字段(BLOB 或 TEXT)的查询性能.合成索引就是根据大文本字段的内容建立一个散列值,并把这个值存储在单独的数据列中,接下来就可以通过检索散列值找到数据行了
* 如果需要对BLOB或者CLOB字段进行模糊查询，MySQL提供了前缀索引，也就是只为字段的前n列创建索引, “%” 不能放在最前面，否则索引将不会被使用。
* 在不必要的时候避免检索大型的 BLOB 或 TEXT 值。
* 把BLOB或TEXT列分离到单独的表中,如果把这些数据列移动到第二张数据表中,可以把原数据表中的数据列转换为固定长度的数据行格式


### ENUM 类型
它的值范围需要在创建表时通过枚举方式显式指定,对 1~ 255个成员的枚举需要1个字节存储;对于 255~65535 个成员，需要2个字节存储
* 忽略大小写的，都转成了大写，还可以看出对于插入不在 ENUM 指定范围内的值时，并没有返回警告，而是插入第一值
* ENUM 类型只允许从值集合中选取单个值，而不能一次取多个值

### SET 类型
是一个字符串对象，里面可以包含 0~64 个成员
```
Create table t (col set ('a','b','c','d');
insert into t values('a,b'),('a,d,a'),('a,b'),('a,c'),('a');
```
* SET 类型可以从允许值集合中选择任意 1 个或多个元素进行组合
---
# MySQL 中的运算符
> 来连接表达式的项

算术运算符、比较 运算符、逻辑运算符和位运算符

## 算术运算符
<img width="405" alt="Screen Shot 2021-11-06 at 1 38 50 PM" src="https://user-images.githubusercontent.com/27160394/140599258-940ded9f-a575-4506-b8c4-c0b0eb475b97.png">

## 比较运算符
<img width="413" alt="Screen Shot 2021-11-06 at 1 41 22 PM" src="https://user-images.githubusercontent.com/27160394/140599317-2a67c454-8800-4685-af61-37765197080c.png">
* 比较运算符可以用于比较数字、字符串和表达式。数字作为浮点数比较，而字符串以不 区分大小写的方式进行比较
* `“=”`运算符，用于比较运算符两侧的操作数是否相等，如果两侧操作数相等返回值为1,否则为 0。注意`NULL`不能用于`“=”`比较
* <=>”安全的等于运算符，和“=”类似,不同之处在于即使 操作的值为 NULL 也可以正确比较
* `“BETWEEN”`运算符的使用格式为`“a BETWEEN min AND max”`
* `“IN”`运算符的使用格式为`“a IN (value1,value2,...)`
* `“LIKE”`运算符的使用格式为`“a LIKE %123%”`,当 a 中含有字符串“123”时，则返回 值为 1，否则返回 0
* `REGEXP`运算符的使用格式为`“str REGEXP str_pat”,`当 `str` 字符串中含有 `str_pat` 相匹配的字符串时，则返回值为 1，否则返回0

## 逻辑运算符
* `NOT` -> `!`,就是 NOT NULL 的返回值为 NULL
* `AND` -> `&&`
* `OR` -> `||`,假如两个操作数均为 NULL，则所得结果 为 NULL
* `XOR`

## 位运算符
<img width="373" alt="Screen Shot 2021-11-06 at 1 48 37 PM" src="https://user-images.githubusercontent.com/27160394/140599449-b4179fd3-3beb-4c4e-9553-468876309e72.png">

<img width="554" alt="Screen Shot 2021-11-06 at 1 49 52 PM" src="https://user-images.githubusercontent.com/27160394/140599471-f66d3dd9-1cf6-4c24-8d6a-7643b745a17d.png">
---
# 常用函数
MySQL 提供了多种内建函数帮助开发人员编写简单快捷的SQL语句，其中常用的函数有字符串函数,日期函数和数值函数

## 字符串函数
* CANCAT(S1,S2,...Sn)函数:把传入的参数连接成为一个字符串。
* INSERT(str ,x,y,instr)函数:将字符串 str 从第 x 位置开始，y 个字符长的子串替换为字符 串 instr
* LOWER(str)和 UPPER(str)函数:把字符串转换成小写或大写
* LEFT(str,x)和 RIGHT(str,x)函数:分别返回字符串最左边的x个字符和最右边的x个字符。 如果第二个参数是 NULL，那么将不返回任何字符串
* LPAD(str,n ,pad)和 RPAD(str,n ,pad)函数:用字符串 pad 对 str 最左边和最右边进行填充, 直到长度为 n 个字符长度
* LTRIM(str)和 RTRIM(str)函数:去掉字符串 str 左侧和右侧空格,TRIM(str)函数:去掉目标字符串的开头和结尾的空格。
* REPEAT(str,x)函数:返回 str 重复 x 次的结果。
* REPLACE(str,a,b)函数:用字符串 b 替换字符串 str 中所有出现的字符串 a。
* SUBSTRING(str,x,y)函数:返回从字符串 str 中的第 x 位置起 y 个字符长度的字串。

## 数值函数
<img width="367" alt="Screen Shot 2021-11-06 at 1 54 21 PM" src="https://user-images.githubusercontent.com/27160394/140599581-1871d7a7-46e4-4841-a367-76c313b7cf26.png">

## 日期和时间函数
<img width="399" alt="Screen Shot 2021-11-06 at 1 55 42 PM" src="https://user-images.githubusercontent.com/27160394/140599618-6686f73d-00bf-4cd0-bfa6-cade8f71bd13.png">

## 流程函数
<img width="309" alt="Screen Shot 2021-11-06 at 1 57 00 PM" src="https://user-images.githubusercontent.com/27160394/140599646-765a8902-859b-48f3-8d8d-0dbc3aa8911b.png">

```
select case salary when 1000 then 'low' when 2000 then 'mid' else 'high' end from salary;
```
---
# 存储引擎
> 负责MySQL中的数据的存储和提取,是与文件打交道的子系统.它是根据MySQL提供的文件访问层抽象接口定制的一种文件访问机制

MySQL 默认支持多种存储引擎，以适用于不同领域 的数据库应用需要，用户可以根据应用的需要选择如 何存储和索引数据、是否使用事务等。

MySQL 5.0 支持的存储引擎包括 MyISAM、InnoDB、BDB、MEMORY、MERGE、EXAMPLE、 NDB Cluster、ARCHIVE、CSV、BLACKHOLE、FEDERATED 等，其中 InnoDB 和 BDB 提供事务安 全表，其他存储引擎都是非事务安全表

默认情况下，创建新表不指定表的存储引擎，则新表是默认存储引擎的，如果需要修改 默认的存储引擎
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
* 不支持外键
* 优势是访问的速度快
* 对事务完整性没有要求或者以SELECT,INSERT为主的应用基本上都可以使用这个引擎来创建表

每个 MyISAM 在磁盘上存储成 3 个文件，其文件名都和表名相同
* `.frm`(存储表定义);
* `.MYD`(MYData，存储数据);
* `.MYI` (MYIndex，存储索引)。


MyISAM 类型的表可能会损坏，原因可能是多种多样的，损坏后的表可能不能访问，会提示需要修复或者访问后返回错误的结果。
* MyISAM 类型的表提供修复的工具，可以用 CHECK TABLE 语句来检查 MyISAM 表的健康，
* 用 REPAIR TABLE 语句修复一个损坏的 MyISAM 表。

|MyISAM存储格式|特点|
|-------------|---|
|静态(固定长度)表|默认的存储格式。静态表中的字段都是非变长字段，这种存储方式的优点是存储非常迅速，容易缓存，出现故障容易恢复;缺点是占用的空间通常比动态表多|
|动态表|包含变长字段,记录不是固定长度的,这样存储的优点是占用的空间相对较少,但是频繁地更新删除记录会产生碎片，需要定期执行 OPTIMIZE TABLE 语句命令来改善性能，|
|压缩表|由 myisampack 工具创建，占据非常小的磁盘空间。因为每个记录是被单独压缩的，所以只有非常小的访问开支|

### InnoDB
* 提供了具有提交、回滚和崩溃恢复能力的事务安全
* 对比MyISAM的存储引擎,InnoDB写的处理效率差一些并且会占用更多的磁盘空间以保留数据和索引。

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

使用多表空间特性的表,可以比较方便地进行单表备份和恢复操作,但是直接复制`.ibd` 文件是不行的《因为没有共享表空间的数据字典信息，直接复制的.ibd 文件和.frm 文 件恢复时是不能被正确识别的
```
 ALTER TABLE tbl_name DISCARD TABLESPACE; 
 ALTER TABLE tbl_name IMPORT TABLESPACE;
```
### MEMORY
MEMORY存储引擎使用存在内存中的内容来创建表.
* 每个 MEMORY表只实际对应一个磁盘文件
* 格式是`.frm`。MEMORY 类型的表访问非常得快，因为它的数据是放在内存中的
* 并且默认使用HASH索引，但是一旦服务关闭，表中的数据就会丢失掉。

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


## 如何选择合适的存储引擎

MyISAM
* 如果应用是以读操作和插入操作为主只有很少的更新和删除操作 
* 并且对事务的完整性,并发性要求不是很高，那么选择这个存储引擎是非常适合的。
* MyISAM 是在 Web，数据仓储和其他应用环境下最常使用的存储引擎

InnoDB
* 用于事务处理应用程序，支持外键
* 如果应用对事务的完整性有比较高的要求
* 在并发条件下要求数据的一致性,数据操作除了插入和查询以外，还包括很多的更新,删除操作
* innoDB 存储引擎有效地降低 由于删除和更新导致的锁定
* 还可以确保事务的完整提交(Commit)和回滚(Rollback)
* 对于类似计费系统或者财务系统等对数据准确性要求比较高的系统

MEMORY
* 将所有数据保存在RAM中,在需要快速定位记录和其他类似数据的环境下,可提供极快的访问。
* MEMORY的缺陷是对表的大小有限制，太大的表无法 CACHE 在内 存中，其次是要确保表的数据可以恢复，数据库异常终止后表中的数据是可以恢复的。 
* MEMORY表通常用于更新不太频繁的小表,用以快速得到访问结果。

MERGE
* 用于将一系列等同的MyISAM表以逻辑方式组合在一起,并作为一个对象引用它们。
* MERGE表的优点在于可以突破对单个MyISAM表大小的限制，并且通过将不同的表分布在多个磁盘上，可以有效地改善MERGE表的访问效率。
* 这对于诸如数据仓储等 VLDB 环境十分适合。

## 面试问题

>MyISAM 和 InnoDB 区别
>
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
---
# 索引
> 索引（Index）是帮助MySQL高效获取数据的数据结构

索引的本质是：数据结构
* 索引的目的在于提高查询效率
* 可以简单的理解为“排好序的快速查找数据结构”
* 数据本身之外，数据库还维护者一个满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据
* 这样就可以在这些数据结构上实现高级查找算法。
* 索引本身也很大，不可能全部存储在内存中，一般以索引文件的形式存储在磁盘

平常说的索引，没有特别指明的话，就是B+树（多路搜索树，不一定是二叉树）结构组织的索引. 其中聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引

MyISAM 和 InnoDB 存储引擎的表默认创建的都是`BTREE`索引。MySQL目前还不支持函数索引，但是支持前缀索引，即对索引字段的前 N 个字符创建索引.

MySQL 中还支持全文本(FULLTEXT)索引，该索引可以用于全文搜索。但是当前最新版 本中(5.0)只有 MyISAM 存储引擎支持 FULLTEXT 索引，并且只限于 CHAR、VARCHAR 和 TEXT 列。索引总是对整个列进行的，不支持局部(前缀)索引.

MEMORY 存储引擎使用 HASH 索引，但也支持 BTREE 索引。

基本语法: 
```
CREATE [UNIQUE] INDEX indexName ON mytable(username(length));
create index cityname on city (city(10));//要为 city 表创建了 10 个字节的前缀索引
ALTER table tableName ADD [UNIQUE] INDEX indexName(columnName);
ALTER TABLE tbl_name ADD PRIMARY KEY (column_list)// 该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。
ALTER TABLE tbl_name ADD FULLTEXT index_name (column_list) // 该语句指定了索引为 FULLTEXT ，用于全文索引。
drop index cityname on city;
SHOW INDEX FROM table_name\G  //可以通过添加 \G 来格式化输出信息。
```
优势
* 提高数据检索效率
* 降低数据库IO成本降低数据排序的成本
* 降低CPU的消耗

劣势
* 索引也是一张表，保存了主键和索引字段，并指向实体表的记录，所以也需要占用内存
* 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。
* 因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新所带来的键值变化后的索引信息

## MySQL索引分类

数据结构角度
* B+树索引
* Hash索引
* Full-Text全文索引
* R-Tree索引

从物理存储角度
* 聚集索引（clustered index）
* 非聚集索引（non-clustered index），也叫辅助索引(secondary index)聚集索引和非聚集索引都是B+树结构

从逻辑角度

* 主键索引：主键索引是一种特殊的唯一索引，不允许有空值
* 普通索引或者单列索引：每个索引只包含单个列，一个表可以有多个单列索引
* 多列索引(复合索引、联合索引): 复合索引指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用复合索引时遵循最左前缀集合
* 唯一索引或者非唯一索引空间索引
* 空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引的列，必须将其声明为NOT NULL，空间索引只能在存储引擎为MYISAM的表中创建


> 为什么MySQL 索引中用B+tree，不用B-tree 或者其他树，为什么不用 Hash 索引 聚簇索引/非聚簇索引，MySQL 索引底层实现，叶子结点存放的是数据还是指向数据的内存地址，使用索引需要注意的几个地方？使用索引查询一定能提高查询的性能吗？为什么?

## MySQL索引结构
> 索引（index）是在存储引擎(storage engine)层面实现的，而不是server层面

### B-Tree
> 为磁盘等外存储设备设计的一种平衡查找树

系统从磁盘读取数据到内存时是以磁盘块（block）为基本单位的，位于同一个磁盘块中的数据会被一次性读取出来，而不是需要什么取什么

InnoDB 存储引擎中有页（Page）的概念,页是其磁盘管理的最小单位。InnoDB存储引擎中默认每个页的大小为16KB，可通过参数 innodb_page_size 将页的大小设置为 4K、8K、16K，在 MySQL 中可通过如下命令查看页的大小：show variables like 'innodb_page_size';

系统一个磁盘块的存储空间往往没有这么大，因此 InnoDB每次申请磁盘空间时都会是若干地址连续磁盘块来达到页的大小16KB
* InnoDB在把磁盘数据读入到磁盘时会以页为基本单位
* 在查询数据时如果一个页中的每条数据都能有助于定位数据记录的位置，这将会减少磁盘I/O次数，提高查询效率。
* B-Tree结构的数据可以让系统高效的找到数据所在的磁盘块

B-tree的node
* 一个二元组[key, data]
* key为记录的键值，对应表中的主键值
* data为一行记录中除主键外的数据。对于不同的记录,key值互不相同
* 每个节点占用一个盘块的磁盘空间
* 一个节点上有两个升序排序的关键字和三个指向子树根节点的指针
* 指针存储的是子节点所在磁盘块的地址

m阶的B-Tree的特性
* 每个节点最多有m个孩子
* 除了根节点和叶子节点外，其它每个节点至少有Ceil(m/2)个孩子
* 若根节点不是叶子节点，则至少有2个孩子
* 所有叶子节点都在同一层，且不包含其它关键字信息
* 每个非终端节点包含n个关键字信息（P0,P1,…Pn, k1,…kn）
* 关键字的个数n满足：ceil(m/2)-1 <= n <= m-1
* ki(i=1,…n)为关键字，且关键字升序排序
* Pi(i=1,…n)为指向子树根节点的指针。P(i-1)指向的子树的所有节点关键字均小于ki，但都大于k(i-1)

### B+Tree索引
* MyISAM 和 InnoDB 存储引擎，都使用 B+Tree的数据结构
* 它相对与 B-Tree结构，所有的数据都存放在叶子节点上，且把叶子节点通过指针连接到一起，形成了一条数据链表，以加快相邻数据的检索效率。


## 设计索引的原则
* 搜索的索引列,不一定是所要选择的列. 换句话说, 最适合索引的列是出现在WHERE子句中的列或连接子句中指定的列,而不是出现在 SELECT 关键字后的选择列表中的列
* 使用惟一索引。考虑某列中值的分布。索引的列的基数越大，索引的效果越好.例如生日和性别
* 使用短索引.如果对字符串列进行索引,应该指定一个前缀长度,只要有可能就应该这样做
 * 如果有一个 CHAR(200)列，如果在前 10 个或 20 个字符内，多数值是惟一的，那么就不要对整个列进行索引。
 * 较小的索引涉及的磁盘 IO 较少，较短的值比较起来更快
 * 对于较短的键值,,索引高速缓存中的块能容纳更多的键值,因此,MySQL 也可以在内存中容纳更多的值。这样就增加了找到行而不用读取索引中较多块的可能性
* 利用最左前缀.在创建一个n列的索引时,实际是创建了MySQL可利用的n个索引.多列索引可起几个索引的作用,因为可利用索引中最左边的列集来匹配行.这样的列集称为最左前缀。
* 不要过度索引。不要以为索引“越多越好”，什么东西都用索引是错误的。
  * 每个额外的索引都要占用额外的磁盘空间,并降低写操作的性能
  * 在修改表的内容时，索引必须进行更新，有时可能需要重构，因此，索引越多，所花的时间越长
  * 如果有一个索引很少利用或从不使用，那么会不必要地减缓表的修改速度
* 对于 InnoDB 存储引擎的表，记录默认会按照一定的顺序保存如果有明确定义的主键，则按照主键顺序保存.如果没有主键，但是有唯一索引，那么就是按照唯一索引的顺序保存
  *  InnoDB表的普通索引都会保存主键的键值,所以主键要尽可能选择较短的数据类型,可以有效地减少索引的磁盘占用，提高索引的缓存效果。


