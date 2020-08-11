---
title: "MongoDB分片集群原理、搭建及测试详解"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["SQL", "优化"]
categories: ["SQL"]
author: "ChavinKing"
---



随着技术的发展，目前数据库系统对于海量数据的存储和高效访问海量数据要求越来越高，MongoDB分片机制就是为了解决海量数据的存储和高效海量数据访问而生。
MongoDB分片集群由mongos路由进程（轻量级且非持久化进程）、复制集组成的片shards（分片一般基于复制集故障转移和冗余备份功能）、一组配置服务器（存储元数据信息，一般冗余3台）构成。

一、部署MongoDB分片集群

mongod参数可以通过"mongod --help"查看。
mongos参数可以通过"mongos --help"查看。

1、配置复制集rs0：

参考文档：http://www.cnblogs.com/wcwen1990/p/8053860.html

创建rs0复制集数据目录、日志目录和配置文件。

rs0配置文件：

cat /home/mongodb/db_rs0/config_rs0/rs0.conf

dbpath = /home/mongodb/db_rs0/data/rs0
logpath = /home/mongodb/db_rs0/logs/rs0.log
logappend = true
journal = true
port = 40000
fork = true
maxConns = 5000
bind_ip = 0.0.0.0
replSet = rs0
shardsvr = true
auth = false

2、配置复制集rs1：

参考文档：http://www.cnblogs.com/wcwen1990/p/8053860.html

创建rs1复制集数据目录、日志目录和配置文件。

rs1配置文件：

cat /home/mongodb/db_rs1/config_rs1/rs1.conf

dbpath = /home/mongodb/db_rs1/data/rs1
logpath = /home/mongodb/db_rs1/logs/rs1.log
logappend = true
journal = true
port = 40001
fork = true
maxConns = 5000
bind_ip = 0.0.0.0
replSet = rs1
shardsvr = true
auth = false

3、配置configura服务器，共3台：

configura服务器也是一个mongod进程，它与我们熟悉的普通mongod进程没有本质区别，只是它上面的数据库和集合存储的是分片集群的元数据信息。
创建configura服务器的数据目录、日志目录和配置文件。

configura配置文件：

cat /home/mongodb/db_configs/config_cfgserver/cfgserver.conf

dbpath = /home/mongodb/db_configs/data/db_config
logpath = /home/mongodb/db_configs/logs/config.log
logappend = true
port = 40002
maxConns = 5000
bind_ip = 0.0.0.0
replSet = cfgset
configsvr = true
auth = false
fork = true

4、配置路由服务器：

mongos路由进程功能为整个分片集群构建一个统一的访问客户端，使复杂的分片集群对用户来说是透明的。上文提到过mongos路由进程是一个轻量级且非持久化的进程，其原因是它不需要像其他进程一样创建数据目录dbpath，只需要创建一个日志目录即可。

mongos配置文件内容如下：

cat /home/mongodb/mongos/cfg_mongos.conf

logpath = /home/mongodb/mongos/logs/mongos.log
logappend = true
port = 40003
fork = true
maxConns = 5000
bind_ip = 0.0.0.0
configdb = cfgset/db01:40002,db02:40002,db03:40002

5、分别启动步骤1、步骤2、步骤3、步骤4配置的9个mongod进程和1个mongos进程：

1）启动三个rs0复制集：

bin/mongod --config /home/mongodb/db_rs0/config_rs0/rs0.conf

2）启动三个rs1复制集：

bin/mongod --config /home/mongodb/db_rs1/config_rs1/rs1.conf

3）启动三个配置服务器,并且初始化配置服务器：

bin/mongod --config /home/mongodb/db_configs/config_cfgserver/cfgserver.conf

登录配置服务器：

bin/mongo --port 40003

执行初始化操作：

rs.initiate({_id:"cfgset",configsvr:true, members:[{_id:1,host:"db01:40002"},{_id:2,host:"db02:40002"},{_id:3,host:"db03:40002"}]})

4）启动mongos服务器：

bin/mongos --config /home/mongodb/mongos/cfg_mongos.conf

6、添加各个分片到集群

上面已经完成了两个片（复制集）、三个配置服务器、一个路由服务器的配置工作。接下来，我们要将各个分片添加到集群中。

1）打开一个mongo客户端连接mongos服务器：

bin/mongo --port 40003

2）添加两个分片到集群：

sh.addShard("rs0/db01:40000,db02:40000")
sh.addShard("rs1/db01:40001,db02:40001")

3）通过sh.status()检查上面配置是否正确：

MongoDB Enterprise mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5a37e16fd7903ab63ab90ebf")
  }
  shards:
     { "_id" : "rs0", "host" : "rs0/db01:40000,db02:40000", "state" : 1 }
     { "_id" : "rs1", "host" : "rs1/db01:40001,db02:40001", "state" : 1 }
  active mongoses:
     "3.6.0" : 1
  autosplit:
     Currently enabled: yes
  balancer:
     Currently enabled: yes
     Currently running: no
     Failed balancer rounds in last 5 attempts: 0
     Migration Results for the last 24 hours: 
         No recent migrations
  databases:
     { "_id" : "config", "primary" : "config", "partitioned" : true }

4）查看分片集群数据库信息：

MongoDB Enterprise mongos> show dbs
admin  0.000GB
config 0.000GB
MongoDB Enterprise mongos> db
test
MongoDB Enterprise mongos> use config
switched to db config
MongoDB Enterprise mongos> show collections
changelog
chunks
lockpings
locks
migrations
mongos
shards
tags
transactions
version

至此，MongoDB分片集群部署成功，生产部署还需要调整一些参数，这部门内容可以通过--help查看参数详情。

三、测试MongoDB分片集群

1、向集群插入文档：

MongoDB Enterprise mongos> use chavin
switched to db chavin
MongoDB Enterprise mongos> db.users.insert({userid:1,username:"ChavinKing",city:"beijing"})
WriteResult({ "nInserted" : 1 })

MongoDB Enterprise mongos> db.users.find()
{ "_id" : ObjectId("5a37eabafa5fcca8c960e893"), "userid" : 1, "username" : "ChavinKing", "city" : "beijing" }

2、查看分片集群状态：

MongoDB Enterprise mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5a37e16fd7903ab63ab90ebf")
  }
  shards:
     { "_id" : "rs0", "host" : "rs0/db01:40000,db02:40000", "state" : 1 }
     { "_id" : "rs1", "host" : "rs1/db01:40001,db02:40001", "state" : 1 }
  active mongoses:
     "3.6.0" : 1
  autosplit:
     Currently enabled: yes
  balancer:
     Currently enabled: yes
     Currently running: no
     Failed balancer rounds in last 5 attempts: 0
     Migration Results for the last 24 hours: 
         No recent migrations
  databases:
     { "_id" : "chavin", "primary" : "rs1", "partitioned" : false } //数据库chavin目前不支持分片（"partitioned" : false），数据库文件存储在rs1片上（"primary" : "rs1"）
     { "_id" : "config", "primary" : "config", "partitioned" : true }
         config.system.sessions
             shard key: { "_id" : 1 }
             unique: false
             balancing: true
             chunks:
                 rs0  1
             { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 0)

3、MongoDB分片是针对集合的，要想使集合支持分片，首先需要使其数据库支持分片，为数据库chavin启动分片：

MongoDB Enterprise mongos> sh.enableSharding("chavin")
{
   "ok" : 1,
   "$clusterTime" : {
     "clusterTime" : Timestamp(1513614275, 5),
     "signature" : {
       "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
       "keyId" : NumberLong(0)
     }
   },
   "operationTime" : Timestamp(1513614275, 5)
}

4、为分片字段建立索引，同时为集合指定片键：

MongoDB Enterprise mongos> db.users.ensureIndex({city:1}) //创建索引
{
   "raw" : {
     "rs1/db01:40001,db02:40001" : {
       "createdCollectionAutomatically" : false,
       "numIndexesBefore" : 1,
       "numIndexesAfter" : 2,
       "ok" : 1
     }
   },
   "ok" : 1,
   "$clusterTime" : {
     "clusterTime" : Timestamp(1513614344, 1),
     "signature" : {
       "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
       "keyId" : NumberLong(0)
     }
   },
   "operationTime" : Timestamp(1513614344, 1)
}
MongoDB Enterprise mongos> sh.shardCollection("chavin.users",{city:1}) //启用集合分片，为其指定片键
{
   "collectionsharded" : "chavin.users",
   "collectionUUID" : UUID("a5de7086-115c-44a3-984e-3db8d945dbab"),
   "ok" : 1,
   "$clusterTime" : {
     "clusterTime" : Timestamp(1513614387, 13),
     "signature" : {
       "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
       "keyId" : NumberLong(0)
     }
   },
   "operationTime" : Timestamp(1513614387, 13)
}

5、再次查看分片集群状态：

MongoDB Enterprise mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5a37e16fd7903ab63ab90ebf")
  }
  shards:
     { "_id" : "rs0", "host" : "rs0/db01:40000,db02:40000", "state" : 1 }
     { "_id" : "rs1", "host" : "rs1/db01:40001,db02:40001", "state" : 1 }
  active mongoses:
     "3.6.0" : 1
  autosplit:
     Currently enabled: yes
  balancer:
     Currently enabled: yes
     Currently running: no
     Failed balancer rounds in last 5 attempts: 0
     Migration Results for the last 24 hours: 
         No recent migrations
  databases:
     { "_id" : "chavin", "primary" : "rs1", "partitioned" : true } //此时chavin数据库已经支持分片
         chavin.users
             shard key: { "city" : 1 }
             unique: false
             balancing: true
             chunks:
                 rs1  1
             { "city" : { "$minKey" : 1 } } -->> { "city" : { "$maxKey" : 1 } } on : rs1 Timestamp(1, 0) //目前存在一个片，存储在rs1上
     { "_id" : "config", "primary" : "config", "partitioned" : true }
         config.system.sessions
             shard key: { "_id" : 1 }
             unique: false
             balancing: true
             chunks:
                 rs0  1
             { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 0) 
         
6、向集群插入测试数据：

MongoDB Enterprise mongos> for(var i=1;i<1000000;i++) db.users.insert({userid:i,username:"chavin"+i,city:"beijing"})      
MongoDB Enterprise mongos> for(var i=1;i<1000000;i++) db.users.insert({userid:i,username:"dbking"+i,city:"changsha"})

7、再次查看分片集群状态：

MongoDB Enterprise mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5a37e16fd7903ab63ab90ebf")
  }
  shards:
     { "_id" : "rs0", "host" : "rs0/db01:40000,db02:40000", "state" : 1 }
     { "_id" : "rs1", "host" : "rs1/db01:40001,db02:40001", "state" : 1 }
  active mongoses:
     "3.6.0" : 1
  autosplit:
     Currently enabled: yes
  balancer:
     Currently enabled: yes
     Currently running: no
     Failed balancer rounds in last 5 attempts: 0
     Migration Results for the last 24 hours: 
         1 : Success
  databases:
     { "_id" : "chavin", "primary" : "rs1", "partitioned" : true }
         chavin.users
             shard key: { "city" : 1 }
             unique: false
             balancing: true
             chunks:
                 rs0  1
                 rs1  2
             { "city" : { "$minKey" : 1 } } -->> { "city" : "beijing" } on : rs0 Timestamp(2, 0) //分片1，存储在rs0中，并且标注了范围
             { "city" : "beijing" } -->> { "city" : "guangdong" } on : rs1 Timestamp(2, 1) //分片2，存储在rs1中，并且标注了范围
             { "city" : "guangdong" } -->> { "city" : { "$maxKey" : 1 } } on : rs1 Timestamp(1, 3) //分片3，存储在rs1中，并且标注了范围
     { "_id" : "config", "primary" : "config", "partitioned" : true }
         config.system.sessions
             shard key: { "_id" : 1 }
             unique: false
             balancing: true
             chunks:
                 rs0  1
             { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 0) 
     
8、更加详细的分析需要从集合changelog入手分析：

db.changelog.find()可以查看到具体的动作信息，这里不再赘述。