# 从 `ElasticSearch` 得到的借鉴
- 每个节点内部都有一张路由表，知道每次查询需要的数据分别在那些节点上
- 节点内部路由表的更新频次决定了 `CAP` 中个 `C`，如果用户对 `数据一致性` 要求特别高，那么每次请求，都需要从 `etcd` 获得对应 `revision` 的最新 `meta`； 如果用户对 `数据一致性` 要求不高，那么可以由 `etcd` 的 `watch` 机制更新 `meta` 自身节点的 `meta`

## 参考资料
- https://leonlibraries.github.io/2017/04/20/ElasticSearch%E5%86%85%E9%83%A8%E6%9C%BA%E5%88%B6%E6%B5%85%E6%9E%90%E4%BA%8C/