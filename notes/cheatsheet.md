## 启动
```bash
# start one master, one volume server, one filer, and one S3 gateway
# /tmp/seaweed需要在启动命令前存在
# server子命令是一个全家桶模式： start a master server, a volume server, and optionally a filer and a S3 gateway
mkdir -p /tmp/seaweed 
AWS_ACCESS_KEY_ID=admin AWS_SECRET_ACCESS_KEY=root@123! weed server -dir=/tmp/seaweed -s3
```
启动后监听的端口
```cgo
➜ ss -tulnp | grep 797780
tcp   LISTEN 0      4096       127.0.0.1:18333      0.0.0.0:*    users:(("weed",pid=797780,fd=65))
tcp   LISTEN 0      4096   172.24.91.234:9333       0.0.0.0:*    users:(("weed",pid=797780,fd=7))
tcp   LISTEN 0      4096   172.24.91.234:18333      0.0.0.0:*    users:(("weed",pid=797780,fd=63))
tcp   LISTEN 0      4096       127.0.0.1:9333       0.0.0.0:*    users:(("weed",pid=797780,fd=8))
tcp   LISTEN 0      4096   172.24.91.234:18080      0.0.0.0:*    users:(("weed",pid=797780,fd=16))
tcp   LISTEN 0      4096   172.24.91.234:8333       0.0.0.0:*    users:(("weed",pid=797780,fd=61))
tcp   LISTEN 0      4096       127.0.0.1:8888       0.0.0.0:*    users:(("weed",pid=797780,fd=55))
tcp   LISTEN 0      4096       127.0.0.1:8333       0.0.0.0:*    users:(("weed",pid=797780,fd=64))
tcp   LISTEN 0      4096   172.24.91.234:8888       0.0.0.0:*    users:(("weed",pid=797780,fd=54))
tcp   LISTEN 0      4096   172.24.91.234:8080       0.0.0.0:*    users:(("weed",pid=797780,fd=17))
tcp   LISTEN 0      4096       127.0.0.1:19333      0.0.0.0:*    users:(("weed",pid=797780,fd=11))
tcp   LISTEN 0      4096   172.24.91.234:18888      0.0.0.0:*    users:(("weed",pid=797780,fd=56))
tcp   LISTEN 0      4096   172.24.91.234:19333      0.0.0.0:*    users:(("weed",pid=797780,fd=10))
tcp   LISTEN 0      4096       127.0.0.1:18888      0.0.0.0:*    users:(("weed",pid=797780,fd=57))
```

### 4.04版本添加了mini子命令
- release: https://github.com/seaweedfs/seaweedfs/releases/tag/4.04
- pr: https://github.com/seaweedfs/seaweedfs/pull/7831

### 使用别的端口

### 增加容量
具体怎么操作? 
```bash
weed volume -dir="/some/data/dir2" -master="<master_host>:9333" -port=8081
```

## dashboard
- 管理界面: weed/admin目录下
- 执行命令: weed admin -adminUser admin -adminPassword admin@123!
- 访问: localhost:23646

## 类似minio的mc的命令行工具
好像没有

### 用纯http的方式
```bash
# 先获取fid
curl http://localhost:9333/dir/assign
{"fid":"7,5de47aa96c","url":"172.24.91.234:8080","publicUrl":"172.24.91.234:8080","count":1}

# 上传文件
curl -F file=@/tmp/cat.jpg http://172.24.91.234:8080/7,5de47aa96c

# 查看文件
curl http://172.24.91.234:8080/7,5de47aa96c

# 删除文件
curl -X DELETE http://172.24.91.234:8080/7,5de47aa96c
```
上面的查看文件这一步, 为什么没法通过bucket这种方式在localhost:23646这个管理页面查看?  
The number 3 at the start represents a volume id. After the comma, it's one file key, 01, and a file cookie, 637037d6.
`7,5de47aa96c`的解读:  
- 7: 代表volume id
- 5d: file key
- e47aa96c: file cookie

## Filer
需要MySql, Postgres, Redis, Cassandra, HBase, Mongodb, Elastic Search, 
LevelDB, RocksDB, Sqlite, MemSql, TiDB, Etcd, CockroachDB, YDB之类的来存储metadata?

## key-value
在典型的分布式键值存储（如 Redis、etcd 或其他 NoSQL 数据库）中，如果某些值（value）体积很大（例如图片、视频、大文件等），
直接存储在原系统中可能会影响性能或扩展性。这时，可以将这些“大值”存放到专门优化用于存储大对象的系统——SeaweedFS 中，
而原键值存储只保留指向这些大值的引用（如 URL 或文件 ID）