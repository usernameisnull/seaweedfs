# SeaweedFS 文件写入流程

整体分两个阶段：**获取写入位置（Master Assign）** + **实际写入数据（Volume Server Write）**。

---

## 阶段一：Master 分配写入位置

```
Client GET /dir/assign
  -> MasterServer.dirAssignHandler   (weed/server/master_server_handlers.go:124)
  -> Topology.PickForWrite           (weed/topology/topology.go:322)
       -> VolumeLayout.PickForWrite  (weed/topology/volume_layout.go:300)  -- 随机选可写 volume
       -> Sequence.NextFileId()                                             -- 生成单调递增的 needle ID
  -> 返回 {fid, url, auth JWT}
```

Client 收到形如 `3,01637037d6` 的 `fid` 和对应 volume server 地址。

也可以通过 gRPC 接口调用：`MasterServer.Assign`（`weed/server/master_grpc_server_assign.go:49`），高吞吐场景下 filer 使用 `StreamAssign` 流式接口（同文件 line 24）。

---

## 阶段二：写入 Volume Server

```
Client POST http://<volume-server>/<fid>
  -> privateStoreHandler                    (weed/server/volume_server_handlers.go:252)
  -> handleUploadRequest                    (weed/server/volume_server_handlers.go:232)  -- 并发限速
  -> PostHandler                            (weed/server/volume_server_handlers_write.go:19)
       -> needle.CreateNeedleFromRequest()  (weed/storage/needle/needle.go:52)
       |    -> ParseUpload()                (weed/storage/needle/needle_parse_upload.go:36)
       |         -- 读取 multipart body，可选 gzip 压缩，校验 Content-MD5
       |    -> 计算 CRC32: n.Checksum = NewCRC(n.Data)
       |    -> 从 URL 解析 n.Id 和 n.Cookie
       |
       -> topology.ReplicatedWrite()        (weed/topology/store_replicate.go:27)
            -> Store.WriteVolumeNeedle()    (weed/storage/store.go:580)
                 -> Volume.writeNeedle2()   (weed/storage/volume_write.go:118)
                      -> Volume.doWriteRequest()  (weed/storage/volume_write.go:139)
                           -> n.UpdateAppendAtNs()        -- 设置纳秒级写入时间戳
                           -> needle.Append()             (weed/storage/needle/needle_write.go:14)
                           |    -> writeNeedleCommon()    -- 序列化为 wire format
                           |    -> DiskFile.WriteAt()     -- 写入 .dat 文件
                           -> v.nm.Put(id, offset, size)  -- 更新内存索引 + .idx 文件
            -> 并行 POST ?type=replicate 到其他副本节点   -- DistributedOperation
```

---

## 磁盘文件结构

每个 volume 由两个文件组成：

| 文件 | 作用 |
|------|------|
| `<id>.dat` | append-only blob 文件，needle 顺序追加，8 字节对齐，文件头为 SuperBlock（8B） |
| `<id>.idx` | 固定宽度索引，每条 16 字节：`NeedleId(8B) + Offset(4B) + Size(4B)` |

读取时，`.idx` 加载到内存（btree 或 LevelDB），查找 needle 后计算 `offset = nv.Offset * 8`，再对 `.dat` 执行 `ReadAt`。

---

## Needle wire format（.dat 文件中的布局）

```
┌─────────────────────────────────────────────────────┐
│ Header                                              │
│   Cookie     4B   anti-brute-force 随机值           │
│   NeedleId   8B   单调递增文件 ID                   │
│   Size       4B   body 总长度                       │
├─────────────────────────────────────────────────────┤
│ Body                                                │
│   DataSize   4B                                     │
│   Data       NB   文件内容                          │
│   Flags      1B   压缩/分块标志                     │
│   Name       可选，原始文件名（max 255B）            │
│   Mime       可选，MIME 类型（max 255B）             │
│   LastModified 可选，unix 秒（5B）                  │
│   TTL        可选                                   │
│   Pairs      可选，JSON 元数据（max 64KB）           │
├─────────────────────────────────────────────────────┤
│ Footer                                              │
│   CRC32      4B                                     │
│   Timestamp  8B   纳秒精度（V3）                    │
│   Padding    NB   补齐至 8 字节边界                 │
└─────────────────────────────────────────────────────┘
```

序列化实现：`weed/storage/needle/needle_write_v2.go:18`

---

## fsync 批量优化

`weed/storage/volume_write.go:132` — 当 `fsync=true` 时，写请求进入 `asyncRequestsChan`，后台 worker（`startWorker`）将最多 **128 个请求**或 **4MB 数据**批量写入后统一调用一次 `Sync()`，显著减少 fsync 次数。

---

## 关键数据结构

### Needle（`weed/storage/needle/needle.go:25`）
```go
type Needle struct {
    Cookie       Cookie    // 4B，防暴力猜测
    Id           NeedleId  // 8B，来自 master 序列器
    Size         Size      // body 总长度

    DataSize     uint32
    Data         []byte    // 文件内容
    Flags        byte      // 压缩/分块标志
    Name         []byte
    Mime         []byte
    Pairs        []byte    // JSON 元数据
    LastModified uint64
    Ttl          *TTL

    Checksum     CRC       // CRC32 of Data
    AppendAtNs   uint64    // 纳秒级写入时间戳（V3）
}
```

### Volume（`weed/storage/volume.go:22`）
```go
type Volume struct {
    Id                needle.VolumeId
    dir               string                     // .dat 文件目录
    dirIdx            string                     // .idx 文件目录
    Collection        string
    DataBackend       backend.BackendStorageFile // DiskFile / mmap / 云存储
    nm                NeedleMapper               // 内存索引
    asyncRequestsChan chan *needle.AsyncRequest  // fsync 批量队列
    SuperBlock                                   // 副本策略、TTL、版本
    dataFileAccessLock sync.RWMutex
    lastAppendAtNs    uint64
}
```

### NeedleValue（`weed/storage/needle_map/needle_value.go:9`）
```go
type NeedleValue struct {
    Key    NeedleId  // 8B
    Offset Offset    // 4B，实际偏移 = Offset * 8
    Size   Size      // 4B
}
```

---

## 完整调用链速查表

| 步骤 | 文件 | 函数 | 行号 |
|------|------|------|------|
| HTTP 路由 | `weed/server/volume_server_handlers.go` | `privateStoreHandler` | 252 |
| 并发限速 | `weed/server/volume_server_handlers.go` | `handleUploadRequest` | 232 |
| 写入处理器 | `weed/server/volume_server_handlers_write.go` | `PostHandler` | 19 |
| 解析请求 | `weed/storage/needle/needle.go` | `CreateNeedleFromRequest` | 52 |
| 解析 body | `weed/storage/needle/needle_parse_upload.go` | `ParseUpload` | 36 |
| Master 分配（HTTP） | `weed/server/master_server_handlers.go` | `dirAssignHandler` | 124 |
| Master 分配（gRPC） | `weed/server/master_grpc_server_assign.go` | `Assign` | 49 |
| 选择可写 volume | `weed/topology/topology.go` | `PickForWrite` | 322 |
| volume layout 选择 | `weed/topology/volume_layout.go` | `VolumeLayout.PickForWrite` | 300 |
| 复制写入 | `weed/topology/store_replicate.go` | `ReplicatedWrite` | 27 |
| Store 分发 | `weed/storage/store.go` | `WriteVolumeNeedle` | 580 |
| Volume 写入 | `weed/storage/volume_write.go` | `writeNeedle2` | 118 |
| 核心写逻辑 | `weed/storage/volume_write.go` | `doWriteRequest` | 139 |
| 序列化 needle | `weed/storage/needle/needle_write.go` | `Needle.Append` | 14 |
| wire format | `weed/storage/needle/needle_write_v2.go` | `writeNeedleCommon` | 18 |
| 磁盘写入 | `weed/storage/backend/disk_file.go` | `DiskFile.WriteAt` | 55 |
| 更新索引 | `weed/storage/volume_write.go` | `v.nm.Put(...)` | 181 |
