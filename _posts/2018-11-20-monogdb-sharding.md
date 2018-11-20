---
layout: post
title: "mongodb分片初探"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - MongoDb
  - NoSql
  - sharding
---

<a href="https://www.jianshu.com/p/6648efd24f25" target="_blank">原文地址</a>

分片(sharding)是指将数据库拆分，将其拆分到不同机器的过程，有时也用分区(partitioning)来表示，它是 MongoDB 为应对数据增长需求而采取的办法。
#### 为什么要分片
- 增加单台服务器可用的磁盘空间
- 减轻单台服务器的负载
- 处理单个mongod无法承受的吞吐量

#### 分片原理
  在搭建mongodb分片集群之前，我们需要先了解一下其架构和原理，下图是mongodb分片集群的架构图:

  ![image](https://dylanblog.github.io/img/in-post/2018-11-20-monogdb-sharding.png)

mongodb分片集群由三大组件组成:mongos路由、配置服务器config Server和分片shard，下面一一介绍它们的作用。
##### mongos路由
 mongos是一个前置路由，我们的应用客户端并不是直接与分片连接，而是与mongos路由连接，mongos接收到客户端请求后根据查询信息将请求任务分发到对应的分片，
 在正式生产环境中，为确保高可用性，一般会配置两台以上的mongos路由，以确保当其中一台宕机后集群还能保持高可用。
##### 配置服务器
  配置服务器相当于集群的大脑，它存储了集群元信息:集群中有哪些分片、分片的是哪些集合以及数据块的分布集群启动后，当接收到请求时，如果mongos路由没有缓
  存配置服务器的元信息，会先从配置服务器获取分片集群对于的映射信息。同样的，为了保持集群的高可用，一般会配置多台配置服务器。
##### shard分片
   分片是存储了一个集合部分数据的MongoDB实例,每个分片 可以是一台服务器运行单独一个Mongod实例，但是为了提高系统的可靠性实现自动故障恢复，一个分片应该是一个复制集。

  通过分片，我们将一个集合拆分为多个数据块，这些数据块分别部署在不同的机器上，这样可以做到增加单台机器的磁盘可用空间，同时将查询分配到不同的机器上，减轻单台机器的负载。
#### 如何分片
   在搭建集群分片之前，我们需要先明确各个组件的数量和服务器的数量，我们需要搭建1个mongos路由进程，3个配置服务器进程和3个分片进程，为方便起见，我们把
   2个mongos路由和3个配置服务器都放在同一台服务器，通过不同的端口号来运行，但是3个分片我们用了3台不同的服务器(其实在同一台服务器也是可以的)，
   组件和服务器配置总体如下:
- 机器 10.20.11.225 :1个mongos路由进程，3个配置服务器进行
- 机器 10.20.20.239 :第一个分片
- 机器 10.20.21.27 :第二个分片
- 机器 10.20.23.50 :第三个分片
#### 配置config server
  配置服务器是一个普通的mongod进程，我们在10.20.11.225机器上配置了3个mongod进程，端口号分别是:20000、20001和20002,以20000端口的进程为例，其配置如下：
```
dbpath=/root/fmm/software/mongodb/data/config_20000/
#日志文件
logpath=/root/fmm/software/mongodb/log/config_20000.log
#日志追加
logappend=true
#端口
port = 20000
#最大连接数
maxConns = 50
pidfilepath = /root/fmm/software/mongodb/log/config_20000.pid
#日志,redo log
journal = true
#守护进程模式
fork = true
configsvr = true
20001和20002端口的mongod进程的配置只是端口号、日志和数据保存路径不同而已，然后就分别启动这三个实例:
```
```
./mongod -f ../config/config_20000.conf
./mongod -f ../config/config_20001.conf
./mongod -f ../config/config_20002.conf
```
#### 配置mongos路由
   mongos路由是提供给客户端连接集群的，我们只配置一个就行了，但是在正式生产环境上，为了集群的高可用，我们需要配置两台以上的mongos路由，
   在10.20.11.225机器上我们用30000端口开启一个mongod进行来运行mongos路由，器配置如下:
```
#日志文件
logpath=/root/fmm/software/mongodb/log/mongos_30000.log
#日志追加
logappend=true
#端口
port = 30000
#最大连接数
maxConns = 50
pidfilepath = /root/fmm/software/mongodb/log/mongos_30000.pid
#日志,redo log
#journal = true
#守护进程模式
fork = true
configdb=10.20.11.225:20000,10.20.11.225:20001,10.20.11.225:20002
```
mongos路由不需要保存数据，所以不用配置数据保存地址，但是一定要配置logpath ,方便我们定位问题，配置中最重要的配置是configdb,需要配置的是3台配置服务器的地址，
需要注意的是配置服务器的地址不能写成localhost或者127.0.0.1,否则会报错，配置完后我们需要启动mongos进程，命令如下:
```
./mongos -f ../config/mongos_30000.conf
```
#### 配置分片服务器
  分片服务器上保存着集合的部分数据，我们分别在 10.20.20.239、10.20.21.27、10.20.23.50三台服务器上启动三个mongod进程，端口号都是40000,配置如下:
```
dbpath=/root/fmm/software/mongodb/data/
#日志文件
logpath=/root/fmm/software/mongodb/log/mongodb_shard.log
#日志追加
logappend=true
#端口
port = 40000
#最大连接数
maxConns = 50
pidfilepath = /root/fmm/software/mongodb/log/mongodb_shard.pid
#日志,redo log
journal = true
#守护进程模式
fork = true
```
配置完之后就可以直接启动了，命令如下:
```
./mongod -f ../config/./mongod
```
所有组件配置完之后，我们可以登陆mongos路由，查看整个集群信息:
```
./mongo --port=30000
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5885a29be970437923e2aa86")
}
  shards:
  balancer:
    Currently enabled:  yes
    Currently running:  no
    Failed balancer rounds in last 5 attempts:  0
    Migration Results for the last 24 hours:
        No recent migrations
  databases:
    {  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
```
#### 添加分片

我们需要把三台分片服务器添加到集群中，操作如下:
```
mongos> sh.addShard("10.20.20.239:40000")
{ "shardAdded" : "shard0000", "ok" : 1 }
mongos> sh.addShard("10.20.21.27:40000")
{ "shardAdded" : "shard0001", "ok" : 1 }
mongos> sh.addShard("10.20.23.50:40000")
{ "shardAdded" : "shard0002", "ok" : 1 }
```
添加完之后再看看集群信息，可以看到三个分片已经被添加到集群了:
```
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5885a29be970437923e2aa86")
}
  shards:
    {  "_id" : "shard0000",  "host" : "10.20.20.239:40000" }
    {  "_id" : "shard0001",  "host" : "10.20.21.27:40000" }
    {  "_id" : "shard0002",  "host" : "10.20.23.50:40000" }
  balancer:
    Currently enabled:  yes
    Currently running:  no
    Failed balancer rounds in last 5 attempts:  0
    Migration Results for the last 24 hours:
        No recent migrations
  databases:
    {  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
```
#### 开启分片
   开启分片首先需要对数据库启用分片，然后才能对数据库的集合开启分片功能，对集合开启分片功能时需要确定片键，片键与索引类似，是集合的某一个或者多个字段，mongodb会根据这个片键对数据进行拆分。
   在进行分片时，片键的选择是非常最要的一件事，可以说是整个分片过程中最为重要的一步，所以需要非常谨慎地选择片键，这个可以单独拿出来写一篇很长的文章了，这里不做过多的描述。

在这里我们创建了一个名为"shardtest"的数据库，数据库中有一张"user"表，user表有name,age,和sex三个字段，我们选择用age作为片键，对该集合进行分片( 这是一个木有数据的空集合)，操作如下:
```
mongos> sh.enableSharding("shardtest")       ##对数据库开启分片
mongos> sh.shardCollection("shardtest.user",{"age":1})   ##对集合user以age作为片键进行分片
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5885a29be970437923e2aa86")
}
  shards:
    {  "_id" : "shard0000",  "host" : "10.20.20.239:40000" }
    {  "_id" : "shard0001",  "host" : "10.20.21.27:40000" }
    {  "_id" : "shard0002",  "host" : "10.20.23.50:40000" }
  balancer:
    Currently enabled:  yes
    Currently running:  no
    Failed balancer rounds in last 5 attempts:  0
    Migration Results for the last 24 hours:
        No recent migrations
  databases:
    {  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
    {  "_id" : "test",  "partitioned" : false,  "primary" : "shard0000" }
    {  "_id" : "shardtest",  "partitioned" : true,  "primary" : "shard0000" }
        shardtest.user
            shard key: { "age" : 1 }
            chunks:
                shard0000   1
            { "age" : { "$minKey" : 1 } } -->> { "age" : { "$maxKey" : 1 } } on : shard0000 Timestamp(1, 0)
```
  我们可以看到，由于目前集合中没有数据，所以新分片的集合起初只有一块，此块的范围是负无穷到正无穷，在shell中用$minKey和$maxKey表示，随着块的增长，mongodb会自动降其分成两块。
  我们用脚本插入10w条数据，年龄大小为1~100之间任一值，插入之后，我们再看看其状态:
```
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5885b5067fefeb309298d15a")
}
  shards:
    {  "_id" : "shard0000",  "host" : "10.20.20.239:40000" }
    {  "_id" : "shard0001",  "host" : "10.20.21.27:40000" }
    {  "_id" : "shard0002",  "host" : "10.20.23.50:40000" }
  balancer:
    Currently enabled:  yes
    Currently running:  yes
    Failed balancer rounds in last 5 attempts:  0
    Migration Results for the last 24 hours:
        2 : Success
  databases:
    {  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
    {  "_id" : "test",  "partitioned" : false,  "primary" : "shard0000" }
    {  "_id" : "shardtest",  "partitioned" : true,  "primary" : "shard0000" }
        shardtest.user
            shard key: { "age" : 1 }
            chunks:
                shard0000   1
                shard0001   1
                shard0002   1
            { "age" : { "$minKey" : 1 } } -->> { "age" : 17 } on : shard0001 Timestamp(2, 0)
            { "age" : 17 } -->> { "age" : 81 } on : shard0002 Timestamp(3, 0)
            { "age" : 81 } -->> { "age" : { "$maxKey" : 1 } } on : shard0000 Timestamp(3, 1)
```
可以看到数据被随意插入到三个分片中，如果各个片中的数据不均衡，那么这时候mongodb的均衡器就会发挥作用了，平衡器的作用是管理数据数据块的移动，
当一个集合的数据块在集群中分布达到移动阈值（Migration Thresholds）的时候，平衡器就将数目最多的分片上的数据块移动到数目比较少的分片上，平衡器内容比较多，
这里不做细讲了。至此，我们已经初步完成了分片。