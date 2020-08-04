# TiDB 中 etcd 使用调研
- TiDB 并不单独启一个 etcd 集群
- TiDB 的 etcd 在 pd 内，pd 中包含了一个 etcd 集群，可以直接使用 etcdctl 访问

## 启动 `TiDB`
使用 `docker-compose` 启动 `TiDB`
```bash
git clone https://github.com/pingcap/tidb-docker-compose.git
cd tidb-docker-compose
docker-compose up -d
```

## 获取 `PD` 的 `ip` 地址
```bash
$ docker exec tidb-docker-compose_pd0_1 cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
192.168.96.6	cb4a05d3cd6b
$ docker exec tidb-docker-compose_pd2_1 cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
192.168.96.2	80d7f7a1839f
$ docker exec tidb-docker-compose_pd1_1 cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
192.168.96.7	a6fe52fce3d8
```

## 使用 `etcd` 访问 `PD`
```bash
$ ETCDCTL_API=3 ./etcdctl --endpoints=192.168.96.6:2379,192.168.96.2:2379,192.168.96.7:2379 member list      
23a875b0ed30f56f, started, pd0, http://pd0:2380, http://pd0:2379, false
47149ed249f68e2b, started, pd1, http://pd1:2380, http://pd1:2379, false
545874f77ee42365, started, pd2, http://pd2:2380, http://pd2:2379, false
$ ETCDCTL_API=3 ./etcdctl --endpoints=192.168.96.6:2379,192.168.96.2:2379,192.168.96.7:2379 endpoint health
192.168.96.7:2379 is healthy: successfully committed proposal: took = 3.89298ms
192.168.96.6:2379 is healthy: successfully committed proposal: took = 3.475542ms
192.168.96.2:2379 is healthy: successfully committed proposal: took = 3.822029ms
```

## 列出 `PD` 中的所有 `key`
```bash
ETCDCTL_API=3 ./etcdctl --endpoints=192.168.96.6:2379,192.168.96.2:2379,192.168.96.7:2379 get '' --prefix
```

## 监视 `PD` 中所有的 `key`
```bash
ETCDCTL_API=3 ./etcdctl --endpoints=192.168.96.6:2379,192.168.96.2:2379,192.168.96.7:2379 watch '' --prefix
```
可以发现 `timestamp` 及 `ttl` 定时更新

## 插入数据
参考 :https://docs.pingcap.com/tidb/stable/import-example-data

使用 `mysql` 客户端链接 `TiDB`
```bash
mysql --host 127.0.0.1 --port 4000 -u root
```

### 创建数据库
```sql
CREATE DATABASE bikeshare;
```
在 `watch` 界面可以发现新增 `key`
```txt
PUT
/tidb/ddl/all_schema_versions/69238bc6-b752-4781-9e4f-7fd143f630a5
23
```

### 使用数据库
```sql
USE bikeshare;
```
该操作不增加任何 `key`

### 创建表格
```sql
CREATE TABLE trips (
 trip_id bigint NOT NULL PRIMARY KEY AUTO_INCREMENT,
 duration integer not null,
 start_date datetime,
 end_date datetime,
 start_station_number integer,
 start_station varchar(255),
 end_station_number integer,
 end_station varchar(255),
 bike_number varchar(255),
 member_type varchar(255)
);
```
在 `watch` 界面可以发现新增 `key`
```txt
PUT
/tidb/ddl/all_schema_versions/69238bc6-b752-4781-9e4f-7fd143f630a5
28
```

### 插入数据
```sql
LOAD DATA LOCAL INFILE '2017Q1-capitalbikeshare-tripdata.csv' INTO TABLE trips
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\r\n'
  IGNORE 1 LINES
(duration, start_date, end_date, start_station_number, start_station,
end_station_number, end_station, bike_number, member_type);
```
在 `watch` 界面可以发现新增 `key`,并且这个 `key` 有部分数据是不可打印的
```txt
PUT
/pd/6856968710077637379/raft/s/00000000000000000004
(binary)tikv1:20160*4.0.4:127.0.0.1:20180B(28e3d44b00700137de4fa933066ab83e5f8306cfH(binary)
PUT
/pd/6856968710077637379/raft/s/00000000000000000005
(binary)tikv0:20160*4.0.4:127.0.0.1:20180B(28e3d44b00700137de4fa933066ab83e5f8306cfH(binary)
PUT
/pd/6856968710077637379/raft/s/00000000000000000001
(binary)tikv2:20160*4.0.4:127.0.0.1:20180B(28e3d44b00700137de4fa933066ab83e5f8306cfH(binary)
```

### 查询数据
```sql
select * from trips order by trip_id desc limit 30;
```
该操作不增加任何 `key`