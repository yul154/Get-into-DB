# 索引
> 索引（Index）是帮助MySQL高效获取数据的数据结构

索引的本质是：数据结构
* 索引的目的在于提高查询效率,可以简单的理解为“排好序的快速查找数据结构”, 这样就可以在这些数据结构上实现高级查找算法。
* 数据本身之外，数据库还维护者一个满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据
* 索引本身也很大，不可能全部存储在内存中，一般以索引文件的形式存储在磁盘
* 它们包含着对数据表里所有记录的引用指针
* 使用索引用于快速找出在某个或多个列中有一特定值的行
* 所有MySQL列类型都可以被索引


平常说的索引,没有特别指明的话,就是B+树（多路搜索树，不一定是二叉树）结构组织的索引. 其中聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引

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
* 插入和更新操作会更改索引，因此会影响数据库插入和更新的性能，并且索引会占用一定的磁盘空间，使数据库变大
* 数据库中索引是以文件的方式存储的，需要用的时候读取到内存中，因此索引的I/O操作会影响数据库的性能

## MySQL索引分类

### 数据结构角度
* B+树索引
* Hash索引
* Full-Text全文索引
* R-Tree索引

### 从物理存储角度
* 聚簇索引(clustered index):每个叶子节点存储了一行完整的表数据，叶子节点间按id列递增连接，可以方便地进行顺序检索
* 非聚集索引（non-clustered index):节点的结构完全一致只是存储的内容不同而已,将数据存储于索引分开结构，叶节点的data域存放的是数据记录的地址
* 聚集索引和非聚集索引都是B+树结构
* InnoDB 主键使用的是聚簇索引,MyISAM 不管是主键索引,还是二级索引使用的都是非聚簇索引

聚簇索引的优缺点
* 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快
* 聚簇索引对于主键的排序查找和范围查找速度非常快
* 插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式，否则将会出现页分裂
* 更新主键的代价很高，因为将会导致被更新的行移动。

回表查询
> 由于二级索引的叶子节点不存储完整的表数据，索引当通过二级索引查询到聚簇索引列值后，还需要回到聚簇索引也就是表数据本身进一步获取数据。

1. 先通过普通索引定位到主键值
2. 在通过聚集索引定位到行记录；


为什么主键通常建议使用自增id
* 聚簇索引的数据的物理存放顺序与索引顺序是一致的，即：只要索引是相邻的，那么对应的数据一定也是相邻地存放在磁盘上的


查询覆盖
> 何为索引覆盖，就是在用这个索引查询时，使它的索引树，查询到的叶子节点上的数据可以覆盖到你查询的所有字段，这样就可以避免回表

### 从逻辑角度
* 主键索引(Primary Key):InnoDB存储引擎的表会存在主键(唯一非null)如果建表的时候没有指定主键,则会使用第一非空的唯一索引作为聚集索引,否则InnoDB会自动帮你创建一个不可见的长度为6字节的row_id用来作为聚集索引。
* 单列索引：单列索引即一个索引只包含单个列
* 组合索引(composite indexes)：组合索引指在表的多个字段组合上创建的索引，只有在查询条件中使用了这些字段的左边字段时，索引才会被使用。使用组合索引时遵循最左前缀集合
* Unique(唯一索引)：索引列的值必须唯一，但允许有空值。若是组合索引，则列值的组合必须唯一。主键索引是一种特殊的唯一索引，不允许有空值
* Key（普通索引）：是MySQL中的基本索引类型，允许在定义索引的列中插入重复值和空值
* FULLTEXT(全文索引):全文索引类型为FULLTEXT，在定义索引的列上支持值的全文查找，允许在这些索引列中插入重复值和空值。全文索引可以在CHAR、VARCHAR或者TEXT类型的列上创建
* 空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引的列，必须将其声明为NOT NULL，空间索引只能在存储引擎为MYISAM的表中创建


最左前缀原则
* 如果查询的时候查询条件精确匹配索引的左边连续一列或几列，则此列就可以被用到
* 在检索数据时从联合索引的最左边开始匹配
* 由于最左前缀原则，在创建联合索引时，索引字段的顺序需要考虑字段值去重之后的个数，较多的放前面。ORDER BY子句也遵循此规则


为什么要使用联合索引
* 减少开销.建一个联合索引 (col1,col2,col3)，实际相当于建了 (col1)，(col1,col2)，(col1,col2,col3) 三个索引
* 覆盖索引.对联合索引 (col1,col2,col3)，如果有如下的 SQL：select col1,col2,col3 from test where col1=1 and col2=2;。那么 MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机 IO 操作
* 效率高.索引列越多，通过索引筛选出的数据越少


### 索引建立原则
1. 索引并非越多越好，一个表中如果有大量的索引，不仅占用磁盘空间，而且会影响INSERT、DELETE、UPDATE等语句的性能，因为在表中的数据更改的同时，索引也会进行调整和更新
2. 避免对经常更新的表进行过多的索引，并且索引中的列尽可能少。而对经常用于查询的字段应该创建索引，但要避免添加不必要的字段
3. 数据量小的表最好不要使用索引
4. 在条件表达式中经常用到的不同值较多的列上建立索引，在不同值很少的列上不要建立索引
5. 当唯一性是某种数据本身的特征时，指定唯一索引
6. 在频繁进行排序或分组（即进行group by或order by操作）的列上建立索
7. 利用最左前缀。在创建一个n列的索引时，实际是创建了MySQL可利用的n个索引


## MySQL索引结构
> 索引（index）是在存储引擎(storage engine)层面实现的，而不是server层面

系统从磁盘读取数据到内存时是以磁盘块（block）为基本单位的,位于同一个磁盘块中的数据会被一次性读取出来,而不是需要什么取什么

MySQL中的数据存储通常以Page为单位,俗称数据页
* 每个Page对应B+Tree的一个节点
* 页是InnoDB磁盘管理的最小单位,默认每个数据页的大小为16kb，也可以通过参数innodb_page_size将页的大小设置成其他值。
* 默认情况下一个Page的大小为16kb，由于每个Page中数据通过指针相连，且每个指针大小为6字节

```
我们通常使用长度为8个字节的bigint类型作为主键id的类型。
假设每条记录长度为1kb（包含指针）；
已知，每一条数据都会包含一个6字节的指针
所以一条索引数据大约占用8+6=14个字节,一个Page中能存储16 * 1024 / 14 ≈ 1170条索引数据
```

系统一个磁盘块的存储空间往往没有这么大,因此,InnoDB每次申请磁盘空间时都会是若干地址连续磁盘块来达到页的大小16KB
* InnoDB在把磁盘数据读入到磁盘时会以页为基本单位


### B-Tree
> 为磁盘等外存储设备设计的一种平衡查找树

B-Tree 借助计算机磁盘预读机制;每次新建节点的时候,都是申请一个页的空间,所以每查找一个节点只需要一次 I/O,因为实际应用当中，节点深度会很少，所以查找效率很高。

* 在查询数据时如果一个页中的每条数据都能有助于定位数据记录的位置，这将会减少磁盘I/O次数，提高查询效率。
* B-Tree结构的数据可以让系统高效的找到数据所在的磁盘块

B-tree的node
* 一个二元组[key, data]
* key为记录的键值，对应表中的主键值
* data为一行记录中除主键外的数据.对于不同的记录,key值互不相同
* 每个节点占用一个盘块的磁盘空间
* 一个节点上有两个升序排序的关键字和三个指向子树根节点的指针
* 指针存储的是子节点所在磁盘块的地址

m阶的B-Tree的特性
* 根结点至少有两个子女
* 每个中间节点都包含k-1个元素和k个孩子，其中 m/2 <= k <= m 
* 每一个叶子节点都包含k-1个元素，其中 m/2 <= k <= m
* 所有的叶子结点都位于同一层
* 每个节点中的元素从小到大排列，节点当中k-1个元素正好是k个孩子包含的元素的值域分划
* ki(i=1,…n)为关键字，且关键字升序排序
* Pi(i=1,…n)为指向子树根节点的指针。P(i-1)指向的子树的所有节点关键字均小于ki，但都大于k(i-1)

### B+Tree索引

数据库中B+Tree的高度一般在2-4之间，即查找某一键值的行记录时最多进行2-4次IO

一个m阶的B+树具有如下几个特征：
* 有k个子树的中间节点包含有k个元素（B树中是k-1个元素）
* 所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接
* 所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素

B+Tree索引可以分为聚集索引(clustered index)和辅助索引(secondary index)
* 聚集索引的B+Tree中的叶子节点存放的是整张表的行记录数据
* 辅助索引的叶子节点并不包含行记录的全部数据,而是存储相应行数据的聚集索引键,即主键.当通过辅助索引来查询数据时.InnoDB会遍历辅助索引找到主键,然后再通过主键在聚集索引中找到完整的行记录数据

B+树的优势：

* 单一节点存储更多的元素，使得查询的IO次数更少。
* 所有查询都要查找到叶子节点，查询性能稳定。
* 所有叶子节点形成有序链表，便于范围查询。

### 存储引擎与索引

MyISAM 和 InnoDB 存储引擎，都使用 B+Tree的数据结构
* 它相对与B-Tree结构，非叶子节点只存储键值信息, 所有的数据都存放在叶子节点上,所有叶子节点之间都有一个链指针.且把叶子节点通过指针连接到一起，形成了一条数据链表，以加快相邻数据的检索效率。
* 对B+Tree进行两种查找运算: 一种是对于主键的范围查找和分页查找,另一种是从根节点开始，进行随机查找.因为在B+Tree上有两个头指针,一个指向根节点,另一个指向关键字最小的叶子节点，所有叶子节点（即数据节点）之间是一种链式环结构
* MySQL的InnoDB存储引擎在设计时是将根节点常驻内存的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作

MyISAM索引结构
* MyISAM存储引擎中数据文件和索引文件是分离的。
* MyISAM的索引主要分为主索引和辅助索引:MyISAM索引方式也称为“非聚集”索引
* 主索引（primary key）：以主键做为索引，因此key是唯一的
* 辅助索引（secondary key）：辅助索引以非主键做为索引，因此key是可以重复的


InnoDB索引结构
* nnoDB存储引擎也分为主索引和辅助索引：InnoDB索引方式也称为聚集索引
* InnoDB数据文件本身就是索引文件.因为InnoDB的数据文件本身是按主键聚集的,所以InnoDB要求表必须有主键,如果没有显示指定主键,InnoDB会自动选择可以唯一标识数据记录的列做主键;如果不存在,则表生成一个隐含字段作为主键


> MyISAM 存储引擎和 InnoDB 的索引有什么区别
* InnoDB是聚集索引，数据文件是和（主键）索引绑在一起的,通过主键索引到整个记录， 辅助索引先查询到主键，通过主键索引到整个记然
* MyISAM是非聚集索引，也是使用B+Tree作为索引结构，索引和数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的
* InnoDB的B+树主键索引的叶子节点就是数据文件，辅助索引的叶子节点是主键的值；
* 而MyISAM的B+树主键索引和辅助索引的叶子节点都是数据文件的地址指针。

> 聚簇索引和非聚簇索引区别
* 聚簇索引的叶子节点就是数据节点
* 而非聚簇索引的叶子节点仍然是索引节点，只不过有指向对应数据块的指针。

> 为什么MySQL 索引中用B+tree，不用B-tree 或者其他树，

* 树的查询时间跟树的高度有关，B+树是一棵多路搜索树可以降低树的高度，提高查找效率
* MySQL并未使用红黑树作为索引的实现，主要原因在于红黑树只有两个子树,深度过大，数据检索时造成磁盘IO频繁
* B-Tree是一种自平衡的多叉搜索树，一个节点可以拥有两个以上的子节点,MySQL索引一般都存储在内存中,如果使用B-Tree作为索引的话,索引和数据存储在一块,一定内存的情况下可以存储的索引数量相对有限，毕竟每条数据的大小一般远大于索引列的大小，导致内存使用率不高.
* 而B-Tree和红黑树对于顺序查询并不友好，B+Tree的数据页只存储在叶子节点中，并且叶子节点之间通过指针相连，为双向链表结构,能够很好支持单值，范围查询，有序性查询
* B+树更适合外部存储(一般指磁盘存储),由于内节点(非叶子节点)不存储data,所以一个节点可以存储更多的内节点,每个节点能索引的范围更大更精确.也就是说使用B+树单次磁盘I/O的信息量相比较B树更大,I/O 效率更高。
* MySQL会按照区间来访问某个索引列,B+树的叶子节点间按顺序建立了链指针,加强了区间访问性,所以 B+树对索引列上的区间范围查询很友好.而B树每个节点的key和data在一起,无法进行区间查找


> 既然增加树的路数可以降低树的高度，那么无限增加树的路数是不是可以有最优的查找效率？

* 这样会形成一个有序数组，文件系统和数据库的索引都是存在硬盘上的,并且如果数据量大的话,不一定能一次性加载到内存中。
* 这时候B+树的多路存储，可以每次加载B+树的一个结点，然后一步步往下找

>  为什么不用Hash索引，聚簇索引/非聚簇索引，

* 哈希结构在单条数据的等值查询是性能非常优秀，但是只能用来搜索等值的查询
* Hash索引通常都是随机的内存访问，对于缓存不友好
* Hash 索引无法被用来避免数据的排序操作
* Hash 索引遇到大量Hash值相等的情况后性能并不一定就会比B+Tree索引高
 

> MySQL主键的设计原则
* 唯一标识一条记录，不能有重复的，不允许为空
* 用来保证数据完整性
* MySQL主键应该是单列的，以便提高连接和筛选操作的效率
* MySQL主键应当是对用户没有意义的。


> MySQL 索引底层实现，叶子结点存放的是数据还是指向数据的内存地址，

> 使用索引需要注意的几个地方？使用索引查询一定能提高查询的性能吗？为什么?


## 设计索引的原则
* 在经常需要搜索的列上，可以加快搜索的速度.搜索的索引列,不一定是所要选择的列. 换句话说, 最适合索引的列是出现在WHERE子句中的列或连接子句中指定的列,而不是出现在 SELECT 关键字后的选择列表中的列
* 使用惟一索引。考虑某列中值的分布。索引的列的基数越大，索引的效果越好.例如生日和性别
* 使用短索引.如果对字符串列进行索引,应该指定一个前缀长度,过长的字段会使索引的数据结构变大，占用更多的空间，因此影响数据库的性能
 * 如果有一个 CHAR(200)列，如果在前 10 个或 20 个字符内，多数值是惟一的，那么就不要对整个列进行索引。
 * 较小的索引涉及的磁盘 IO 较少，较短的值比较起来更快
 * 对于较短的键值,,索引高速缓存中的块能容纳更多的键值,因此,MySQL 也可以在内存中容纳更多的值。这样就增加了找到行而不用读取索引中较多块的可能性
* 利用最左前缀.在创建一个n列的索引时,实际是创建了MySQL可利用的n个索引.多列索引可起几个索引的作用,因为可利用索引中最左边的列集来匹配行.这样的列集称为最左前缀。
* 不要过度索引。不要以为索引“越多越好”，什么东西都用索引是错误的。
  * 每个额外的索引都要占用额外的磁盘空间,并降低写操作的性能
  * 在修改表的内容时，索引必须进行更新，有时可能需要重构，因此，索引越多，所花的时间越长
  * 如果有一个索引很少利用或从不使用，那么会不必要地减缓表的修改速度
* 在数据库中尽量使用单调增的字段做为主键，因为索引本身是一个B+树，如果字段是非单调增的，则插入操作会频繁的分裂B+树来调增数据的有序性和平衡性
* 对于 InnoDB 存储引擎的表，记录默认会按照一定的顺序保存如果有明确定义的主键，则按照主键顺序保存.如果没有主键，但是有唯一索引，那么就是按照唯一索引的顺序保存
  *  InnoDB表的普通索引都会保存主键的键值,所以主键要尽可能选择较短的数据类型,可以有效地减少索引的磁盘占用，提高索引的缓存效果。
