# MySQL索引

数据库索引好比一本书前面的目录，能加快数据库的查询速度。

# 创建索引

**普通索引**
这是最基本的索引，它没有任何限制，MyIASM中默认的BTREE类型的索引，也是我们大多数情况下用到的索引。
创建方式：

- **直接创建索引**

```sql
CREATE INDEX index_name ON table(column(length))
```

- **修改表结构的方式添加索引**

```sql
ALTER TABLE table_name ADD INDEX index_name ON (column(length))
```

- **创建表的时候同时创建索引**

```sql
CREATE TABLE `table` (
	`id` int(11) NOT NULL AUTO_INCREMENT ,
	`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
	`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
	`time` int(10) NULL DEFAULT NULL ,
	PRIMARY KEY (`id`),
	INDEX index_name (title(length))
)
```

- 删除索引

```sql
DROP INDEX index_name ON table
```

**2、唯一索引**
与普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须是唯一的，创建方法和普通索引类似。

- 创建唯一索引

```sql
CREATE UNIQUE INDEX index_name ON table(column(length)) 
```

- 修改表结构

```sql
ALTER TABLE table_name ADD UNIQUE INDEX index_name ON (column(length))
```

- 创建表时同时创建索引

```sql
CREATE TABLE `table` (
	`id` int(11) NOT NULL AUTO_INCREMENT ,
	`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
	`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
	`time` int(10) NULL DEFAULT NULL ,
	PRIMARY KEY (`id`),
	UNIQUE indexName (title(length))
);
```

**3、组合索引**
平时用的SQL查询
语句一般都有比较多的限制条件，所以为了进一步榨取MySQL的效率，就要考虑建立组合索引。例如上表中针对title和time建立一个组合索引：

```sql
ALTER TABLE article ADD INDEX index_titme_time (title(50),time(10))
```

建立这样的组合索引，其实是相当于分别建立了下面两组组合索引：

```
–title,time

–title
```

为什么没有time这样的组合索引呢？这是因为**MySQL组合索引**“**最左前缀**”的结果。
**简单的理解就是只从最左面的开始组合。并不是只要包含这两列的查询都会用到该组合索引**，如下面的几个SQL所示：

- 使用到上面的索引

```sql
SELECT * FROM article WHREE title='测试' AND time=1234567890;

SELECT * FROM article WHREE utitle='测试';
```

- 未使用到上面的索引

```sql
SELECT * FROM article WHREE time=1234567890;
```

**总结：**
常用的索引类型
**1、普通索引
2、唯一索引
3、组合索引**
**普通索引和唯一索引**的创建方式有三种，分别是**直接创建、修改表结构创建、创建表时同时创建**，注意组合索引的组合规则是最左前缀索引

# **索引分类**

1.普通索引：不附加任何限制条件，可创建在任何数据类型中

2.唯一性索引：使用unique参数可以设置索引为唯一性索引，在创建索引时，限制该索引的值必须唯一，主键就是一种唯一性索引

3.全文索引：使用fulltext参数可以设置索引为全文索引。全文索引只能创建在char、varchar或text类型的字段上。查询数据量较大的字符串类型字段时，效果明显。但只有MyISAM存储引擎支持全文检索

4.单列索引：在表中单个字段上创建的索引，单列索引可以是任何类型，只要保证索引只对应一个一个字段

5.多列索引：在表中多个字段上创建的索引，该索引指向创建时对应的多个字段

6.空间索引：使用spatial参数可以设置索引为空间索引，空间索引只能建立在空间数据类型上比如geometry，并且不能为空，目前只有MyISAM存储引擎支持

# **执行计划**

explain 执行计划

![image-20201204204335648](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20201204204335648.png)

***Id:*** MySQL QueryOptimizer 选定的执行计划中查询的序列号。表示查询中执行select 子句或操作表的顺序,id 值越大优先级越高,越先被执行。id 相同,执行顺序由上至下。

***Select_type:*** 一共有9中类型,只介绍常用的4种:

> SIMPLE: 简单的 select 查询,不使用 union 及子查询
>
> PRIMARY: 最外层的 select 查询
>
> UNION: UNION 中的第二个或随后的 select 查询,不 依赖于外部查询的结果集
>
> DERIVED: 用于 from 子句里有子查询的情况。 MySQL 会 递归执行这些子查询, 把结果放在临时表里。

***Table:*** 输出行所引用的表

***Type:*** 从优到差的顺序如下:（红色标识的是常见的级别。）

system-->***const-->eq_ref-->ref***-->ref_or_null-->index_merge-->unique_subquery-->index_subquery-->***range-->index-->all***.

各自的含义如下:

```sql
system:  表仅有一行。这是 const 连接类型的一个特例。

const:  const 用于用常数值比较 PRIMARY KEY 时。

eq_ref: 查询使用了索引为主键或唯一键的全部时使用。即：通过索引关键字可能查找到一个符合条件的行。

ref:  通过索引关键字可能查找到多个符合条件的行。

ref_or_null:  如同 ref, 但是 MySQL 必须在初次查找的结果里找出 null 条目,然后进行二次查找。

index_merge: 说明索引合并优化被使用了。

unique_subquery:  在某些 IN 查询中使用此种类型,而不是常规的 ref:valueIN (SELECT primary_key FROM single_table WHERE some_expr)

index_subquery:  在 某 些 IN 查 询 中 使 用 此 种 类 型 , 与unique_subquery 类似,但是查询的是非唯一 性索引

range:  检索给定范围的行。当使用 <>、>、>=、<、<=、BETWEEN 或者 IN 操作符时,会使用到range。

index:  全表扫描,只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大。

all: 最坏的情况,从头到尾全表扫描。
```

***possible_keys :*** 哪些索引可能有助于查询。如果为空,说明没有可用的索引。

***key:*** 实际从 possible_key 选择使用的索引，如果为 NULL,则没有使用索引。很少的情况 下,MYSQL 会选择优化不足的索引。这种情 况下,可以在 SELECT语句中使用 USE INDEX (indexname)来强制使用一个索引或者用IGNORE INDEX(indexname)来强制 MYSQL 忽略索引

***key_len:*** 使用的索引的长度。在不损失精确性的情况 下,长度越短越好。

***ref:*** 显示索引的哪一列被使用了 

***rows:***  请求数据返回的大概行数

***extra:*** 其他信息，出现Using filesort、Using temporary 意味着不能使用索引,效率会受到重大影响。应尽可能对此进行优化。

> Using filesort: 没有办法利用现有索引进行排序，需要额外排序，建议：根据排序需要，创建相应合适的索引
>
> Using temporary: 需要用临时表存储结果集，通常是因为group by的列列上没有索引。也有可能是因为同
> 时有group by和order by，但group by和order by的列又不一样 
>
> Using index ： 利用覆盖索引，无需回表即可取得结果数据（即数据直接从索引文件中读取），这种结果是好的。

其中重要的几个就是 **key、type 、rows、extra**，其中key为null、all 、index时，需要调整、优化索引。一般需要达到 ref、eq_ref 级别，范围查找需要达到 range，extra有Using filesort、Using temporary 的一定需要优化，根据rows可以直观看出优化结果。