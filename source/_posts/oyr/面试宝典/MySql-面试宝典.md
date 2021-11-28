---
title: Mysql-面试宝典
date: 2021-11-25 00:00:00
author: 神奇的荣荣
summary: ""
categories: 面试宝典
tags: 
    - Mysql
    - 面试宝典
---

# 数据库基础

## 主流的数据库

SqlServer数据库
是微软，.net程序员最爱，中型和大型项目，性能高

Oracle数据库
是甲骨文的，java程序员（必学），大型项目，特点是适合处理复杂业务逻辑。

Mysql数据库
是sun公司，属于甲骨文公司，中型大型项目，特点：并发性好。对简单的sql处理效率高。

db2数据库
是ibm公司，处理海量数据，大型项目。很强悍。

Informix数据库
是ibm公司。在银行系统，安全性高

Sybase数据库

<!-- more -->

## 数据库中DQL，DML，DDL，TCL语言都是什么？

DQL语言：查询语句
DML语言：新增，修改，删除语句
DDL语言：库，表，约束的结构管理
TCL语言：事物管理

# MySql 基础

## crud语句

select * from 表名
insert into 表名 values(),(),......
update 表名 set 列名=？ where 列名=？
delete from 表名 where 列名=?

## sql怎么去重?

使用distinct去重

## case的使用

case
作用：判断
语法：
```sql
case 要带的字段或表达式
when 常量1 then 要显示的值或语句;
when 常量2 then 要显示的值或语句;
....
else 要显示的值或语句
end
```

```sql
/*
案例：查询员工的工资，要求
部门号=30,显示的工资为1.1倍
部门号=40，显示的工资为1.2倍
部门号=50，显示的工资为1.3倍
其他部门，显示的工资为原工资
*/
SELECT last_name, salary 原工资, 
CASE department_id
WHEN 30 THEN salary*1.1
WHEN 40 THEN salary*1.2
WHEN 50 THEN salary*1.3
ELSE salary
END AS 新工资
FROM employees;

/*
案例：查询员工的工资的情况
如果工资>20000，显示A级别
如果工资>15000，显示B级别
如果工资>10000,显示C级别
否则，显示D级别
*/
SELECT last_name,salary,
CASE
WHEN salary>20000 THEN 'A'
WHEN salary>15000 THEN 'B'
WHEN salary>10000 THEN 'C'
ELSE 'D'
END AS 级别
FROM employees;
```

## cahr 和 varchar 的区别？

char：是一个定长字段，假如申请了char(10)的空间，那么无论实际存储多少内容，该字段都占用10个字符
varchar：varchar是变长字段，也就是说申请的只是最大长度，占用的空间为实际字符+1，最后一位字符存储了使用多少空间。

在检索效率上来讲，char > varchar，因此在使用中,如果确定某个字段的值的长度，可以使用char，否则应该尽量使用varchar。例如存储用户MD5加密后的密码，则应该使用char。

## union 与 union all有什么区别？

- union：合并去重返回结果集。
- union all：合并不去重返回结果集。

## in 与 exists 的区别

- in：作用是判断是否匹配列表中的值，返回true或false。适合于外表大而内表小的情况；
- exists：作用是指定一个子查询，返回true或false。适合于外表小而内表大的情况。

## 外连接、内连接与自连接的区别？

- 内连接：关键字是inner join，功能是把匹配的关联数据显示出来，两边的表数据都会显示。
- 左连接：关键字是left join，功能是左边的表全部显示出来，右边的表显示出符合条件的数据。
- 右连接：关键字是right join，功能是右边的表全部显示出来，左边的表显示出符合条件的数据。

## drop、delete、truncate 的区别？

- drop：删除关于表的一切（数据，结构，约束），不会记录日志，是DDL操作。
- delete：删除表数据，delete可以加where条件，delete删除有返回值，delete删除可以回滚。使用delete删除后，再插入数据，自增长列的值从断点开始。
- truncate：删除表所有数据，truncate不可以加where条件，truncate删除没有返回值，truncate删除不能回滚。使用truncate删除后，再插入值，自增长列的值从1开始。truncate删除效率比delete高。

## 查询sql的写法与机读顺序

查询sql写法：
```xml
Select Distinct
<Select_List>
From 
<left_table>	<join_type> 
Join <right_table> on <join_condition>
WHERE	
<where_condition>
GROUP BY 
<group_by_list>
HAVING
<having_condition>
ORDER BY
<order_by_condition>
LIMIT <limit_number>
```

机读顺序：
```xml
FROM <left_table>
ON <join_condition>
<join_type> JOIN <right_table>
WHERE <where_condition>
GROUP BY <group_by_list>
HAVING <having_condition>
Select 
Distinct <Select_List>
ORDER BY <order_by_condition>
LIMIT <limit_number>
```

机读顺序，如图所示：
![查询SQL机读顺序](https://rong0624.gitee.io/images/MySql/20211126131122.png)

# MySql 事务

## 事务是什么？

事务是一个不可分割的数据库操作序列，也是数据库并发控制的基本单位，其执行的结果必须使数据库从一种一致性状态变到另一种一致性状态。事务是逻辑上的一组操作，要么都执行，要么都不执行。

## MySql支持事务吗？

mysql是否支持事务要根据存储引擎决定。InnoDB支持事务，MyISAM不支持事务。

## ACID是什么？

ACID指的是原子性（Atomicity），一致性（Consistency）， 隔离性（Isolation），持久性（Durability）

- 原子性（Atomicity）：指事务是一个不可分割的工作单位，事务中的操作要么都执行成功，要么都执行失败。
- 一致性（Consistency）：事务必须使数据库从一个一致性状态变换到另外一个一致性状态。就拿转账为例，A有500元，B有300元，如果在一个事务里A成功转给B50元，那么不管并发多少，不管发生什么，只要事务执行成功了，那么最后A账户一定是450元，B账户一定是350元。不管转账多少次两个人的钱加起来都是800元。
- 隔离性（Isolation）：并发执行的各个事务之间不能互相干扰。事务的隔离性是指一个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对并发的其他事务是隔离的。
- 持久性（Durability）：持久性是指一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来的其他操作和数据库故障不应该对其有任何影响。

## 事务并发会带哪些问题？

脏读，不可重复读，幻读。

对于同时运行的多个事务，当这些事务访问数据库中相同的数据时，如果没有采取必要的隔离机制，就会导致各种并发问题:
- 脏读：一个事务读取到了其他事务未提交的数据；对于两个事务 T1, T2, T1 读取了已经被 T2 更新但还没有被提交的字段。之后, 若 T2 回滚, T1读取的内容就是临时且无效的。
- 不可重复读：一个事务中多次读取同一个字段，值可能不同；对于两个事务T1, T2, T1 读取了一个字段, 然后 T2 更新了该字段。之后, T1再次读取同一个字段, 值就不同了。
- 幻读：一个事务中多次读取一张表数据，行可能不同；对于两个事务T1, T2, T1 从一个表中读取了一个字段, 然后 T2 在该表中插入了一些新的行。之后, 如果 T1 再次读取同一个表, 就会多出几行。

## 事物的隔离级别有哪些？

![事务隔离级别](https://rong0624.gitee.io/images/MySql/20211125190201.png)

Mysql支持4种事务隔离级别。Mysql 默认的事务隔离级别为: REPEATABLE READ。

# MySql 架构

## MySql 总体架构

![总体架构](https://rong0624.gitee.io/images/MySql/20211126124332.png)

整个MySQL Server由以下组成
Connection Pool : 连接池组件
Management Services & Utilities : 管理服务和工具组件
SQL Interface : SQL接口组件
Parser : 查询分析器组件
Optimizer : 优化器组件
Caches & Buffers : 缓冲池组件
Pluggable Storage Engines : 存储引擎
File System : 文件系统

主要分为：连接层、服务层、引擎层、存储层。
也可以认为是：服务层（连接层和服务层），存储层（引擎层和存储层）

### 连接层

最上层是一些客户端和连接服务，包含本地sock通信和大多数基于客户端/服务端工具实现的类似于tcp/ip的通信。主要完成一些类似于连接处理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

### 服务层

第二层架构主要完成大多数的核心服务功能：
- SQL Interface: SQL接口；接受用户的SQL命令，并且返回用户需要查询的结果。比如select from就是调用SQL Interface
- Parser: 解析器；SQL命令传递到解析器的时候会被解析器验证和解析。 
- Optimizer: 查询优化器；SQL语句在查询之前会使用查询优化器对查询进行优化。 
- Cache和Buffer：缓存和缓冲；如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。这个缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，key缓存，权限缓存等

### 引擎层

存储引擎层，存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。

### 存储层

数据存储层，主要是将数据存储在运行于裸设备的文件系统之上，并完成与存储引擎的交互。

## Mysql支持哪些存储引擎？

MySql在5.0版本，支持的存储引擎有：InnoDB 、MyISAM 、BDB、MEMORY、MERGE、EXAMPLE、NDB Cluster、
ARCHIVE、CSV、BLACKHOLE、FEDERATED等。其中InnoDB和BDB提供事务安全表，其他存储引擎是非事务安
全表。

其中常用的存储引擎是InnoDB和MyISAM。

## 引擎 InnoDB 和 MyISAM 的区别？

MyISAM：不支持主外键和事物，支持表级锁，缓存索引但不缓存数据。

InnoDB：是Mysql5.1后默认的存储引擎，支持主外键和事务，支持表级锁和行级锁，默认采用行级锁，缓存索引也缓存数据。

## 数据文件

MyISAM：
- .frm（Framework）：存放表结构
- .myd（data）：存放表数据
- .myi（index）：存放表索引

InnoDB：
- frm文件：存放表结构
- 数据文件：
    - 使用共享表空间存储，默认情况下使用，数据和索引保存在同一个表空间中，可以是多文件，通过innodb_data_home_dir 和 innodb_data_file_path定义的表空间。
    - 使用多表空间存储，每个表的数据和索引单独保存在.ibd文件中。

## 日志文件

Mysql有7种日志文件，分别是：  
- errorlog（错误日志）  
- generallog（普通日志）  
- slow query log（慢查询日志）  
- binlog（二进制日志）  
- relaylog（中继日志）  
- redolog（重做日志）  
- undolog（回滚日志）

总要的日志文件：
- slow query log：慢查询日志  
- undolog-redolog：事务日志（innoDB存储引擎日志）  
- binlog：二进制日志（server层日志）  
- relaylog：中继日志（主从复制）

## 一条查询SQL语句在执行时，其经历了那些过程？

会经历服务层和存储层
服务层：
连接器（连接mysql服务），
查询缓存（查询sql缓存），
分析器（词法分析，语法分析），
优化器（优化查询sql，执行计划生成，索引选择），
执行器（操作存储引擎，返回结果）

存储层：
存储引擎：存储数据，提供读写接口

# MySql 索引

## 索引是什么？

索引（index）是帮助MySQL高效获取数据的数据结构（有序）。
索引的目的：提高查找效率，类比字典。
索引的本质：索引是数据结构（有序）。

## 索引的优势和劣势

优势：
- 类似于书籍的目录索引，提高数据检索的效率，降低数据库的IO成本。
- 通过索引列对数据进行排序，降低数据排序的成本，降低了CPU的消耗。

劣势：
- 实际上索引也是一张表，每个索引也需要占用物理空间
- 当对表数据进行INSERT、UPDATE和DELETE时，也需要动态维护索引，降低了数据的更新速度。

## 索引都运用在哪些地方？

- where：查询字段建立索引，通过索引进行检索
- join：表连接中，关联字段建立索引，提高关联效率
- 索引覆盖：如果要查询的字段都建立过索引，那么引擎会直接在索引表中查询而不会访问原始数据（只要有一个字段没有建立索引就会做全表扫描），这叫索引覆盖。因此我们需要尽可能的在select后只写必要的查询字段，以增加索引覆盖的几率。
- order by：排序字段建立索引，通过索引进行排序

## MySql索引类型有哪些？

常用的索引类型：
- 单值索引：即一个索引只包含单个列，一个表可以有多个单值索引
- 复合索引：即一个索引包含多个列
- 唯一索引：索引列的值必须唯一，但允许有空值
- 主键索引：索引列的值必须唯一，且不能为空

不常用的索引类型：
- 前缀索引
- 聚簇索引

## MySql索引结构有哪些？

MySql索引结构主要分为4种，分别是：B-Tree索引、Hash索引、R-Tree索引、Full-Text索引（全文索引）。

InnoDB引擎支持B-Tree索引、Full-Text索引（全文索引）。
MyISAM引擎支持B-Tree索引、R-Tree索引、Full-Text索引（全文索引）

我们平常所说的索引，如果没有特别指明，都是指B+树（多路搜索树，并不一定是二叉的）结构组织的索引。其中
聚集索引、复合索引、前缀索引、唯一索引默认都是使用 B+tree 索引，统称为索引。

## B+Tree索引原理

// TODO

## 什么情况下需要创建索引，什么情况下不用创建索引？

哪些情况下需要创建索引：
- 频繁作为查询条件的字段创建索引
- 在关联表查询中，外键关联字段创建索引
- 查询中排序的字段创建索引，通过索引排序提高效率
- 查询中统计或分组的字段

哪些情况下不要创建索引：
- 表数据量少（mysql查询也是很强的，在数据量大的时候去建索引，百万条数据后查询性能逐渐下降）
- 更新频繁字段不适合创建索引（提高了查询速度，同时却会降低更新表的速度，MySQL不仅要保存数据，还要维护一下索引文件）
- 对于那些查询中很少涉及的列，重复值比较多的列不要创建索引

## 什么是最左前缀原则？什么是最左匹配原则？

如果是复合索引（由多字段组成），要遵守最左前缀法则。指的是查询从索引的最左列开始并且不跳过索引中的列。所以在创建复合索引时，要根据业务需求，where中使用最频繁的一列放在最左边，用于命中索引。

```sql
创建复合索引: 
CREATE INDEX idx_name_email_status ON tb_seller(NAME,email,STATUS); 

就相当于对：
name 创建索引 ;
对name , email 创建了索引; 
对name , email, status 创建了索引;
```

## 什么是前缀索引？

前缀索引：```index(field(10))```，使用字段值的前10个字符建立索引，默认索引是使用字段的全部内容。适用于前缀的标识度高，比如密码就适合建立前缀索引，因为密码几乎各不相同。

## 什么是聚簇索引？何时使用聚簇索引与非聚簇索引？

- 聚簇索引：将数据与索引存储到了一起，找到索引也就找到了数据
- 非聚簇索引：将数据存储于索引分开结构，索引结构的叶子节点指向了数据的对应行，myisam通过key_buffer把索引先缓存到内存中，当需要访问数据时（通过索引访问数据），在内存中直接搜索索引，然后通过索引找到磁盘相应数据，这也就是为什么索引不在key buffer命中时，速度慢的原因

澄清一个概念：innodb中，在聚簇索引之上创建的索引称之为辅助索引，辅助索引访问数据总是需要二次查找，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引，辅助索引叶子节点存储的不再是行的物理位置，而是主键值

何时使用聚簇索引与非聚簇索引：
![何时使用聚簇索引与非聚簇索引](https://rong0624.gitee.io/images/MySql/20211126231116.png)

## 非聚簇索引一定会回表查询吗？

不一定，如果索引覆盖了，字段全部命中索引，就不需要再进行回表查询了。

> 举个简单的例子，假设我们在员工表的年龄上建立了索引，那么当进行select age from employee where age < 20的查询时，在索引的叶子节点上，已经包含了age信息，不会再次进行回表查询。

## 索引如何查看、创建、删除？

查看索引：
```sql
SHOW INDEX FROM tableName
```

创建索引：
```sql
-- 第一种方式
CREATE [UNIQUE] INDEX IndexName ON tableName(columnName(length));
-- 第二种方式
ALTER TableName add [unique] INDEX [IndexName] ON (columnName(length));
-- 添加普通索引，索引值可出现多次。
ALTER TABLE tbl_name ADD INDEX index_name (column_list): 
-- 该条语句创建索引的值必须是唯一的（除了NULL外，NULL可能会出现多次）。
ALTER TABLE tbl_name ADD UNIQUE index_name (column_list): 
-- 该语句添加主键索引,这意味着索引值必须是唯一的，且不能为NULL。
ALTER TABLE tab_name ADD PRIMARY KEY(column_list); 
-- 该语句指定了索引为 FULLTEXT ，用于全文索引。
ALTER TABLE tbl_name ADD FULLTEXT index_name (column_list):
```

删除索引：
```sql
DORP INDEX IndexName ON TableName;
```

# Sql优化

## 慢查询日志

## explain

explanin关键字可以模拟服务层sql优化器执行sql查询，从而知到mysql如何执行sql查询语句的。可以分析sql的性能瓶颈。

explain 关键字分析sql查询的执行情况，能明确看到当前sql查询类型，是否使用索引，使用的索引名，索引精度，还有其他的关键信息（使用了覆盖索引，使用了文件内排序，使用了临时表等等）。

explain使用语法：
```sql
-- explain sql
explain select * from user;
```

explain分析sql，相关属性有：id、select_type、table、type、possible_keys、key、key_len、ref、rows、extra。

### id

id：select查询的序号，由数字组成。表示一个查询中各个子查询的执行顺序。
- id相同，执行顺序由上至下。
- id不同，id值越大优先级越高，越先被执行。（如果是子查询，id的序号会递增）
- id为null时表示一个结果集，不需要使用它查询，常出现在包含union等查询语句中

### select_type

select_type：表示每个查询的类型。主要用于区分普通查询、联合查询、子查询等的复杂查询。
- SIMPLE：简单的 select 查询，查询中不包含任何子查询或者UNION等查询
- PRIMARY：查询中若包含任何复杂的子查询，最外层查询则被标记为Primary
- SUBQUERY：在 select 或 where 中包含的子查询
- DERIVED：在FROM列表中包含的子查询被标记为DERIVED(衍生)，MySQL会递归执行这些子查询, 把结果放在临时表里。
- DEPENDENT SUBQUERY：在SELECT或WHERE列表中包含了子查询,子查询基于外层
- UNCACHEABLE SUBQUREY：无法被缓存的子查询
- UNION：若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM子句的子查询中,外层SELECT将被标记为：DERIVED
- UNION RESULT：从UNION表获取结果的SELECT

### table

table：表示当前id查询的数据是那张表的。

### type（非常重要）

table：表示查询的类型，最常见的最好到最差依次是：system>const>eq_ref>ref>range>index>ALL。
- system：表只有一行记录（等于系统表），这是const类型的特列，平时不会出现，这个也可以忽略不计。
- const：表示通过索引一次就找到了,const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快，如将主键置于where列表中，MySQL就能将该查询转换为一个常量。
- eq_ref：唯一性索引查找，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描。
- ref：非唯一性索引查找，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而，它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体
- range：索引范围查找，只检索给定范围的行,使用一个索引来选择行。key 列显示使用了哪个索引一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围索引扫描比全表扫描要好，因为它只需要开始于索引的某一点，而结束语另一点，不用扫描全部索引。
- index：Full Index Scan，遍历索引，index与all区别为index类型只遍历索引树。这通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和Index都是读全表，但index是从索引中读取的，而all是从表所有数据中读的）
- all：Full Table Scan，扫描全表数据，找到匹配的行。

### possible_keys

possible_keys：显示可能使用到的索引，注意不一定会使用。显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。

### key

key：表示实际使用的索引，如果为null，则没有使用索引。查询中若使用了覆盖索引，则该索引就出现在key列表中。

### key_len

key_len：表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。

### ref

ref：显示索引的哪一列被使用了，如果可能的话，是一个常量。显示哪些列或常量被用于查找索引列上的值。

### rows

rows：根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数。越少越好。

### extra

extra：包含不适合在其他列中显示，但十分重要的额外信息。
- USING index：使用覆盖索引，避免访问了表的数据行，效率不错！如果同时出现using where，表明索引被用来执行索引键值的查找；如果没有同时出现using where，表明索引只是用来读取数据而非利用索引执行查找。
- Using where：表明使用了where过滤。
- Using filesort：说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。
MySQL中无法利用索引完成的排序操作称为“文件排序”。内部再捣鼓了一次，性能下降。（九死一生，很重要，必须优化）
- Using temporary：使用了临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by。（十死无生，很重要，必须优化）

## show profile

show profile 分析力度比 explain 更细。
show profile 可以分析sql运行的生命周期，可以帮助我们了解sql执行时间都耗费到哪里去了。可以看到很多信息，列如：可以看到开启表，查询数据，使用临时表，关闭表等等步骤信息，也可以见到每个步骤的cpu和io的消耗等等。

## 如何避免索引失效？

1. 使用复合索引时，要遵守最左前缀法则。（指的是查询从索引的最左列开始并且不跳过索引中的列）
2. 不在索引列上做任何操作（计算、函数、自动或手动类型转换），会导致索引失效而转向全表扫描
3. 在同一复合索引下，范围条件右边索引失效，若是不同索引则不会失效
4. 在使用不等于(!= 或者<>)的时候会导致索引失效
5. is null 或 is not null会导致索引失效（在索引列中不要保存null，用0代替，避免判null操作）
6. like通配符%开头索引失效（可以使用覆盖索引）
7. 用or连接时索引失效（可以使用union来处理，使用各自的索引）

## or关键字优化？

建议使用 union 来替代 or ，多个查询使用各自的索引。

## is null & is not null 关键字优化？

is null 或 is not null会导致索引失效？
尽量给索引字段设置为not null，用0代替null，避免判null操作。

## order by 关键字优化？

尽量使用索引排序：
- order by语句使用索引，如果是复合索引需要满足最左前缀法则，会使用索引排序
- where子句 和 order子句条件列满足复合索引最左前缀法则，会使用索引排序
- 在使用order by时，不要用select *，只查询所需的字段，尽量使用索引内字段

如果没有办法使用索引排序就会触发文件排序，单路排序或双路排序。
文件排序（filesort）优化
- 尝试提高sort_buffer_size。
- 尝试提高max_length_for_sort_data。

## group by 关键字优化？

group by实质是先排序后进行分组，遵照索引的最左前缀法则。
尽量使用索引列分组，可以通过```order by null```禁止排序，提高效率。

## 子查询优化？

建议使用连接（join）查询代替，避免创建临时表。

## 连接（join）查询优化？

- 尽量保证小表驱动大表。
- 在被驱动表关联字段上建立索引。

## 超大分页怎么优化？

初始：
```sql
select * from table where age > 20 limit 1000000,10;
```
这种查询其实也是有可以优化的余地的. 这条语句需要加载1000000数据然后基本上全部丢弃，只取10条当然比较慢。

优化思路一：在索引上完成排序分页操作，最后根据主键关联回原表查询所需要的其他列内容
```sql
select * from table where id in (select id from table where age > 20 limit 1000000,10)
```
这样虽然也load了一百万的数据,但是由于索引覆盖,要查询的所有字段都在索引中,所以速度会很快。

优化思路二：该方案适用于主键自增的表，可以把 Limit 查询转换成某个位置的查询
```sql
select * from table where id > 1000000 limit 10
```
效率也是不错的,优化的可能性有许多种,但是核心思想都一样,就是减少加载的数据.

## sql优化思路

常见的SQL优化：
连接查询替代子查询
连接查询时保证小表驱动大表，给被驱动表join字段建索引
给条件查询字段上建索引
尽量避免索引失效
尽量在order by中使用索引（避免文件内排序，索引即可查询，也可以排序。）
尽量在group by中使用索引（避免使用临时表，禁止排序，能在where中过滤掉的数据，就不要使用having了）

测试环境分析优化sql：
开启慢查询日志（捕获慢查询sql）
分析慢查询sql（explanin+show profile）
对慢sql做优化（关键字相关优化，建立索引，避免索引失效）
mysql数据库参数调优（dba或运维经理）

大数据量优化：(数据库架构和架构方面)
读写分离，负载均衡
垂直切分和水平切分
如果是因为表多而数据多，这时候适合使用垂直切分，即把关系紧密（比如同一模块）的表切分出来放在一个server上。
如果表并不多，但每张表的数据非常多，这时候适合水平切分，即把表的数据按某种规则（比如按ID散列）切分到多个数据库(server)上。

当然，现实中更多是这两种情况混杂在一起，这时候需要根据实际情况做出选择，也可能会综合使用垂直与水平切分，从而将原有数据库切分成类似矩阵一样可以无限扩充的数据库(server)阵列。

服务器硬件优化：
砸钱。一个办法。

# MySql 锁

## 为什么要使用锁？

当事务并发情况下，可能会产生数据的不一致，这时候需要一些机制来保证数据安全问题，锁就是这样的一个机制。

## MySql 锁分类

从锁的类别上分类：
- 共享锁（读锁）：针对同一份数据，多个读操作可以同时进行而不会互相影响。
- 排它锁（写锁）：当前写操作没有完成前，它会阻断其他写锁和读锁。

## 表锁

## 行锁



## 间隙锁

# MySql 主从复制

# MySql 分库分表

# 其他

## 事物日志

// TODO

## 数据库的三大范式，范式化设计优缺点？

三大范式：
- 第一范式(1NF)：是对属性的原子性约束，要求属性具有原子性，不可再分解；
- 第二范式(2NF)：是对记录的惟一性约束，要求记录有惟一标识，即实体的惟一性；
- 第三范式(3NF)：是对字段冗余性的约束，即任何字段不能由其他字段派生出来，它要求字段没有冗余。

优点：可以尽量得减少数据冗余，使得更新快，体积小。
缺点：对于查询需要多个表进行关联，减少写操作的效率和增加读操作的效率，更难进行索引优化。