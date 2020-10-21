# Milvus cloud 部署设计方案

## 总体原则
1. 为每个用户创建各自的 aws 实例，用户间不共享 aws 实例
2. 创建 aws 实例需要提供两个参数: 实例类型、磁盘大小； cpu、gpu、memory 由实例类型决定
3. 不支持动态更改 aws 实例类型
4. 直接对用户暴露 aws 实例
5. 用户必须首先设置访问白名单，才能访问 aws 上的 miluvs 实例
6. miluvs 客户端不需要做任何修改

## 创建 miluvs 实例
`request`
```json
{
    "session-token" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "username" : "milvus-test",
    "milvus-name": "search-cats",
    "milvus-type" : "M5",
    "disk-size" : 50
}
```

`response` on success
```json
{
    "status" : "success",
    "milvs-name" : "search-cats",
    "milvus-url" : "ec2-52-82-52-246.cn-northwest-1.compute.amazonaws.com.cn",
    "milvus-port" : "19530",
    "milvus-em-port" : "19531"
}
```

`response` on failed
```json
{
    "status" : "failed",
    "description" : "can't create aws instances"
}
```

mongodb 中存储的数据
```json
{
    "username" : "milvus-test",
    "milvus-name" : "search-cats",
    "milvus-url" : "ec2-52-82-52-246.cn-northwest-1.compute.amazonaws.com.cn",
    "milvus-port" : "19530",
    "milvus-em-port" : "19531",
    "milvus-type" : "M5",
    "subnet-id" : "subnet-64b1090d",
    "security-groups" : [
        {
            "group-name" : "sg-base",
        },
        {
            "group-name" : "sg-miluvs-test"
        }
    ],
    "instances" :[
        {
            "node-type" : "master",
            "instance-id" : "i-00879b91cf475a597",
            "key-pair" : "zilliz-hz02",
            "public-ip" : "52.82.52.246",
            "private-ip" : "172.31.9.114",
        },
        {
            "node-type" : "worker",
            "instance-id" : "i-0102e93a4413df1ba",
            "key-pair" : "zilliz-hz02",
            "public-ip" : "3.12.47.200",
            "private-ip" : "172.31.9.115",
        }
    ]
}
```

后台工作：
1. 创建 aws ec2 实例
2. 每个 aws ec2 实例和 disk 至少包含 3 个 tag : "username", "milvus-name", "miluse-username"
3. miluvs 集群的所有机器使用同一个 "subnet-id" 和 "security-groups"
4. "sg-base" 内部包含基础的安全规则，比如允许控制台机器远程登录
5. "sg-miluvs-test" 内部放置用户设置的，允许方位 miluvs 实例的机器 ip 地址白名单


## 设置白名单
白名单的规则放置在 "sg-miluvs-test" 内
`request`
```json
{
    "session-token" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "username" : "milvus-test",
    "milvus-name": "search-cats",
    "ip-ranges" : [
        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-port"
        }

        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-em-port"
        }
    ]
}
```
"port-type" 设置向用户开放的端口，可选值为 "miluvs-port", "milvus-em-port"

`response` on success
```json
{
    "status" : "success",
}
```

`response` on faied
```json
{
    "status" : "failed",
   "description" : "unsported port-type"
}
```