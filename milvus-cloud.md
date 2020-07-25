# Miluvs 云方案的思考

本方案参考 snowflake，整体框架图如下：

![](./milvus-cloud/milvus-cloud.jpg)

---

## meta server为什么是全局的，为什么不是一个miluvs cluster一个meta server?
- meta server 采用etcd，为了保证数据可靠不丢失，至少3台机器，一个miluvs cluster 一套meata server 太费钱， 全局meta server比较节约
- meta数据量比较少，一套全局的 meta server足够支撑服务
- .如果业务迅速扩展导致一套全局的meta server不足以支撑服务，可以部署多个meta server，每个meta server分别负责多个 milvus cluser

---

## 存储端为什么不用Raft?
- Vearch使用 raft 实现，256维度，float，插入1万条，耗时26s，太慢
- 测试环境为办公室的台式机，相同配置，插入10万条，耗时4s

---

## 为什么选择 S3作为storage?
- snowflake的论文对S3做了一番调研，并且提到S3 是 “ hard to beat”
- snowflake针对的场景是OLAP，即数据批量导入，并且delete 和 update 为低频操作，milvus的使用环境满足这种假设

---

## milvus cluster 中的 load balance 和 reduce是干什么的？
- load balance和reduce分别对应这milvus cluster的两种工作模式
- load balance模式中，milvus cluster中的每个 milvus实例都拥有全量数据，针对提高QPS
- reduce中，每个milvus实例拥有部分数据，针对超大数据集查询
- load balance和reduce不能同时工作

---

## 当前milvus需要哪些改变，针对即将发布的 milvus-0.11.0？
- 必须做到计算和存储分离
- 使用etcd存储meta
- 删除 WAL
- 删除自动flush
- milvus cloud需要自动升级，那么milvus后续版本的数据格式必须向后兼容
- 数据文件需要有校验

---

## S3中存什么?
- 原始的向量数据 raw data
- 向量索引文件 index data
- 数据删除记录 delete doc

---

## 计费模式?
- 仿照 snowflake，storage和 milvus cluster单独收费
- 用户只购买 milvus cluster的服务，并不购买 storage服务
- milvus cluster 可以选择机器配置及机器数目，不同的配置对应不同的单价
- 学 snowflake，milvus cluster 按照使用时长收费
- Storage的收费模式学snowflake，按照每个月的平均存储量计算

---

## 计算资源是否可以挂起?
- 学 snowflake，milvus cluster可以挂起
- 创建 milvus cluster时可以设置运行一段时间后自动挂起
- 和snowflake一样，自动挂起需要对应这自动恢复，在milvus cloud收到第一条查询查询指令时，自动恢复最近被被挂起的 milvus cluster
- milvus cluster恢复后，内存中并没有数据，因此第一次查询会比较缓慢 

---

## 是否提供免费的试用途径?
- 提供免费的试用途径
- 用户登录后，在购买milvus cluster时，可以选择免费版本的free cluster
- free cluster只有一个milvus实例，并且限制 memory 和 storage 的使用
- free cluster的目标人群是milvus 用户中的开发人员，使用free cluster做接口调试，mongodb中的M0也是这么设定的
- free cluster在创建后2小时内自动挂起，因为挂起的 milvus cluster 在收到查询指令后会自动回复，所以对开发人员而言是透明的

---

## 用户管理模式?
- 仿照 snowflake，用户管理分两级 account 和 user
- 每个account有独立的URL
- 一个account下可以有多个user，这些user共享一份账单
- 因为当前的milvus并不具备用户管理的能力，所以这个功能可能需要放在 milvus service中的
