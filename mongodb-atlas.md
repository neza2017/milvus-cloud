# MongoDB Atlas
## Paper (01 High)
TODO

---
## Independent Entrance
官网地址 : https://www.mongodb.com/cloud/atlas

---

## Cloud Infra
![](mongodb-atlas/cloud-provider.png)

---

## Infra dependency
不依赖云平台，但是提供额外的直接从 `S3` 查询的功能

![](mongodb-atlas/data-lake.png)


---

## SLA

只针对 `M30` ，并且开机时间必须达到 `24小时` 以上
![](mongodb-atlas/sla.png)
https://www.mongodb.com/cloud/atlas/sla

---

## Product package
该部分在下文详细介绍，包括 `HA`、`DR`、`User Management`等

---

## Pricing model

![](mongodb-atlas/M0.png)
![](mongodb-atlas/M2.png)
![](mongodb-atlas/M5.png)
![](mongodb-atlas/M10.png)
![](mongodb-atlas/M20.png)
![](mongodb-atlas/M30.png)
![](mongodb-atlas/M40.png)
![](mongodb-atlas/M50.png)
![](mongodb-atlas/M60.png)
![](mongodb-atlas/M80.png)
![](mongodb-atlas/M200.png)
![](mongodb-atlas/M300.png)
![](mongodb-atlas/M400.png)

---

## 7x24 Support
提供 7x24 小时支持，但是要收费
![](mongodb-atlas/support-plan.png)

---

## Competition (02 Medium)

TODO

---

## Authentication
官网注册后有独立的操作界面
![](mongodb-atlas/user-page.png)

创建数据库

![](mongodb-atlas/database.png)

链接`mongoDB`

![](mongodb-atlas/connect.png)

`mongdb-shell`

![](mongodb-atlas/mongo-shell.png)

本机链接 `mongodb` 测试
```bash
$ mongo "mongodb+srv://cluster0.6jzw6.gcp.mongodb.net" -u zilliz -p XD2XgHiKisktTzS
MongoDB shell version v4.2.8
connecting to: mongodb://cluster0-shard-00-00.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-01.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-02.6jzw6.gcp.mongodb.net:27017/?authSource=admin&compressors=disabled&gssapiServiceName=mongodb&replicaSet=atlas-fa9q4g-shard-0&ssl=true
2020-07-20T07:56:12.601+0000 I  NETWORK  [js] Starting new replica set monitor for atlas-fa9q4g-shard-0/cluster0-shard-00-00.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-01.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-02.6jzw6.gcp.mongodb.net:27017
2020-07-20T07:56:12.601+0000 I  CONNPOOL [ReplicaSetMonitor-TaskExecutor] Connecting to cluster0-shard-00-00.6jzw6.gcp.mongodb.net:27017
2020-07-20T07:56:12.602+0000 I  CONNPOOL [ReplicaSetMonitor-TaskExecutor] Connecting to cluster0-shard-00-02.6jzw6.gcp.mongodb.net:27017
2020-07-20T07:56:12.602+0000 I  CONNPOOL [ReplicaSetMonitor-TaskExecutor] Connecting to cluster0-shard-00-01.6jzw6.gcp.mongodb.net:27017
2020-07-20T07:56:13.096+0000 I  NETWORK  [ReplicaSetMonitor-TaskExecutor] Confirmed replica set for atlas-fa9q4g-shard-0 is atlas-fa9q4g-shard-0/cluster0-shard-00-00.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-01.6jzw6.gcp.mongodb.net:27017,cluster0-shard-00-02.6jzw6.gcp.mongodb.net:27017
Implicit session: session { "id" : UUID("fa955147-338c-4053-b0ef-73685daca91d") }
MongoDB server version: 4.2.8

MongoDB Enterprise atlas-fa9q4g-shard-0:PRIMARY> use sample_airbnb
switched to db sample_airbnb

MongoDB Enterprise atlas-fa9q4g-shard-0:PRIMARY> db.listingsAndReviews.count()
5555
```
---

## Scale out
Yes，详见 `Pricing model`

注意事项

![](mongodb-atlas/scaling-up.png)

---

## HA
Yes，详见 `SLA`

---

## DR
Yes

数据备份： https://docs.atlas.mongodb.com/backup-restore-cluster/#backup-methods

`M2/M5`采用快照备份

![](mongodb-atlas/M2-5-snapshots.png)

`M10`以上使用云平台的`snapshot`备份

![](mongodb-atlas/M10-backup.png)

数据恢复： 

`M2/M5`数据恢复 : https://docs.atlas.mongodb.com/backup/shared-tier/restore/#m2-m5-restore-snapshot

![](mongodb-atlas/M2-5-restore.png)


`M10+` 数据恢复 : https://docs.atlas.mongodb.com/backup/cloud-backup/restore/#restoration-cloud-provider-snapshot

![](mongodb-atlas/M10-restore.png)


---

## Scale up
Yes，详见 `SLA`

---

## Serverless (03 low)
TODO

---

## Disaggregated Storage and Compute Architecture (03 low)
TODO 


---

## User Management
可以直接在网页上创建登录数据库的用户

![](mongodb-atlas/user-access.png)

可以设置登录`ip`白名单

![](mongodb-atlas/network-access.png)

---

## Regulation/Compliance (01 High)
TODO

