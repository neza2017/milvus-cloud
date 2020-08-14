- DBFS 的写入延迟?
- WAL 是共享的，所有的写节点写同一个WAL? 
- WAL 中 的数据是否参与 search
- 是否存在定时 flush
- clientA insert vectorA，但是 没有 flush，那么 clientA 能否 search vecotrA，clientB 能否 search  vectorA
- master 是什么节点 ?
- etcd 的 meta 中是否一个 S3 文件一个  meta 记录

---

- read 节点是预先加载数据还是按需加载数据

---
- 两个队列，收数据的队列 处理数据的队列，中间数据丢失的问题？ 第一队列收数据，向第二个队列发数据的过程中的高可用问题

---
- 客户端的数据整体成功或整体失败是如何实现的，两阶段提交并无法100%保证
- TiDB 不是为了替换 mysql，当前的 milvus 设计是包含单机分布式所有的解决方案？
- 第一排队列是多个的，如何保证主键的唯一性 ? (preprocess) 另外一个中心服务提供主键？
- pulsar 或 kafka 都是 jvm 系，运维难度
- 建索引的逻辑?
- 批量数据插入，集体成功，集体失败，如果主键冲突，则自动更新数据
- 主键的唯一性检查的必要性，添加 upsert语义，insert 不做主键检查，upsert做主键检查
- 后台的异步主键查重？
- pulsar 多个客户端负载的延时

---
- 客户端做 reduce? 好主意
- ~~仿照ES，如果insert带id，则主键查重；insert不带id，自动生成id，主键不查重~~

---
写节点需要向所有的读节点广播delete log信息，即当前delete record属于哪个segment