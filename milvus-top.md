# milvus 设计方案
- 使用共享存储，计算与存储分离

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

---

## 配置文件
- `auto_flush`:
  - 类型 : `bool`
  - 默认值 : `false`
  - 无论是否设置 `auto_flush` , `milvus` 程序保证 `flush` 间的数据要么整体成功，要么整体失败
  - 当 `auto_flush` 为 `false` 时，`client` 端需要手动调用 `flush`
  - 当 `auto_flush` 为 `true` 时，以下两种情况均可以触发 `flush`
    - 定时超过 `flush_interval` 后触发
    - 插入的数据超过 `flush_limitition` 后触发
  - 当 `auto_flush` 为 `true` 时，如果自动触发的 `flush` 操作运行失败，则数据丢失，不向用户返回任何信息，仅仅记录日志 
- `flush_interval`
  - 类型 : `int32`
  - 默认值 ： `1000`
  - 定时触发 `flush` 的时间间隔，单位为毫秒
- `flush_limitition`
  - 类型 : `string`
  - 默认值 : `16M`
  - 插入的数据超过 `flush_limitition` 后触发 `flush` 操作
- `num_replicas`:
  - 类型 : `int32`
  - 默认值 : `0`
  - 数据在不同节点间备份的数目,`0` 表示数据没有备份
  - 数据只能在不同的节点间备份，因此该值的上限为 `num_nodes-1`
- `num_nodes`:
  - 类型 : `int32`
  - 默认值 : `1`
  - 负责数据插入及查询节点的数目
- `sync_level`:
  - 类型 : `int32`
  - 默认值  : `0`
  - 同步等级，插入或删除操作在 `Milvus-Cluster` 上产生 `sync_level` 个 `replica` 后才向 `client` 返回，因此该值的上限为 `num_replicas`

---

  
## 整体框图
```txt
+------------------------------------------+ +------------------------------------------+
|               Milvus-Cluster             | |               Milvus-Cluster             |
|  +------------------------------------+  | |  +------------------------------------+  |
|  |             Master                 |  | |  |             Master                 |  |
|  +------------------------------------+  | |  +------------------------------------+  |
|  +----------+ +----------+ +----------+  | |  +----------+ +----------+ +----------+  |
|  |  Milvus  | |  Milvus  | |  Milvus  |  | |  |  Milvus  | |  Milvus  | |  Milvus  |  |
|  +----------+ +----------+ +----------+  | |  +----------+ +----------+ +----------+  |
+------------------------------------------+ +------------------------------------------+
+---------------------------------------------------------------------------------------+
|                                       S3,etcd                                         |
+---------------------------------------------------------------------------------------+
```
- `Milvus-Cluster` 包含多个 `Miluvs` 节点
- `Milvus-Cluster` 启动是需要获得当前集群元数据在 `etcd` 中的前缀，一般格式为： `/<username>`
- `Milvus-Cluster` 有一个全局唯一的`ID`
- 底层存储不一定是 `S3`，只要是共享存储即可， `etcd` 中 `<S3_file_path>` 改为对应的共享存储路径即可

---

## `Master` 节点
- `Master` 是无状态的服务
- `Master` 具有简单的转发功能
  - `client` 首先链接 `Master`,`Master`根据各个`Miluvs`节点的负载状态，选择一个合适的 `Miluvs`节点，并其 `IP` 地址和端口返回给 `client`
  - `client` 收到 `Master` 返回的 `Miluvs` 节点 `IP`地址和端口后，主动断开和 `Master` 的链接
  - `client` 链接目标的 `Milvus` 节点
  - 因此 `Milus-Cluster` 内的所有 `Milvus` 节点和 `Master` 节点都必须对外暴露自己的IP地址
- `Master` 定期的向所有的 `milvus` 节点请求心跳信号，确定 `milvus` 节点是否依然存活
- `Milvus` 节点可以向 `Master` 节点请求其他节点的负载状态；该功能在 `Reduce`模式的插入操作中使用，如果当前`Milvus`节点的内存超过警戒线，则`Milvus` 节点向 `Master` 节点请求其它`Miluvs`节点的负载状态，然后选择一个选择一个低负载的节点，将插入数据转发到那个节点
- `Master` 需要 `watch etcd` 的 这个 `/<user_name>/<miluvs_cluster_id>_property`，当有新的节点加入后会更新这个 `key`

---

## `etcd` 及元数据
- `etcd` 保存全局的 `meta` 信息
- 多个 `Milvus-Cluster` 可以共用一个 `etcd`
- `etcd` 中的 `meta` 类型
  - 用户列表
    - `key` 为 `/user_list`
    - `value` 为一个 `array`，每条记录包含一个`<user_name>`
  - `key` 对应 `S3` 的数据文件
    - `key` 的命名格式为: `/<user_name>/<collection_name>/data/<S3_file_path>`
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
    - `value` 为一个 `array`,每条记录的格式为: `<collection_name> : <create_time>, <index_type>` 按照 `<create_time>` 降序排列
  - `key` 对应属于当前用户的 `Milvus-Cluster` 属性，一个用户可以拥有多个 `Milvus-Cluster`
    - `key` 命名格式为：`/<user_name>/config/<Milvus-Cluster_ID>`
    - `value` 内容为 `json` 格式存储的 `Milvus-Cluster` 配置文件
## 插入操作
