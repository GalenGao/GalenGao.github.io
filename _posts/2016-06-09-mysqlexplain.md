---
layout: post
title:  "MYSQL EXPLAIN执行计划命令详解（支持更新中）"
date:   2016-06-09 14:25:20 +0700
categories: [mysql]
---

**摘要:**  

* **本篇是根据官网中的每个一点来翻译、举例、验证的；英语不好，所以有些话语未必准确，请自行查看官网，若有些点下面没有例子的是因为当时一下子没有想出那么多来，如果大家有遇上好的例子，欢迎在下面留言我持续更新**
* **查看执行计划的关键EXPLAIN**
* **版本MYSQL5.6，用到的库是官网例子sakila，自行下载导入**


**由于要把每个点都翻译出来，还需要举例，所以需要一定的时间，本人先把架构理出来，然后逐个点开始**  

**官网地址：http://dev.mysql.com/doc/refman/5.6/en/explain-output.html** 
 
**EXPLAIN语句返回MYSLQ的执行计划，通过他返回的信息，我们能了解到MYSQL优化器是如何执行SQL语句的，通过分析他能帮助你提供优化的思路。**  


# 语法

> MYSQL 5.6.3以前只能EXPLAIN SELECT;
> MYSQL5.6.3以后就可以EXPLAIN SELECT,UPDATE,DELETE

* EXPLAIN 语法例子：  

{% highlight ruby %}
mysql> explain select customer_id,a.store_id,first_name,last_name, b.manager_staff_id from customer a left join store b on a.store_id=b.store_id;
+----+-------------+-------+--------+---------------+---------+---------+-------------------+------+-------+
| id | select_type | table | type   | possible_keys | key     | key_len | ref               | rows | Extra |
+----+-------------+-------+--------+---------------+---------+---------+-------------------+------+-------+
|  1 | SIMPLE      | a     | ALL    | NULL          | NULL    | NULL    | NULL              |  599 | NULL  |
|  1 | SIMPLE      | b     | eq_ref | PRIMARY       | PRIMARY | 1       | sakila.a.store_id |    1 | NULL  |
+----+-------------+-------+--------+---------------+---------+---------+-------------------+------+-------+
2 rows in set
{% endhighlight %}

* EXPLAIN还有一种语法，类似于desc

{% highlight ruby %}
mysql> explain actor;
+-------------+----------------------+------+-----+-------------------+-----------------------------+
| Field       | Type                 | Null | Key | Default           | Extra                       |
+-------------+----------------------+------+-----+-------------------+-----------------------------+
| actor_id    | smallint(5) unsigned | NO   | PRI | NULL              | auto_increment              |
| first_name  | varchar(45)          | NO   |     | NULL              |                             |
| last_name   | varchar(45)          | NO   | MUL | NULL              |                             |
| last_update | timestamp            | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------------+----------------------+------+-----+-------------------+-----------------------------+
4 rows in set
{% endhighlight %}

# EXPLAIN的输出

EXPLAIN主要包含以下信息：

|Column	   |JSON Name	   |Meaning|
|----------|---------------|--------|
|id	|select_id	|The SELECT identifier|
|select_type	|None	|The SELECT type|
|table	|table_name	|The table for the output row|
|partitions	|partitions	|The matching partitions|
|type	|access_type	|The join type|
|possible_keys	|possible_keys	|The possible indexes to choose|
|key	|key	|The index actually chosen|
|key_len	|key_length	|The length of the chosen key|
|ref	|ref	|The columns compared to the index|
|rows	|rows	|Estimate of rows to be examined|
|filtered	|filtered	|Percentage of rows filtered by table condition|
|Extra	|None	|Additional information|



### id (JSON name: select_id)
 
SQL查询中的序列号。

### select_type (JSON name: none)
 
查询的类型，可以是下表的任何一种类型：  

|select_type Value      |	JSON Name|	Meaning|
|-----------------------|------------|---------|
|SIMPLE	|None|简单查询(不适用union和子查询的)|
|PRIMARY|	None|最外层的查询|
|UNION|	None|UNION中的第二个或者后面的SELECT语句|
|DEPENDENT UNION|	dependent (true)|UNION中的第二个或者后面的SELECT语句,依赖于外部查询|
|UNION RESULT	|union_result|	UNION结果|
|SUBQUERY|	None|子查询中的第一个SELECT语句|
|DEPENDENT SUBQUERY|	dependent (true)|子查询中的第一个SELECT语句，依赖于外部查询|
|DERIVED|	None	|派生表的SELECT(FROM子句的子查询)|
|MATERIALIZED|	materialized_from_subquery|	物化子查询|
|UNCACHEABLE SUBQUERY|	cacheable (false)|	对于该结果不能被缓存，必须重新评估外部查询的每一行子查询|
|UNCACHEABLE UNION|	cacheable (false)|	UNION中的第二个或者后面的SELECT语句属于不可缓存子查询 (see UNCACHEABLE SUBQUERY)|

查询类型例子：

**1、SIMPLE 简单查询(不适用union和子查询的)**

{% highlight ruby %}
mysql> explain select * from staff;
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | staff | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL  |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
1 row in set
{% endhighlight %}

**2、PRIMARY 最外层的查询**

{% highlight ruby %}
mysql> explain select * from (select last_name,first_name from customer) a;
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | ALL  | NULL          | NULL | NULL    | NULL |  599 | NULL  |
|  2 | DERIVED     | customer   | ALL  | NULL          | NULL | NULL    | NULL |  599 | NULL  |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
2 rows in set
{% endhighlight %}

**3、UNION UNION中的第二个或者后面的SELECT语句**

{% highlight ruby %}
mysql> explain select first_name,last_name from customer a where customer_id=1 union select first_name,last_name from customer b where customer_id=2;
+------+--------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
| id   | select_type  | table      | type  | possible_keys | key     | key_len | ref   | rows | Extra           |
+------+--------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
|    1 | PRIMARY      | a          | const | PRIMARY       | PRIMARY | 2       | const |    1 | NULL            |
|    2 | UNION        | b          | const | PRIMARY       | PRIMARY | 2       | const |    1 | NULL            |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL          | NULL    | NULL    | NULL  | NULL | Using temporary |
+------+--------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
3 rows in set
{% endhighlight %}

**4、DEPENDENT UNION UNION中的第二个或者后面的SELECT语句，依赖于外部查询**

{% highlight ruby %}
mysql> explain select * from customer where customer_id in(select customer_id from customer a where customer_id=1 union all select customer_id from customer b where customer_id=2);
+------+--------------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
| id   | select_type        | table      | type  | possible_keys | key     | key_len | ref   | rows | Extra           |
+------+--------------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
|    1 | PRIMARY            | customer   | ALL   | NULL          | NULL    | NULL    | NULL  |  599 | Using where     |
|    2 | DEPENDENT SUBQUERY | a          | const | PRIMARY       | PRIMARY | 2       | const |    1 | Using index     |
|    3 | DEPENDENT UNION    | b          | const | PRIMARY       | PRIMARY | 2       | const |    1 | Using index     |
| NULL | UNION RESULT       | <union2,3> | ALL   | NULL          | NULL    | NULL    | NULL  | NULL | Using temporary |
+------+--------------------+------------+-------+---------------+---------+---------+-------+------+-----------------+
4 rows in set
{% endhighlight %}

**5、UNION RESULT UNION结果**

{% highlight ruby %}
mysql> explain select * from staff union select * from staff;
+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+
| id   | select_type  | table      | type | possible_keys | key  | key_len | ref  | rows | Extra           |
+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+
|    1 | PRIMARY      | staff      | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL            |
|    2 | UNION        | staff      | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL            |
| NULL | UNION RESULT | <union1,2> | ALL  | NULL          | NULL | NULL    | NULL | NULL | Using temporary |
+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+
3 rows in set
{% endhighlight %}

**6、SUBQUERY 子查询中的第一个SELECT语句**

{% highlight ruby %}
mysql> explain select customer_id from customer where store_id =
(select store_id from store where store_id=1);
+----+-------------+----------+-------+-----------------+-----------------+---------+-------+------+--------------------------+
| id | select_type | table    | type  | possible_keys   | key             | key_len | ref   | rows | Extra                    |
+----+-------------+----------+-------+-----------------+-----------------+---------+-------+------+--------------------------+
|  1 | PRIMARY     | customer | ref   | idx_fk_store_id | idx_fk_store_id | 1       | const |  326 | Using where; Using index |
|  2 | SUBQUERY    | store    | const | PRIMARY         | PRIMARY         | 1       | const |    1 | Using index              |
+----+-------------+----------+-------+-----------------+-----------------+---------+-------+------+--------------------------+
2 rows in set
{% endhighlight %}

> 有兴趣的可以去把=号换成`in`试试

**7、DERIVED 派生表的SELECT(FROM子句的子查询)**

{% highlight ruby %}
mysql> explain select * from (select * from customer) a;
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | ALL  | NULL          | NULL | NULL    | NULL |  599 | NULL  |
|  2 | DERIVED     | customer   | ALL  | NULL          | NULL | NULL    | NULL |  599 | NULL  |
+----+-------------+------------+------+---------------+------+---------+------+------+-------+
2 rows in set
{% endhighlight %}

**8、其它如物化视图等查询自己去造例子去**


### table(JSON name: table_name)

显示这一行的数据是关于哪张表的，也可以是下列值之一：  
unionM,N: The row refers to the union of the rows with id values of M and N.  
derivedN: The row refers to the derived table result for the row with an id value of N. A derived table may result, for example, from a subquery in the FROM clause.  
subqueryN: The row refers to the result of a materialized subquery for the row with an id value of N.  

### partitions (JSON name: partitions)

分区中的记录将被查询相匹配。显示此列仅在使用分区关键字。该值为NULL对于非分区表。  

### type (JSON name: access_type)

EXPLAIN输出的类型列描述了表的连接方法。下面的列表介绍了连接类型，从最好的类型到最差的命令：    

**1、system**   
这是const的一个特例联接类型。表只有一行（=系统表）。  

{% highlight ruby %}
mysql> explain select * from (select * from customer where customer_id=1) a;
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
| id | select_type | table      | type   | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL    | NULL    | NULL  |    1 | NULL  |
|  2 | DERIVED     | customer   | const  | PRIMARY       | PRIMARY | 2       | const |    1 | NULL  |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
2 rows in set
{% endhighlight %}

**2、const**   

表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次！   
const用于用常数值比较PRIMARY KEY或UNIQUE索引的所有部分时。在下面的查询中，tbl_name可以用于const表：  

{% highlight ruby %}
SELECT * from tbl_name WHERE primary_key=1；
SELECT * from tbl_name WHERE primary_key_part1=1和 primary_key_part2=2；
{% endhighlight %}

**3、eq_ref**  

对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型。它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY。   
eq_ref可以用于使用= 操作符比较的带索引的列。比较值可以为常量或一个使用在该表前面所读取的表的列的表达式。  
在下面的例子中，MySQL可以使用eq_ref联接来处理ref_tables：  

{% highlight ruby %}
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
{% endhighlight %}

{% highlight ruby %}
# 相对于下面的ref区别就是它使用的唯一索引，即主键或唯一索引，而ref使用的是非唯一索引或者普通索引。id是主键
mysql> explain select a.*,b.* from testa a,testb b where a.id=b.id
;
+----+-------------+-------+--------+---------------+---------+---------+-------------+------+-------------+
| id | select_type | table | type   | possible_keys | key     | key_len | ref         | rows | Extra       |
+----+-------------+-------+--------+---------------+---------+---------+-------------+------+-------------+
|  1 | SIMPLE      | b     | ALL    | NULL          | NULL    | NULL    | NULL        |    1 | Using where |
|  1 | SIMPLE      | a     | eq_ref | PRIMARY       | PRIMARY | 4       | sakila.b.id |    1 | NULL        |
+----+-------------+-------+--------+---------------+---------+---------+-------------+------+-------------+
2 rows in set
{% endhighlight %}

**4、ref**  

对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。    
ref可以用于使用=或<=>操作符的带索引的列。    
在下面的例子中，MySQL可以使用ref联接来处理ref_tables：  

{% highlight ruby %}
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
{% endhighlight %}

{% highlight ruby %}
# 使用非唯一性索引或者唯一索引的前缀扫描，返回匹配某个单独值的记录行。name有非唯一性索引
mysql> explain select * from testa where name='aaa';
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
| id | select_type | table | type | possible_keys | key      | key_len | ref   | rows | Extra                 |
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
|  1 | SIMPLE      | testa | ref  | idx_name      | idx_name | 33      | const |    2 | Using index condition |
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
1 row in set

mysql> explain select a.*,b.* from testa a,testb b where a.name=b.cname;
+----+-------------+-------+------+---------------+----------+---------+----------------+------+-------------+
| id | select_type | table | type | possible_keys | key      | key_len | ref            | rows | Extra       |
+----+-------------+-------+------+---------------+----------+---------+----------------+------+-------------+
|  1 | SIMPLE      | b     | ALL  | NULL          | NULL     | NULL    | NULL           |    1 | Using where |
|  1 | SIMPLE      | a     | ref  | idx_name      | idx_name | 33      | sakila.b.cname |    1 | NULL        |
+----+-------------+-------+------+---------------+----------+---------+----------------+------+-------------+
2 rows in set
{% endhighlight %}

**5、 fulltext**   

使用FULLTEXT索引进行联接。  

**6、ref_or_null**   

该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。  
在下面的例子中，MySQL可以使用ref_or_null联接来处理ref_tables：  

{% highlight ruby %}
SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
{% endhighlight %}

{% highlight ruby %}
mysql> explain select * from (select cusno from testa t1,testb t2 where t1.id=t2.id) t where cusno =2 or cusno is null;
+----+-------------+------------+-------------+---------------+-------------+---------+--------------+------+--------------------------+
| id | select_type | table      | type        | possible_keys | key         | key_len | ref          | rows | Extra                    |
+----+-------------+------------+-------------+---------------+-------------+---------+--------------+------+--------------------------+
|  1 | PRIMARY     | <derived2> | ref_or_null | <auto_key0>   | <auto_key0> | 5       | const        |    2 | Using where; Using index |
|  2 | DERIVED     | t2         | index       | PRIMARY       | PRIMARY     | 4       | NULL         |    1 | Using index              |
|  2 | DERIVED     | t1         | eq_ref      | PRIMARY       | PRIMARY     | 4       | sakila.t2.id |    1 | NULL                     |
+----+-------------+------------+-------------+---------------+-------------+---------+--------------+------+--------------------------+
3 rows in set
{% endhighlight %}

> 此处按照官网的格式未测试出例子来，若有例子的请留言，我测试更新   

**7、index_merge**    

该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。  

> 此处按照官网的格式未测试出例子来，若有例子的请留言，我测试更新   

**8、unique_subquery**  

unique_subquery是一个索引查找函数，可以完全替换子查询，效率更高。  
该类型替换了下面形式的IN子查询的ref：  

{% highlight ruby %}
value IN (SELECT primary_key FROM single_table WHERE some_expr)
{% endhighlight %}

> 此处按照官网的格式未测试出例子来，若有例子的请留言，我测试更新   

**9、index_subquery**  

该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：  
{% highlight ruby %}
value IN (SELECT key_column FROM single_table WHERE some_expr)
{% endhighlight %}

> 此处按照官网的格式未测试出例子来，若有例子的请留言，我测试更新   

**10、range**  

只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。    
当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range  

{% highlight ruby %}
SELECT * FROM tbl_name
  WHERE key_column = 10;

SELECT * FROM tbl_name
  WHERE key_column BETWEEN 10 and 20;

SELECT * FROM tbl_name
  WHERE key_column IN (10,20,30);

SELECT * FROM tbl_name
  WHERE key_part1 = 10 AND key_part2 IN (10,20,30);
{% endhighlight %}

**11、index**  

索引类型与ALL类型一样，除了它是走索引树扫描的，它有两种方式：  

如果该覆盖索引能满足查询的所有数据，那仅仅扫描这索引树。在这种情况下，`Extra`列就会显示用`Using index`。一般仅仅用索引是扫描的比ALL扫描的要快，因为索引树比表数据小很多。  

全表扫描被用到从索引中去读取数据， `Extra`列就不会显示用`Using index`。  

如果查询仅仅是索引列，那MySQL会这个`index`索引类型  

{% highlight ruby %}
mysql> alter table testa add primary key p_id(id);
Query OK, 0 rows affected
Records: 0  Duplicates: 0  Warnings: 0

mysql> create index idx_name on testa(name);
Query OK, 0 rows affected
Records: 0  Duplicates: 0  Warnings: 0

mysql> insert into testa values(2,2,'aaa');
Query OK, 1 row affected

# 因为查询的列name上建有索引，所以如果这样type走的是index
mysql> explain select name from testa;
+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key      | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+
|  1 | SIMPLE      | testa | index | NULL          | idx_name | 33      | NULL |    2 | Using index |
+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+
1 row in set

# 因为查询的列cusno没有建索引，或者查询的列包含没有索引的列，这样查询就会走ALL扫描，如下：
mysql> explain select cusno from testa;
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | testa | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL  |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
1 row in set

# *包含有未见索引的列
mysql> explain select * from testa;
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | testa | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL  |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
1 row in set

{% endhighlight %}

**12、all**  

对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在它情况下很差。通常可以增加更多的索引而不要使用ALL，使得行能基于前面的表中的常数值或列值被检索出。  

### possible_keys (JSON name: possible_keys)

possible_keys列指出MySQL能使用哪个索引在该表中找到行。而下面的key是MYSQL实际用到的索引，这意味着在possible_keys中的是计划中的，而key是实际的，也就是计划中有这个索引，实际执行时未必能用到。  

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询。  

> 例子参考下面

### key (JSON name: key)  

key列显示MySQL实际决定使用的键（索引）。  

如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。 

> 例子参考下面  

### key_len (JSON name: key_length)
 
key_len列显示MySQL决定使用的键长度。如果KEY键是NULL，则长度为NULL。  
使用的索引的长度。在不损失精确性的情况下，长度越短越好.  

> 例子参考下面   

### ref (JSON name: ref)
 
ref列显示使用哪个列或常数与key一起从表中选择行。 
它显示的是列的名字（或单词“const”），MySQL将根据这些列来选择行。  

> 例子参考下面  

### rows (JSON name: rows)  
 
rows列显示MySQL认为它执行查询时必须检查的行数。 

> 例子参考下面  

### filtered (JSON name: filtered)
 
如果你用EXPLAIN EXTENDED将会展示出这列filtered(MySQL5.7缺省就会输出filtered)，它指返回结果的行占需要读到的行(rows列的值)的百分比。按说filtered是个非常有用的值，因为对于join操作，前一个表的结果集大小直接影响了循环的次数。但是我的环境下测试的结果却是，filtered的值一直是100%，也就是说失去了意义。  


**上面部分EXPLAIN展示的列的例子：**   

{% highlight ruby %}
mysql> alter table testa add primary key p_id(id);
Query OK, 0 rows affected
Records: 0  Duplicates: 0  Warnings: 0

mysql> create index idx_name on testa(name);
Query OK, 0 rows affected
Records: 0  Duplicates: 0  Warnings: 0

mysql> insert into testa values(2,2,'aaa');
Query OK, 1 row affected

# 下面possible_keys可能会用到的索引有主键和我建的索引，但是key实际用到的是主键，主键长度是4，ref用的列的名字（或单词“const”，此处用的是常量const，速度快，rows扫描的只有1行
mysql> explain select cusno from testa where id=2 and name='aaa';
+----+-------------+-------+-------+------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys    | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+------------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | testa | const | PRIMARY,idx_name | PRIMARY | 4       | const |    1 | NULL  |
+----+-------------+-------+-------+------------------+---------+---------+-------+------+-------+
1 row in set

# 下面虽然name有索引，但是查询的列cusno没有索引，这时mysql计划possible_keys有索引，但实际key未走索引，若果cusno换成有索引的列，参照下面。
mysql> explain select cusno from testa where name='aaa';
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | testa | ALL  | idx_name      | NULL | NULL    | NULL |    1 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set

# id是有主键索引的，这时实际key已经走索引了，若果查询列换成既有索引的列也有无索引的列，参照下面
mysql> explain select id from testa where name='aaa';
+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+
| id | select_type | table | type | possible_keys | key      | key_len | ref   | rows | Extra                    |
+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+
|  1 | SIMPLE      | testa | ref  | idx_name      | idx_name | 33      | const |    2 | Using where; Using index |
+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+
1 row in set

# 证明只要你查询的列包含有无索引列就不走索引
mysql> explain select cusno,name from testa where name='aaa';
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | testa | ALL  | idx_name      | NULL | NULL    | NULL |    1 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set
{% endhighlight %}

### Extra (JSON name: none)

 该列包含MySQL解决查询的详细信息,下面详细.

**1、Child of 'table' pushed join@1 (JSON: message text)**  

This table is referenced as the child of table in a join that can be pushed down to the NDB kernel. Applies only in MySQL Cluster, when pushed-down joins are enabled. See the description of the ndb_join_pushdown server system variable for more information and examples.    

**2、const row not found (JSON property: const_row_not_found)**  

For a query such as SELECT ... FROM tbl_name, the table was empty.     

**3、Deleting all rows (JSON property: message)**  

For DELETE, some storage engines (such as MyISAM) support a handler method that removes all table rows in a simple and fast way. This Extra value is displayed if the engine uses this optimization.  

**4、Distinct (JSON property: distinct)**  

MySQL is looking for distinct values, so it stops searching for more rows for the current row combination after it has found the first matching row.  

**5、FirstMatch(tbl_name) (JSON property: first_match)**  

The semi-join FirstMatch join shortcutting strategy is used for tbl_name.  

**6、Full scan on NULL key (JSON property: message)**

This occurs for subquery optimization as a fallback strategy when the optimizer cannot use an index-lookup access method.  

**7、Impossible HAVING (JSON property: message)**

The HAVING clause is always false and cannot select any rows.  

**8、Impossible WHERE (JSON property: message)**

The WHERE clause is always false and cannot select any rows.  

**9、Impossible WHERE noticed after reading const tables (JSON property: message)**  

MySQL has read all const (and system) tables and notice that the WHERE clause is always false.  

**10、LooseScan(m..n) (JSON property: message)**

The semi-join LooseScan strategy is used. m and n are key part numbers.  

**11、Materialize, Scan (JSON: message text)**

Before MySQL 5.6.7, this indicates use of a single materialized temporary table. If Scan is present, no temporary table index is used for table reads. Otherwise, an index lookup is used. See also the Start materialize entry.  

As of MySQL 5.6.7, materialization is indicated by rows with a select_type value of MATERIALIZED and rows with a table value of <subqueryN>.  

**12、No matching min/max row (JSON property: message)**

No row satisfies the condition for a query such as SELECT MIN(...) FROM ... WHERE condition.  

**13、no matching row in const table (JSON property: message)**

For a query with a join, there was an empty table or a table with no rows satisfying a unique index condition.

**14、No matching rows after partition pruning (JSON property: message)**

For DELETE or UPDATE, the optimizer found nothing to delete or update after partition pruning. It is similar in meaning to Impossible WHERE for SELECT statements.  

**15、No tables used (JSON property: message)**

The query has no FROM clause, or has a FROM DUAL clause.  

For INSERT or REPLACE statements, EXPLAIN displays this value when there is no SELECT part. For example, it appears for EXPLAIN INSERT INTO t VALUES(10) because that is equivalent to EXPLAIN INSERT INTO t SELECT 10 FROM DUAL.  

**16、Not exists (JSON property: message)**  

MySQL was able to do a LEFT JOIN optimization on the query and does not examine more rows in this table for the previous row combination after it finds one row that matches the LEFT JOIN criteria. Here is an example of the type of query that can be optimized this way:  

SELECT * FROM t1 LEFT JOIN t2 ON t1.id=t2.id
  WHERE t2.id IS NULL;  

Assume that t2.id is defined as NOT NULL. In this case, MySQL scans t1 and looks up the rows in t2 using the values of t1.id. If MySQL finds a matching row in t2, it knows that t2.id can never be NULL, and does not scan through the rest of the rows in t2 that have the same id value. In other words, for each row in t1, MySQL needs to do only a single lookup in t2, regardless of how many rows actually match in t2.  

**17、Range checked for each record (index map: N) (JSON property: message)**  

MySQL found no good index to use, but found that some of indexes might be used after column values from preceding tables are known. For each row combination in the preceding tables, MySQL checks whether it is possible to use a range or index_merge access method to retrieve rows. This is not very fast, but is faster than performing a join with no index at all. The applicability criteria are as described in Section 8.2.1.3, “Range Optimization”, and Section 8.2.1.4, “Index Merge Optimization”, with the exception that all column values for the preceding table are known and considered to be constants.  

Indexes are numbered beginning with 1, in the same order as shown by SHOW INDEX for the table. The index map value N is a bitmask value that indicates which indexes are candidates. For example, a value of 0x19 (binary 11001) means that indexes 1, 4, and 5 will be considered.  

**18、Scanned N databases (JSON property: message)**  

This indicates how many directory scans the server performs when processing a query for INFORMATION_SCHEMA tables, as described in Section 8.2.4, “Optimizing INFORMATION_SCHEMA Queries”. The value of N can be 0, 1, or all.  

**19、Select tables optimized away (JSON property: message)**  

The optimizer determined 1) that at most one row should be returned, and 2) that to produce this row, a deterministic set of rows must be read. When the rows to be read can be read during the optimization phase (for example, by reading index rows), there is no need to read any tables during query execution.  

The first condition is fulfilled when the query is implicitly grouped (contains an aggregate function but no GROUP BY clause). The second condition is fulfilled when one row lookup is performed per index used. The number of indexes read determines the number of rows to read.  

Consider the following implicitly grouped query:  

SELECT MIN(c1), MIN(c2) FROM t1;  

Suppose that MIN(c1) can be retrieved by reading one index row and MIN(c2) can be retrieved by reading one row from a different index. That is, for each column c1 and c2, there exists an index where the column is the first column of the index. In this case, one row is returned, produced by reading two deterministic rows.  

This Extra value does not occur if the rows to read are not deterministic. Consider this query:  

SELECT MIN(c2) FROM t1 WHERE c1 <= 10;  

Suppose that (c1, c2) is a covering index. Using this index, all rows with c1 <= 10 must be scanned to find the minimum c2 value. By contrast, consider this query:  

SELECT MIN(c2) FROM t1 WHERE c1 = 10;
  
In this case, the first index row with c1 = 10 contains the minimum c2 value. Only one row must be read to produce the returned row.  

For storage engines that maintain an exact row count per table (such as MyISAM, but not InnoDB), this Extra value can occur for COUNT(*) queries for which the WHERE clause is missing or always true and there is no GROUP BY clause. (This is an instance of an implicitly grouped query where the storage engine influences whether a deterministic number of rows can be read.)  

**20、Skip_open_table, Open_frm_only, Open_trigger_only, Open_full_table (JSON property: message)**  

These values indicate file-opening optimizations that apply to queries for INFORMATION_SCHEMA tables, as described in Section 8.2.4, “Optimizing INFORMATION_SCHEMA Queries”.  

Skip_open_table: Table files do not need to be opened. The information has already become available within the query by scanning the database directory.  

Open_frm_only: Only the table's .frm file need be opened.  

Open_trigger_only: Only the table's .TRG file need be opened.  

Open_full_table: The unoptimized information lookup. The .frm, .MYD, and .MYI files must be opened.  

**21、Start materialize, End materialize, Scan (JSON: message text)**  

Before MySQL 5.6.7, this indicates use of multiple materialized temporary tables. If Scan is present, no temporary table index is used for table reads. Otherwise, an index lookup is used. See also the Materialize entry.  

As of MySQL 5.6.7, materialization is indicated by rows with a select_type value of MATERIALIZED and rows with a table value of <subqueryN>.  

**22、Start temporary, End temporary (JSON property: message)**  

This indicates temporary table use for the semi-join Duplicate Weedout strategy.  

**23、unique row not found (JSON property: message)**  

For a query such as SELECT ... FROM tbl_name, no rows satisfy the condition for a UNIQUE index or PRIMARY KEY on the table.  

**24、Using filesort (JSON property: using_filesort)**  

MySQL must do an extra pass to find out how to retrieve the rows in sorted order. The sort is done by going through all rows according to the join type and storing the sort key and pointer to the row for all rows that match the WHERE clause. The keys then are sorted and the rows are retrieved in sorted order. See Section 8.2.1.15, “ORDER BY Optimization”.  

**25、Using index (JSON property: using_index)**  

The column information is retrieved from the table using only information in the index tree without having to do an additional seek to read the actual row. This strategy can be used when the query uses only columns that are part of a single index.  

For InnoDB tables that have a user-defined clustered index, that index can be used even when Using index is absent from the Extra column. This is the case if type is index and key is PRIMARY.  

**26、Using index condition (JSON property: using_index_condition)**  

Tables are read by accessing index tuples and testing them first to determine whether to read full table rows. In this way, index information is used to defer (“push down”) reading full table rows unless it is necessary. See Section 8.2.1.6, “Index Condition Pushdown Optimization”.  

**27、Using index for group-by (JSON property: using_index_for_group_by)**  

Similar to the Using index table access method, Using index for group-by indicates that MySQL found an index that can be used to retrieve all columns of a GROUP BY or DISTINCT query without any extra disk access to the actual table. Additionally, the index is used in the most efficient way so that for each group, only a few index entries are read. For details, see Section 8.2.1.16, “GROUP BY Optimization”.  

**28、Using join buffer (Block Nested Loop), Using join buffer (Batched Key Access) (JSON property: using_join_buffer)**  

Tables from earlier joins are read in portions into the join buffer, and then their rows are used from the buffer to perform the join with the current table. (Block Nested Loop) indicates use of the Block Nested-Loop algorithm and (Batched Key Access) indicates use of the Batched Key Access algorithm. That is, the keys from the table on the preceding line of the EXPLAIN output will be buffered, and the matching rows will be fetched in batches from the table represented by the line in which Using join buffer appears.  

In JSON-formatted output, the value of using_join_buffer is always either one of Block Nested Loop or Batched Key Access.  

**29、Using MRR (JSON property: message)**  

Tables are read using the Multi-Range Read optimization strategy. See Section 8.2.1.13, “Multi-Range Read Optimization”.  

**30、Using sort_union(...), Using union(...), Using intersect(...) (JSON property: message)**  

These indicate how index scans are merged for the index_merge join type. See Section 8.2.1.4, “Index Merge Optimization”.  

**31、Using temporary (JSON property: using_temporary_table)**  

To resolve the query, MySQL needs to create a temporary table to hold the result. This typically happens if the query contains GROUP BY and ORDER BY clauses that list columns differently.  

**32、Using where (JSON property: attached_condition)**  

A WHERE clause is used to restrict which rows to match against the next table or send to the client. Unless you specifically intend to fetch or examine all rows from the table, you may have something wrong in your query if the Extra value is not Using where and the table join type is ALL or index.  

Using where has no direct counterpart in JSON-formatted output; the attached_condition property contains any WHERE condition used.  

**33、Using where with pushed condition (JSON property: message)**  

This item applies to NDB tables only. It means that MySQL Cluster is using the Condition Pushdown optimization to improve the efficiency of a direct comparison between a nonindexed column and a constant. In such cases, the condition is “pushed down” to the cluster's data nodes and is evaluated on all data nodes simultaneously. This eliminates the need to send nonmatching rows over the network, and can speed up such queries by a factor of 5 to 10 times over cases where Condition Pushdown could be but is not used. For more information, see Section 8.2.1.5, “Engine Condition Pushdown Optimization”.  