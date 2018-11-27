---
layout: post
title: "建索引时注意字段选择性 & 范围查询注意组合索引的字段顺序"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Mysql
  - 慢查优化
  - Index
---

<a href="https://www.cnblogs.com/zhengyun_ustc/p/slowquery2.html" target="_blank">原文地址</a>

确保亲手查过SQL的执行计划，一定要注意看执行计划里的 possible_keys、key和rows这三个值，让影响行数尽量少，保证使用到正确的索引，
减少不必要的Using temporary/Using filesort；

不要在选择性非常差的字段上建索引，原因参见优化策略A；
查询条件里出现范围查询（如A>7，A in (2,3)）时，要警惕，不要建了组合索引却完全用不上，原因参见优化策略B；

我们先回顾一下字段选择性的基础知识。

#### 1. 字段选择性的基础知识

引子：什么字段都可以建索引吗？

如下表所示，sort 字段的选择性非常差，你可以执行 **show index from ads** 命令可以看到 sort 的 Cardinality（散列程度）只有 9，这种字段上本不应该建索引：

Table|Non_unique|Key_name|Seq_in_index|Column_name|Collation|Cardinality|Sub_part|Packed|Null|Index_type|Comment
---|---|---|---|---|---|---|---|---|---|---|---
ads|1|sort|1|sort|A|9|\N|\N||BTREE

#### 优化策略A：字段选择性

1. 选择性较低索引 可能带来的性能问题
    - 索引选择性=索引列唯一值/表记录数；
    - 选择性越高索引检索价值越高，消耗系统资源越少；选择性越低索引检索价值越低，消耗系统资源越多；

2. 查询条件含有多个字段时，不要在选择性很低字段上创建索引
    - 可通过创建组合索引来增强低字段选择性和避免选择性很低字段创建索引带来副作用；
    - 尽量减少possible_keys，正确索引会提高sql查询速度，过多索引会增加优化器选择索引的代价，不要滥用索引；

#### 2 组合索引字段顺序与范围查询之间的关系

引子：范围查询 city_id in (0,8,10) 能用组合索引 (ads_id,city_id) 吗？

举例，

ac 表有一个组合索引(ads_id,city_id)。

那么如下 ac.city_id IN (0, 8005) 查询条件能用到 ac表的组合索引(ads_id,city_id) 吗？

```sql
EXPLAIN
SELECT ac.ads_id
FROM ads, ac
WHERE
      ads.id = ac.ads_id
      AND ac.city_id IN (0, 8005)
      AND ads.status = 'online'
      AND ac.start_time<UNIX_TIMESTAMP()
      AND ac.end_time>UNIX_TIMESTAMP()
```

#### 优化策略B：

由于 mysql 索引是基于 B-Tree 的，所以组合索引有“字段顺序”概念。

所以，查询条件中有 ac.city_id IN (0, 8005)，而组合索引是 (ads_id,city_id)，则该查询无法使用到这个组合索引。

#### DBA总结道：

组合索引查询的各种场景
兹有 Index (A,B,C) ——组合索引多字段是有序的，并且是个完整的BTree 索引。
下面条件可以用上该组合索引查询：
- A>5
- A=5 AND B>6
- A=5 AND B=6 AND C=7
- A=5 AND B IN (2,3) AND C>5

下面条件将不能用上组合索引查询：
- B>5 ——查询条件不包含组合索引首列字段
- B=6 AND C=7 ——查询条件不包含组合索引首列字段

下面条件将能用上部分组合索引查询：
- A>5 AND B=2 ——当范围查询使用第一列，查询条件仅仅能使用第一列
- A=5 AND B>6 AND C=2 ——范围查询使用第二列，查询条件仅仅能使用前二列

组合索引排序的各种场景

有组合索引 Index(A,B)。

下面条件可以用上组合索引排序：

- ORDER BY A——首列排序
- A=5 ORDER BY B——第一列过滤后第二列排序
- ORDER BY A DESC, B DESC——注意，此时两列以相同顺序排序
- A>5 ORDER BY A——数据检索和排序都在第一列

下面条件不能用上组合索引排序：
- ORDER BY B ——排序在索引的第二列
- A>5 ORDER BY B ——范围查询在第一列，排序在第二列
- A IN(1,2) ORDER BY B ——理由同上
- ORDER BY A ASC, B DESC ——注意，此时两列以不同顺序排序

顺着组合索引怎么建继续往下延伸，请各位注意“索引合并”概念：

- MySQL 5,0以下版本，SQL查询时，一张表只能用一个索引（use at most only one index for each referenced table），
- 从 MySQL 5.0开始，引入了 index merge 概念，包括 Index Merge Union Access Algorithm（多个索引并集访问），包括Index Merge Intersection Access Algorithm（多个索引交集访问），可以在一个SQL查询里用到一张表里的多个索引。
- MySQL 在5.6.7之前，使用 index merge 有一个重要的前提条件：没有 range 可以使用。[出自参考资源2]

索引合并的简单说明：
MySQL 索引合并能使用多个索引

1. SELECT * FROM TB WHERE A=5 AND B=6
    - 能分别使用索引(A) 和 (B) 或 索引合并；
    - 创建组合索引(A,B) 更好；

2. SELECT * FROM TB WHERE A=5 OR B=6
    - 能分别使用索引(A) 和 (B) 或 索引合并；
    - 组合索引(A,B)不能用于此查询，分别创建索引(A) 和 (B)会更好；


#### 索引失效的几种情况
- 如果条件中有or，即使其中有条件带索引也不会使用(这也是为什么尽量少用or的原因),要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引
- 对于多列索引，不是使用的第一部分，则不会使用索引
- like查询以%开头
- 如果列类型是字符串，那一定要在条件中将数据使用引号引用起来,否则不使用索引
- 如果mysql估计使用全表扫描要比使用索引快,则不使用索引

