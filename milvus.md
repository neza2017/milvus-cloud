# milvus 
milvus 方案设计
## 前提假设
- 同一个客户端严格保操作时序，不同客户端不保证时序
    - C<sub>1</sub> 在本地时刻 T<sub>0</sub> 发起插入操作 I<sub>0</sub>，插入数据 D<sub>A</sub>，并执行 `flush` 操作
    - C<sub>1</sub> 在本地时刻T<sub>1</sub> 发起查询操作 Q<sub>1</sub>，数据 D<sub>A</sub>一定对Q<sub>1</sub>可见
    - C<sub>2</sub> 在本地时刻T<sub>2</sub>时刻发起查询操作 Q<sub>2</sub>，数据 D<sub>A</sub>不保证对Q<sub>2</sub>可见
    - 因为 I<sub>0</sub> 与 Q<sub>1</sub> 都是由 C<sub>1</sub> 发起，I<sub>0</sub> 与 Q<sub>1</sub>属于同步操作，所以 Q<sub>1</sub> 命令到达服务端时，I<sub>0</sub>操作已经完成 ，所以D<sub>A</sub>一定对Q<sub>1</sub>可见
    - Q<sub>2</sub> 和  I<sub>0</sub> 属于不同客户端发起的操作,所以不能保证 Q<sub>2</sub> 达到服务器后， I<sub>0</sub> 操作已经完成，数据 D<sub>A</sub>不保证对Q<sub>2</sub>可见
    - 如果 C<sub>1</sub> 和 C<sub>2</sub> 是由同一个用户发起的的链接，并且用户确保 I<sub>0</sub> 操作返回之后，再发起 Q<sub>2</sub> 操作，那么数据 D<sub>A</sub>保证对Q<sub>2</sub>可见
- 未 `flush` 之前，数据对所有客户端均不可见，`flush` 之后数据对所有客户端可见
  - C<sub>1</sub> 在本地时刻 T<sub>0</sub> 发起插入操作 I<sub>0</sub>，插入数据 D<sub>A</sub>，未执行 `flush` 操作
  - C<sub>1</sub> 在本地时刻 T<sub>1</sub> 发起查询操作 Q<sub>1</sub>，数据 D<sub>A</sub>对Q<sub>1</sub>不可见
  - C<sub>1</sub> 在本地时刻 T<sub>2</sub> 发起 `flush` 操作
  - C<sub>1</sub> 在本地时刻 T<sub>3</sub> 发起查询操作 Q<sub>3</sub>，数据 D<sub>A</sub>对Q<sub>3</sub>可见
- 未 `flush` 的操作在客户端断开后均被抛弃
  - C<sub>1</sub> 在本地时刻 T<sub>0</sub> 发起插入操作 I<sub>0</sub>，插入数据 D<sub>A</sub>，未执行 `flush` 操作
  - C<sub>1</sub> 在本地时刻 T<sub>1</sub> 发起删除操作 D<sub>1</sub>，删除数据 D<sub>B</sub>，未执行 `flush` 操作
  - C<sub>1</sub> 在本地时刻 T<sub>2</sub> 与服务端断开链接
  - 数据 D<sub>A</sub> 在服务器端被抛弃，数据 D<sub>B</sub>在服务端未被删除，依然可以被查询到
- `Insert` 和 `Delete` 操作需要 `flush`，其它不需要,如`create table`,`drop table`
- 不涉及用户管理，及访问控制，访问控制由 `IP` 地址白名单实现
  
## 整体框图
```txt
|---------------------------------------------------------------------------------------|
|                                          etcd                                         |
|---------------------------------------------------------------------------------------|

|------------------------------------------| |------------------------------------------|
|               Milvus-Cluster             | |               Milvus-Cluster             |
|  |----------| |----------| |----------|  | |  |----------| |----------| |----------|  |
|  |  Milvus  | |  Milvus  | |  Milvus  |  | |  |  Milvus  | |  Milvus  | |  Milvus  |  |
|  |----------| |----------| |----------|  | |  |----------| |----------| |----------|  |
|------------------------------------------| |------------------------------------------|

|---------------------------------------------------------------------------------------|
|                                          S3                                           |
|---------------------------------------------------------------------------------------|
```
- `Milvus-Cluster` 包含多个 `Miluvs` 节点
- `Milvus-Cluster` 有两种工作模式，`ReadWrite` 和 `ReadOnly`
- `Milvus-Cluster` 启动是需要获得当前集群元数据在 `etcd` 中的前缀，一般格式为： `/<username>`


## Milvus 内部框图
```txt
|-----------------------------|
|          Miluvs             |
|  |-----------------------|  |
|  |  Reduce / LoadBalance |  |
|  |----------|------------|  |
|  | Write    |  Reade     |  |
|  |----------|------------|  |
|-----------------------------|
```
- `Milvus` 节点有两种工作模式 `Reduce` 和 `Load Balance`
- `Reduce` 模式针对数据量过大，单节点无法处理
- `Load Balance` 针对高并发应用
- `Milvus-Cluster` 内部的所有 `Milvus` 节点工作模式必须一致，即所有节点必须全为`Reduer`模式或全为`Load Balance`模式，不能部分节点为`Reduce`模式，部分节点为`Load Balance`模式

## `etcd` 及元数据
- `etcd` 保存全局的 `meta` 信息
- 多个 `Milvus-Cluster` 可以共用一个 `etcd`
- `etcd` 的 `meta` 分两大类:
  - 一个 `key` 对应 `S3` 里面的一个文件
    - `key` 的命名格式为: `/<user_name>/<collect_name>/<S3_file_path>`
    - `value` 内容包含
      - 文件类型：索引文件，原始向量文件，标量文件
      - 文件大小，单位为字节
      - 如果为标量文件，还需要记录对应的列名称
  - 一个 `key` 对应一个属于当前用户的 `collection list`
    - `key` 命名格式为：`/<user_name>/collection_list`
    - `value` 为一个 `array`,每条记录的格式为: `<collect_name> : <index_type>`

 
## 插入操作

