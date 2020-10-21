# milvus 基础操作
## `S3` 为存储介质
以共享内存的方式实现 `HA`

### 插入流程
- 一个客户端只能在一个 `milvus` 实例上做插入操作，但是可以在多个 `milvus` 实例上做查询操作

### 数据写入 `S3` 流程
1. `client` 执行 `commit` 操作触发数据写 `S3`
2. 数据 `put` 到 `S3`，文件名为 `tmp-s3`
3. 获得 `tmp-s3` 对应 `etcd` 中的 `key`，假设为 `tmp-etcd`
4. 向 `client` 发送 `tmp-etcd` 字符串
5. `client` 向 `milvus` 返回 `tmp-etcd` 字符串确认收到
6. `milvus` 收到 `client`的返回后，再 `etcd` 插入 `tmp-etcd`-`tmp-s3`
7. 向 `client` 发送 `commit` 操作成功
8. 如果 `milvus` 超时未收到 `client` 的返回，则删除 `tmp-s3` 文件，切直接切断当前与 `client` 的链接

**注意事项**
`client`有下列三中状态
1. `client` 收到 `commit` 操作成功或失败
2. `client` 未收到任何信息，直接超时，表示当前操作失败
3. `client` 收到 `tmp-etcd` 信息，当时未收到 `commit` 操作成功的信息，此时 `client` 需要重新链接并向 `milvus` 查询 `tmp-etcd` 是否在 `etcd` 中

如果 `tmp-etcd` 对应的 `tmp-s3` 文件被合并导致 `tmp-etcd` 不存在，可能导致 `client` 重新链接时查询 `tmp-etcd` 失败；为了防止这种情况出现，文件合并后，需要在某个地方依然保存 `tmp-etcd`， 确保可以被查询到  


### 查询流程
- 收到查询指令后，先获取 `etcd` 当前的 `revision`
- 对比 `milvus meta` 缓存的 `revision` 与 `etcd` 的 `revision` 是否一致，如果落后，使用 `etcd revision` 更新 `milvus meta` 缓存
- 以 `collection` 为单位更新 `milvus meata` 缓存


---

## 以 `Disk` 为存储介质
以主备复制的方式实现 `HA`