---
layout: post
title: "Mysql死锁"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Mysql
  - 死锁
---

<a href="https://www.cnblogs.com/LBSer/p/5183300.html" target="_blank">参考文章</a>

https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247484486&idx=1&sn=215450f11e042bca8a58eac9f4a97686&chksm=fd985227caefdb3117b8375f150676f5824aa20d1ebfdbcfb93ff06e23e26efbafae6cf6b48e&scene=21#wechat_redirect

### 如何尽可能避免死锁
1）以固定的顺序访问表和行。比如对第2节两个job批量更新的情形，简单方法是对id列表先排序，后执行，这样就避免了交叉等待锁的情形；又比如对于3.1节的情形，将两个事务的sql顺序调整为一致，也能避免死锁。

2）大事务拆小。大事务更倾向于死锁，如果业务允许，将大事务拆小。

3）在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率。

4）降低隔离级别。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。

5）为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大。

### 如何定位死锁成因
下面以本文开头的死锁案例为例，讲下如何排查死锁成因。

1）通过应用业务日志定位到问题代码，找到相应的事务对应的sql；

因为死锁被检测到后会回滚，这些信息都会以异常反应在应用的业务日志中，通过这些日志我们可以定位到相应的代码，并把事务的sql给梳理出来。

```
start tran
deleteHeartCheckDOByToken
updateSessionUser
...
commit
```
此外，我们根据日志回滚的信息发现在检测出死锁时这个事务被回滚。

2）确定数据库隔离级别。

执行select @@global.tx_isolation，可以确定数据库的隔离级别，我们数据库的隔离级别是RC，这样可以很大概率排除gap锁造成死锁的嫌疑;

3）找DBA执行下show InnoDB STATUS看看最近死锁的日志。

这个步骤非常关键。通过DBA的帮忙，我们可以有更为详细的死锁信息。通过此详细日志一看就能发现，与之前事务相冲突的事务结构如下：

```
start tran
updateSessionUser
deleteHeartCheckDOByToken
...
commit
```