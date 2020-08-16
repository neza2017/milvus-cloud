# milvus 
milvus 方案设计

---

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
- 暂时不涉及主键唯一性检查

---
  
## 整体框图
```txt
|------------------------------------------| |------------------------------------------|
|               Milvus-Cluster             | |               Milvus-Cluster             |
|  |------------------------------------|  | |  |------------------------------------|  |
|  |             Master                 |  | |  |             Master                 |  |
|  |------------------------------------|  | |  |------------------------------------|  |
|  |----------| |----------| |----------|  | |  |----------| |----------| |----------|  |
|  |  Milvus  | |  Milvus  | |  Milvus  |  | |  |  Milvus  | |  Milvus  | |  Milvus  |  |
|  |----------| |----------| |----------|  | |  |----------| |----------| |----------|  |
|------------------------------------------| |------------------------------------------|

|---------------------------------------------------------------------------------------|
|                                       S3 / etcd                                       |
|---------------------------------------------------------------------------------------|
```
- `Milvus-Cluster` 包含多个 `Miluvs` 节点
- `Milvus-Cluster` 有两种工作模式，`ReadWrite` 和 `ReadOnly`
- `Milvus-Cluster` 启动是需要获得当前集群元数据在 `etcd` 中的前缀，一般格式为： `/<username>`
- `Milvus-Cluster` 有一个全局唯一的`ID`

---

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
- `Load Balance` 模式下，每个 `milvus` 节点的数据完全一致，并且是全量的用户数据
- `Load Balance` 模式下，支持用户无感知的动态扩缩容
- `Reduce` 模式下，每个 `milvus` 节点拥有部分数据，所有节点的数据构成一份全量的用户数据，节点间不会有任何数据备份，如果数据丢失，则直接从 `S3` 加载
- `Reduce` 模式下，不支持用户无感知的动态扩缩容
- 每个 `Milvus` 节点有个 `Milvus-Cluster` 内唯一的节点序号，这个序号从 `1` 开始；如果序号 `1` 的节点意外宕机，则重启的节点依然为序号 `1`
- 扩容的过程中依次增加序号，缩容的过程中，从高序号的节点开始 
- 每个 `Milvus` 对同一个 `Milvus-Cluster` 内的其他节点提供以下服务:
  - 根据 `S3` 文件名，返回对应的数据文件，如果需要的 `S3` 文件不在当前 `Milvus` 节点内，则返回错误码，提示调用者直接从 `S3` 读取
  - 向调用者返回当前 `Milvus` 的工作状态，包括
    - 内存使用率
    - `client`链接数目
  - 心跳信号，每请求一次，心跳信号加1，并向调用者返回最新的心跳信号


---

## `Master` 节点
- `master` 具有简单的转发功能
  - `client` 首先链接 `master`,`master`根据各个`Miluvs`节点的负载状态，选择一个合适的 `Miluvs`节点，并其 `IP` 地址和端口返回给 `client`
  - `client` 收到 `master` 返回的 `Miluvs` 节点 `IP`地址和端口后，主动断开和 `master` 的链接
  - `client` 链接目标的 `Milvus` 节点
  - 因此 `Milus-Cluster` 内的所有 `Milvus` 节点和 `Master` 节点都必须对外暴露自己的IP地址
- `master` 定期的向所有的 `milvus` 节点请求心跳信号，确定 `milvus` 节点是否依然存活

---

## `etcd` 及元数据
- `etcd` 保存全局的 `meta` 信息
- 多个 `Milvus-Cluster` 可以共用一个 `etcd`
- `etcd` 中的 `meta` 类型 
  - `key` 对应 `S3` 里面的文件
    - `key` 的命名格式为: `/<user_name>/<collect_name>/<S3_file_path>`
    - `value` 内容包含
      - 文件类型：索引文件，原始向量文件，标量文件
      - 文件大小，单位为字节
      - 插入该文件的 `Milvus` 节点序号
      - 如果为标量文件，还需要记录对应的列名称
  - `key` 对应属于当前用户的 `collection list`
    - `key` 命名格式为：`/<user_name>/collection_list`
    - `value` 为一个 `array`,每条记录的格式为: `<collect_name> : <create_time>, <index_type>` 按照 `<create_time>` 降序排列
  - `key` 对应属于当前用户的 `Milvus-Cluster` 属性，一个用户可以拥有多个 `Milvus-Cluster`
    - `key` 命名格式为：`/<user_name>/<miluvs_cluster_id>_property`
    - `value` 内容包含
      - `Milvus` 节点数目
      - 每个 `Milvus` 节点的序号，及对应的 `IP` 地址

---

## 数据插入的总体规则
- 数据由哪个 `Milvus` 节点插入，则索引由哪个节点创建
- 采用批处理模式，保证 `flush` 内的操作整理成功或整体失败

---

## 数据两阶段写入 `S3` 的流程
1. `client` 执行 `flush` 操作触发数据写 `S3`
2. `Milvus` 节点将数据 `put` 到 `S3`，假设文件名为 `tmp-s3`
3. 获得 `tmp-s3` 对应 `etcd` 中的 `key`，假设为 `tmp-etcd`
4. 向 `client` 发送 `tmp-etcd` 字符串
5. `client` 向 `milvus` 返回 `tmp-etcd` 字符串确认收到
6. `milvus` 收到 `client`的返回后，再 `etcd` 插入 `tmp-etcd` -> `tmp-s3`
7. 向 `client` 发送 `commit` 操作成功
8. 如果 `Milvus` 超时未收到 `client` 的返回，则删除 `tmp-s3` 文件，切直接切断当前与 `client` 的链接

**注意事项**
`client`有下列3种状态
1. `client` 收到 `commit` 操作成功或失败
2. `client` 未收到任何信息，直接超时，表示当前操作失败
3. `client` 收到 `tmp-etcd` 信息，当时未收到 `commit` 操作成功的信息，此时 `client` 需要重新链接并向 `milvus` 查询 `tmp-etcd` 是否在 `etcd` 中

如果 `tmp-etcd` 对应的 `tmp-s3` 文件被合并导致 `tmp-etcd` 不存在，可能导致 `client` 重新链接时查询 `tmp-etcd` 失败；为了防止这种情况出现，文件合并后，需要在某个地方依然保存 `tmp-etcd`， 确保可以被查询到  

---

## 删除的总体规则
- 不记录 `delete log`，必须在对应的记录的标志位设为删除后才向用户返回，因此删除操作比较耗时

## `Load Balance` 模式下的操作


### 插入数据

### 删除数据

### 查询数据

### 建立索引

### 动态扩容

### 动态缩容

## `Reduce` 模式下的操作

### 动态扩容
- 可以设置一个警戒线，当资源使用率达到警戒线后提示用户需要扩容
- 警戒线只针对扩容操作，没有缩容的警戒线
- 在扩缩容的过程中，用户需要暂停 `milvus-cluster` 的对外服务，向`milvus-cluster` 添加 `milvus` 节点，然后再打开 `milvus-cluster` 的对外服务
- 扩容过程中，原`milvus`节点的内存数据不会丢失，无需重新加载，只有新加入的节点需要加载数据
- 将原 `milvus` 节点中因内存放不下而存到磁盘的数据转移到新加入的 `milvus` 节点，并修改中下面这个配置项目，标记这些数据是由新的节点插入的
  - `key` 的命名格式为: `/<user_name>/<collect_name>/<S3_file_path>`
  - `value` 内容包含
    - 插入该文件的 `Milvus` 节点序号
- 扩容操作，一次只能添加一个节点

### 动态缩容


### `Milvus` 节点宕机重启后需要加载的数据
- 根据 `/<user_name>/collection_list` 从 `etcd` 请求获得当前用户的所有 `collection`
