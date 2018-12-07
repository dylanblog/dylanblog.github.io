---
layout: post
title: "MySQL Explain详解"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Mysql
  - explain
  - sql优化
---

<a href="https://www.cnblogs.com/xuanzhi201111/p/4175635.html" target="_blank">原文地址</a>

在日常工作中，我们会有时会开慢查询去记录一些执行时间比较久的SQL语句，找出这些SQL语句并不意味着完事了，些时我们常常用到explain这个命令来查看一个这些SQL语句的执行计划，查看该SQL语句有没有使用上了索引，有没有做全表扫描，这都可以通过explain命令来查看。所以我们深入了解MySQL的基于开销的优化器，还可以获得很多可能被优化器考虑到的访问策略的细节，以及当运行SQL语句时哪种策略预计会被优化器采用。（QEP：sql生成一个执行计划query Execution plan）

```

mysql> explain select * from servers;
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | servers | ALL  | NULL          | NULL | NULL    | NULL |    1 | NULL  |
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
1 row in set (0.03 sec)
```

expain 出来的信息有10列，分别是id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra,下面对这些字段出现的可能进行解释：

##### id
我的理解是SQL执行的顺序的标识,SQL从大到小的执行

- id相同时，执行顺序由上至下
- 如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
- id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

##### select_type
示查询中每个select子句的类型

- SIMPLE(简单SELECT,不使用UNION或子查询等)
- PRIMARY(查询中若包含任何复杂的子部分,最外层的select被标记为PRIMARY)
- UNION(UNION中的第二个或后面的SELECT语句)
- DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)
- UNION RESULT(UNION的结果)
- SUBQUERY(子查询中的第一个SELECT)
- DEPENDENT SUBQUERY(子查询中的第一个SELECT，取决于外面的查询)
- DERIVED(派生表的SELECT, FROM子句的子查询)
- UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

##### table

显示这一行的数据是关于哪张表的，有时不是真实的表名字,看到的是derivedx(x是个数字,我的理解是第几步执行的结果)

```
mysql> explain select * from (select * from ( select * from t1 where id=2602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | <derived3> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  3 | DERIVED     | t1         | const  | PRIMARY,idx_t1_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```
##### type

表示MySQL在表中找到所需行的方式，又称“访问类型”。

常用的类型有： ALL, index,  range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）
- ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行
- index: Full Index Scan，index与ALL区别为index类型只遍历索引树
- range:只检索给定范围的行，使用一个索引来选择行
- ref: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值
- eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件
- const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量,system是const类型的特例，当查询的表只有一行的情况下，使用system
- NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

##### possible_keys

指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用

该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。
如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询

##### Key

key列显示MySQL实际决定使用的键（索引）

如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。

##### key_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）

不损失精确性的情况下，长度越短越好

##### ref

表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

##### rows

 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数

##### Extra

该列包含MySQL解决查询的详细信息,有以下几种情况：

- Using index：出现这个说明mysql使用了覆盖索引，避免访问了表的数据行，效率不错！如果同时出现using where，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。
- Using where:列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤
- Using temporary：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询
- Using filesort：MySQL中无法利用索引完成的排序操作称为“文件排序”
- Using join buffer：该值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。
- Impossible where：这个值强调了where语句会导致没有符合条件的行。
- Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行

总结：
- EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况
- EXPLAIN不考虑各种Cache
- EXPLAIN不能显示MySQL在执行查询时所作的优化工作
- 部分统计信息是估算的，并非精确值
- EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划。


#### 优化order by 语句

MySQL的排序方式

优化order by语句就不得不了解mysql的排序方式。

- 第一种通过有序索引返回数据，这种方式的extra显示为Using Index,不需要额外的排序，操作效率较高。
- 第二种是对返回的数据进行排序，也就是通常看到的Using filesort，filesort是通过相应的排序算法，将数据放在sort_buffer_size系统变量设置的内存排序区中进行排序，如果内存装载不下，它就会将磁盘上的数据进行分块，再对各个数据块进行排序，然后将各个块合并成有序的结果集。

##### filesort的优化

了解了MySQL排序的方式，优化目标就清晰了：尽量减少额外的排序，通过索引直接返回有序数据。where条件和order by使用相同的索引。

- 创建合适的索引减少filesort的出现。
- 查询时尽量只使用必要的字段，select 具体字段的名称，而不是select * 选择所有字段，这样可以减少排序区的使用，提高SQL性能。

##### 优化group by 语句

为什么order by后面不能跟group by ?

事实上，MySQL在所有的group by 后面隐式的加了order by ，也就是说group by语句的结果会默认进行排序。

如果你要在order by后面加group by ，那结果执行的SQL是不是这样：select * from tb order by … group by … order by … ？ 这不是搞笑吗？

禁止排序

既然知道问题了，那么就容易优化了，如果查询包括group by但又不关心结果集的顺序，而这种默认排序又导致了需要文件排序，则可以指定order by null 禁止排序。

例如：
```
select * from tb group by name order by null;
```

##### 优化limit 分页

一个非常常见又非常头痛的场景：‘limit 1000,20’。

这时MySQL需要查询1020条记录然后只返回最后20条，前面的1000条都将被抛弃，这样的代价非常高。如果所有页面的访问频率都相同，那么这样的查询平均需要访问半个表的数据。

第一种思路 在索引上分页

在索引上完成分页操作，最后根据主键关联回原表查询所需要的其他列的内容。

例如：
```
SELECT * FROM tb_user LIMIT 1000,10
```

可以优化成这样：
```
SELECT * FROM tb_user u
INNER JOIN (SELECT id FROM tb_user LIMIT 1000,10) AS b ON b.id=u.id
```

第二种思路 将limit转换成位置查询

这种思路需要加一个参数来辅助，标记分页的开始位置：
```
SELECT * FROM tb_user WHERE id > 1000 LIMIT 10
```

##### 优化子查询

子查询，也就是查询中有查询，常见的是where后面跟一个括号里面又是一条查询sql

尽可能的使用join关联查询来代替子查询。

当然 这不是绝对的，比如某些非常简单的子查询就比关联查询效率高，事实效果如何还要看执行计划。

只能说大部分的子查询都可以优化成Join关联查询。

改变执行计划

##### 提高索引优先级

use index 可以让MySQL去参考指定的索引，但是无法强制MySQL去使用这个索引，当MySQL觉得这个索引效率太差，它宁愿去走全表扫描。。。
```
SELECT * FROM tb_user USE INDEX (user_name)
```

注意：必须是索引，不能是普通字段，（亲测主键也不行）。

##### 忽略索引

ignore index 可以让MySQL忽略一个索引
```
SELECT * FROM tb_user IGNORE INDEX (user_name) WHERE user_name="张学友"
```

##### 强制使用索引

使用了force index 之后 尽管效率非常低，MySQL也会照你的话去执行
```
SELECT * FROM tb_user FORCE INDEX (user_name) WHERE user_name="张学友"
```

查看执行计划时建议依次观察以下几个要点：

- SQL内部的执行顺序。
- 查看select的查询类型。
- 实际有没有使用索引。
- Extra描述信息