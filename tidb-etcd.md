# TiDB 中 etcd 使用调研
- TiDB 并不单独启一个 etcd 集群
- TiDB 的 etcd 在 pd 内，pd 中包含了一个 etcd 集群，可以直接使用 etcdctl 访问