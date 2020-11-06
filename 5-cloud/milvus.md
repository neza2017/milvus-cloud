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
- 这个设计针对批插入，并且删除属于低频操作
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
- 底层存储不一定是 `S3`，只要是共享存储即可， `etcd` 中 `<S3_file_path>` 改为对应的共享存储路径即可

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
- 以 `etcd` 的 `revison` 为 `key`，在 `Milvus`的内存中记录多个版本的 `meta` 信息
- `Milvus` 节点具备转发功能，可以把`插入`、`删除`、`查询` 转发到同一个 `Milvus-Cluster` 的其它 `Miluvs` 节点上

---

## `Master` 节点
- `master` 是无状态的服务
- `master` 具有简单的转发功能
  - `client` 首先链接 `master`,`master`根据各个`Miluvs`节点的负载状态，选择一个合适的 `Miluvs`节点，并其 `IP` 地址和端口返回给 `client`
  - `client` 收到 `master` 返回的 `Miluvs` 节点 `IP`地址和端口后，主动断开和 `master` 的链接
  - `client` 链接目标的 `Milvus` 节点
  - 因此 `Milus-Cluster` 内的所有 `Milvus` 节点和 `Master` 节点都必须对外暴露自己的IP地址
- `master` 定期的向所有的 `milvus` 节点请求心跳信号，确定 `milvus` 节点是否依然存活
- `Milvus` 节点可以向 `master` 节点请求其他节点的负载状态；该功能在 `Reduce`模式的插入操作中使用，如果当前`Milvus`节点的内存超过警戒线，则`Milvus` 节点向 `master` 节点请求其它`Miluvs`节点的负载状态，然后选择一个选择一个低负载的节点，将插入数据转发到那个节点
- `master` 需要 `watch etcd` 的 这个 `/<user_name>/<miluvs_cluster_id>_property`，当有新的节点加入后会更新这个 `key`

---

## `etcd` 及元数据
- `etcd` 保存全局的 `meta` 信息
- 多个 `Milvus-Cluster` 可以共用一个 `etcd`
- `etcd` 中的 `meta` 类型
  - 用户列表
    - `key` 为 `/user_list`
    - `value` 为一个 `array`，每条记录包含一个`<user_name>`
  - `key` 对应 `S3` 的数据文件
    - `key` 的命名格式为: `/<user_name>/<collect_name>/data/<S3_file_path>`
    - `value` 内容包含
      - 文件类型：索引文件，原始向量文件，标量文件,`delete log`
      - 文件大小，单位为字节
      - 插入该文件的 `Milvus` 节点序号
      - 如果为标量文件，还需记录以下内容
        - 对应的列名
        - 当前标量文件对应的向量文件名
      - 如果为索引文件，则需要记录所对应的原始向量文件列表，索引文件可以由多个向量文件构成
  - `key` 对应属于当前用户的 `collection list`
    - `key` 命名格式为：`/<user_name>/collection_list`
    - `value` 为一个 `array`,每条记录的格式为: `<collect_name> : <create_time>, <index_type>` 按照 `<create_time>` 降序排列
  - `key` 对应属于当前用户的 `Milvus-Cluster` 属性，一个用户可以拥有多个 `Milvus-Cluster`
    - `key` 命名格式为：`/<user_name>/<miluvs_cluster_id>_property`
    - `value` 内容包含
      - `Milvus` 节点数目
      - 每个 `Milvus` 节点的序号，及对应的 `IP` 地址
      - `Master` 节点 `IP` 地址

---

## 数据插入的通用规则
- 数据插入到哪个节点，则由哪个节点负责将数据写入 `S3`，不存在数据首先写入 `milvus-1` ，然后由 `milvus-2` 节点从`milvus-1`节点复制数据，再将数据写入 `S3`
- 数据由哪个 `Milvus` 节点插入，则索引文件由哪个节点创建
- 采用批处理模式，保证 `flush` 内的操作整理成功或整体失败
- 插入请求中包含当前插入的数据量，以 `Byte` 计算

---

## 数据两阶段插入流程
- `client` 执行 `flush` 操作触发数据写 `S3`
- `Milvus` 节点将数据 `put` 到 `S3`，假设文件名为 `tmp-s3`
- 获得 `tmp-s3` 对应 `etcd` 中的 `key`，假设为 `tmp-etcd`
- 向 `client` 发送 `tmp-etcd` 字符串
- `client` 向 `milvus` 返回 `tmp-etcd` 字符串确认收到
- `milvus` 收到 `client`的返回后，再 `etcd` 插入 `tmp-etcd` -> `tmp-s3`
- 向 `client` 发送 `commit` 操作成功
- 如果 `Milvus` 超时未收到 `client` 的返回，则删除 `tmp-s3` 文件，切直接切断当前与 `client` 的链接

**注意事项**
`client`有下列3种状态
- `client` 收到 `commit` 操作成功或失败
- `client` 未收到任何信息，直接超时，表示当前操作失败
- `client` 收到 `tmp-etcd` 信息，当时未收到 `commit` 操作成功的信息，此时 `client` 需要重新链接并向 `milvus` 查询 `tmp-etcd` 是否在 `etcd` 中

如果 `tmp-etcd` 对应的 `tmp-s3` 文件被合并导致 `tmp-etcd` 不存在，可能导致 `client` 重新链接时查询 `tmp-etcd` 失败；为了防止这种情况出现，文件合并后，需要在某个地方依然保存 `tmp-etcd`， 确保可以被查询到  

---

## 数据删除的通用规则
- 删除操作并不直接修改文件，而是产生一条删除记录 `delete log`， 每条记录包含以下内容:
  - collection_name
  - 向量 `ID`
  - 向量所在的 `S3` 文件名
  - 向量被删除向量的行号索引，从0 开始
- 单次 `flush` 内的所有删除操作，产生一个 `delete log` 文件
- 删除操作为阻塞操作，需要获得 `delete log` 的需要的完整信息后才能向 `client` 返回，这里涉及到对比所有向量的 `ID`，所以删除操作是一个耗时的操作

---

## 查询的通用规则
- `Milvus` 节点收到查询指令后，首先从 `etcd` 获得当前的 `revision`，记为 `etcd-revisoin`
- 对 `etcd` 的每次修改都会导致 `revision` 值加1
- 在 `Milvus` 节点的内存中查询，是否有 `etcd-revision` 对应的 `meta` 信息，如果没有则向 `etcd` 请求 `etcd-revisoin` 对应的 `meta` 信息
- 节点根据 `meta` 信息，能够得出本次查询需要用到哪些文件，以及哪些文件在本节点内
- `Milvus` 节点首先在内存中寻找需要的文件，如果没有，则到本节点的磁盘查找，如果依然没有，则到 `S3` 加载

---

## 文件合并的通用规则
- 零散的向量文件和 `delete log` 文件可以合并
- 向量文件只针对已经建立索引的文件，原始向量文件没必要合并
- 合并操作在程序后台启动
- 合并操作的所有原始文件必须在同一个 `Milvus` 节点内，不能跨节点合并
- 考虑到多版本的问题，某些原始文件可能正在被某个查询使用，所以合并操作不删除原始文件，而是产生一个新的文件
- 合并操作采用事务的方式一次性更新`etcd`中的`meta`信息，因此所有这些被跟新的记录，`revision`值是一样的
- 因为每次查询均获得 `etcd` 的 `revision`值，所以只要不删除原始文件， 可以不用担心更新后的 `etcd meta` 对正在执行的查询有任何影响 
- `Milvus` 节点上有个后台程序，定期检查哪些文件是合并了，并且可以删除的
- 删除合并后的原始文件后，需要对 `etcd` 做 `compact` 操作，删除没必要保留的 `revision`

---

## 如何保证数据的一致性?
- 本设计方案只能确保插入或删除操作更新 `meta` 后一定能被后续后续操作查询到，不管查询操作是否与插入或删除操作来自同一个客户端
- 因为 `Milvus` 节点收到查询指令后，会先从 `etcd` 获取当前最新的 `revision`，然后获得这个 `revision` 需要的所有 `meta` 信息，所以只要更新了 `meta`， 一定能被后续操作查询到，不管这些操作是否来自同一个客户端

---

## 如何在 `Reduce` 模式提高 QPS
```txt
|------------------------------------------------------------------|
|                           Milvus-Proxy                           |
|------------------------------------------------------------------|
|                                                                  |
|  |------------------| |------------------| |------------------|  |
|  |  Milvus-Cluster  | |  Milvus-Cluster  | |  Milvus-Cluster  |  |
|  |   (ReadWrite)    | |    (ReadOnly)    | |    (ReadOnly)    |  |
|  |------------------| |------------------| |------------------|  |
|                                                                  |
|------------------------------------------------------------------|

|------------------------------------------------------------------|
|                            S3 / etcd                             |
|------------------------------------------------------------------|

```
- 多个 `Milvus-Cluster` 组成一个更大的集群
- 集群中只有一个 `Milvus-Cluster` 为`ReaderWrite` 模式，其它 `Milvus-Cluster` 为 `ReadOnly` 模式

---

## `Load Balance` 模式下的操作
- `Load Balance` 模式下，每个 `Milvus` 节点的数据完全一致，并且是全量的用户数据
- `Load Balance` 针对提高`QPS`设计
- `Load Balance` 模式下，支持用户无感知的动态扩缩容
- 
---

### 插入数据
- 所有`Milvus`节点 `watch etcd`
- 实施插入操作的节点
  - 按照`两阶段插入流程`，将数据写入`S3`，同时更新 `meta` 信息，一次完整的插入，对应一个文件
  - 跟新 `meta` 后需要跟新本地的 `etcd revision` 值
  - 如果插入数据导致内存超警戒线， 则按照 文件最近一次使用时间，`collect` 创建时间，文件创建时间三级排序，将历史文件写入磁盘
  - 如果磁盘也放不下历史文件，则依然按照  文件最近一次使用时间, `collect` 创建时间，文件创建时间三级排序，丢弃历史文件
- 非实施插入操作的节点
  - 因为`watch etcd`，当数据写入 `S3` 并更新 `meta` 后，节点能够得到刚刚插入数据的 `meta` 信息
  - 根据 `meta` 信心，可以得知是哪个节点实施了插入操作，以及插入的文件名
  - 节点向实施插入操作的节点请求刚刚被插入的文件
  - 更新节点本地的 `etcd revision` 值

### 删除数据
- 删除操作可以看成另一个模式的插入操作，只是插入的数据从向量文件变成`delete log`文件

### 查询数据
- 结合`查询的通用规则`，`Load Balance` 模式下每个节点拥有全量数据，所以查询只在单个节点上实施

### 建立索引
- 建立索引操作可以看成另一个模式的插入操作，只是插入的数据从原始向量文件变成索引文件
- 和插入数据有一点不同，建立索引是由 `Milvus` 节点自己发起的，不存在 `client` ，所以没有两阶段提交，只需要更新 `meta`
- 建立索引操作与合并向量文件的操作可以一同完成
  - 多个小的原始向量文件，在内存中合并生成一个新的大的原始向量文件
  - 在这个大的原始向量文件上创建索引，并写入 `S3`
  - 使用事务的方式一次性更新 `meta` ，包括添加新的索引文件的 `meta`，删除小文件的 `meta`
  - 合并过程中只删除小文件在 `etcd` 中的 `meta`，不删除 `S3` 上的小文件数据，这些数据可能正在被某一次查询使用
- 只针对当前  `Milvus` 节点自己的数据建立索引，不存在跨节点建索引的情况
- 因为 `Load Balance` 模式下每个节点都拥有全量数据，那么原始向量文件 D<sub>A</sub> 在每个节点上都存在，为了防止多个 `Milvus` 节点对同一封数据创建索引，所以原始向量文件 D<sub>A</sub> 由那个节点负责插入的，则由那个节点负责建立索引

### 动态扩容
- `Load Balance` 模式下支持用户无感知的动态扩容
- 一次只能扩容新增一个节点，在本次扩容操作完成前，不能再次请求扩容操作
- 因为 `Load Balance` 模式下，每个 `Milvus` 节点的数据完全一致，并且每个节点对外提供请求数据的服务，所以新加入的节点只需要向已经存在的节点请求数据, 无需从  `S3` 获取
- 新加入的节点更新 `etcd` 的 `/<user_name>/<miluvs_cluster_id>_property`，并且 `watch etcd` 后视为本次扩容操作结束
- 如果数据 D<sub>A</sub> 在新节点 更新 `/<user_name>/<miluvs_cluster_id>_property` 后, `watch etcd` 前插入，如何保证数据 D<sub>A</sub> 在新加入节点上可见？ 因为每次查询需要从 `etcd` 获得最新的 `revision` 值，根据 `revision` 获得 `meta`，所以数据 D<sub>A</sub>只要更新了 `meta` 就一定能被查询到
- 因为 `master` 会监视  `/<user_name>/<miluvs_cluster_id>_property`，所以新加入的节点更新这个  `key` 后, `master` 可以感知到，让后会向改节点请求状态信息，并根据状态信息向改节点抓发请求任务

### `Milvus` 节点宕机重启后需要加载的数据
- 宕机重启后的操作与动态扩容的操作基本一致，注意以下几点：
  - 宕机重启后的节点无需更新 `etcd` 的 `/<user_name>/<miluvs_cluster_id>_property`
  - 宕机重启的节点在数据加载完成前不对 `master`  的状态请求做任何回应

---

## `Reduce` 模式下的操作
- `Reduce` 模式下，每个 `milvus` 节点拥有部分数据，所有节点的数据构成一份全量的用户数据，节点间不会有任何数据备份，如果数据丢失，则直接从 `S3` 加载
- `Reduce` 模式下，不支持用户无感知的动态扩容，扩容过程中需要暂停服务器，但是已经存在节点的内存数据不会被删除，重新提供服务后依然可用

---

### 插入数据
- 与　`Load Balance` 不同，`Reduce` 模式下，每个节点的数据互不相同，所有节点的数据构成一份全量的用户数据，所以插入节点数据更新 `etcd meta` 后无需通知其他节点从 `etcd` 获取刚刚更新的 `meta` 并获得刚刚插入的数据文件，因为一份数据文件只在一个 `Milvus` 节点上
- 按照`两阶段插入流程`，将数据写入`S3`，同时更新 `meta` 信息，一次完整的插入，对应一个文件
- 跟新 `meta` 后需要跟新本地的 `etcd revision` 值
- 如果当前节点的内存使用量已经查过警戒线，则向 `master` 节点发送请求，查询哪个节点有富余的内存,则将插入请求转发到对应的节点上
- 如果插入请求转发到其他节点，则对应的 `meta` 以及后续创建索引文件均由那个节点负责，与当前节点无关
- 如果没有内存富余的节点，则按照文件最近一次使用时间，`collect` 创建时间，文件创建时间三级排序，将历史文件写入磁盘
- 如果磁盘也放不下历史文件，则依然按照  文件最近一次使用时间, `collect` 创建时间，文件创建时间三级排序，丢弃历史文件
  
---

### 删除数据
- 与 `Load Balance` 不同，`Reduce` 模式下，节点收到删除数据的操作后，会把 删除指令转发到所有其它 `Milvus` 节点上，因为需要删除的数据不一定在当前节点上
- 每个 `Milvus` 节点检查的内存及磁盘上是否有对应的数据，如果有，则删除；否则返回空
- 如果每个 `Milvus` 节点的内存及磁盘均不存在对应的数据，则说明需要删除的数据在 `S3` 存储上，则由最初收到删除请求的 `Milvus` 节点负责到 `S3` 上搜索数据并删除
- 如果 `S3` 上也不存在，则说明改数据不存在，直接返回错误
- 找到序号删除的记录后，删除操作剩下的动作可以类比一个插入操作，插入一个 `delete log` 文件

---

### 查询数据
- 与　`Load Balance` 不同，`Reduce` 模式下，每个节点的数据互不相同，所有节点的数据构成一份全量的用户数据，所以 `Milvus` 节点会把查询指令转发到所有的其他节点上
- 收到查询指令后，`Milvus` 节点首先从 `etcd` 获得 `revision` 值，然后查询指令附带 `revision` 值一起转发到其它节点上
- 其它节点收到转发来的查询操作后，直接使用请求内附带的 `revision` 值
- 哪个节点转发的查询指令，则由哪个节点负责最后数据的 `Reduce` 操作

---

### 建立索引
- 建立索引的方法与 `Load Balance` 一样，每个节点只针对自己插入的数据建立索引，并且建立索引的过程中可以合并小的原始向量文件

---

### 扩容操作
- 可以设置一个警戒线，当资源使用率达到警戒线后提示用户需要扩容
- 警戒线只针对扩容操作，没有缩容的警戒线
- 在扩缩容的过程中，用户需要暂停 `milvus-cluster` 的对外服务，向`milvus-cluster` 添加 `milvus` 节点，然后再打开 `milvus-cluster` 的对外服务
- 扩容过程中，原`milvus`节点的内存数据不会丢失，无需重新加载，只有新加入的节点需要加载数据
- 将原 `milvus` 节点中因内存放不下而存到磁盘的数据转移到新加入的 `milvus` 节点，并修改中下面这个配置项目，标记这些数据是由新的节点插入的
  - `key` 的命名格式为: `/<user_name>/<collect_name>/<S3_file_path>`
  - `value` 内容包含
    - 插入该文件的 `Milvus` 节点序号
- 扩容操作，一次只能添加一个节点
- 新加入的节点更新 `etcd` 的 `/<user_name>/<miluvs_cluster_id>_property` 后视为本次扩容操作结束

---

### `Milvus` 节点宕机重启后需要加载的数据
- `Reduce` 模式下，如果发生宕机，则暂时无法对外提供服务，等节点回复后才能对外提供服务
- 根据 `/<user_name>/collection_list` 从 `etcd` 请求获得当前用户的所有 `collection`
- 使用前缀查询的方式，`/<user_name>/<collect_name>/<S3_file_path>`，所有的 `S3` 文件名
- 根据文件内记录的插入该文件的 `Milvus` 节点序号，从 `S3` 读取属于该节点的所有文件，如果内存放不小，则 `collection` 创建的时间顺序，文件创建的时间顺序两级排序，优先将近期的文件放到内存中，其余存到磁盘中，直到磁盘放不小