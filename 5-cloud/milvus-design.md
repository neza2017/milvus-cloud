# milvus 设计方案
- proxy : 直接与client链接的最前端队列
- I&D : proxy 的数据进入 I&D 队列

## proxy
- 直接将 插入删除操作 哈希到对应的队里中
- 将查询插入 静态的(系统启动后预生成) 查询队列，插入过程中直接指定当前结果的放回的 channel 名称
- 将每个操作打时标 timestamp
- 为每个操作赋值一个 `UUID`
- 当前查询语句的合法性检查
- 如果当前插入没有主键，则自动生成主键,从 `master` 出获取一段 key 号
- 删除操作进两个队列
    - 全局的删除的队列
    - 删除 ID 对应的 hash 队列
- 定时向 time_sync 队列发送消息，内容包含 proxy ID 和 时间

## client
- client 只与 `proxy` 链接，不与 `master` 链接


## reader
- 查询的结果信息需要包含当前 reader 节点负责的 channel 序列
- 针对删除数据，读节点需要额外监听全局的删除的队列
- 监听 time_sync 队列，用来推进可以安全执行的 request
- 监听 control 队列，获得 segment 名称及范围定义，segment 消息格式 
    - segment name
    - channel range
    - timestamp range，如果未指定区间右键，则为Segment Open，否则为  Semgment Close
    - collect/parttion/segment group

## writer
- 监听指定 channel
- 针对删除数据，写节点只负责监听自己负责的队列
- 监听 time_sync 队列，只有收到 time_sync 信号后，才将之前数据写入 S3
- 上线后先从 pulsar 那最近一条记录，根据这个这记录查询 S3 是否有这个记录，如果没有，则继续往前查
- 监听 control 队列，获得 segment 名称及范围定义，segment 消息格式 
    - segment name
    - channel range
    - timestamp range，如果未指定区间右键，则为Segment Open，否则为  Semgment Close
    - collect/parttion/segment group
- 

## master
- 能够分配全局 ID,考虑替换 `UUID`
- 只有 `master` 与 `etcd` 交互，其它节点不与 `etcd` 有任何交互
- 监听 time_sync 队列，确保 proxy 是否存活
- 

## S3 文件名格式
- collect_key_time_segment

## meta 存储格式

## 全局队列
- time_sync

