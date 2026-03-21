# weed mini 命令分析

`weed mini` 是一个**单进程全家桶**，把所有服务都塞进一个进程里跑。

源文件：`weed/command/mini.go`

---

## 启动了哪些服务

| 服务 | HTTP 端口 | gRPC 端口 | 说明 |
|------|-----------|-----------|------|
| Master | **9333** | 19333 | Raft 选主 + 卷分配 |
| Volume Server | **9340** | 19340 | 实际存储数据 |
| Filer | **8888** | 18888 | 文件系统元数据（LevelDB） |
| S3 + IAM | **8333** | 18333 | S3 兼容接口，读取 `AWS_*` 环境变量 |
| Iceberg REST Catalog | **8181** | — | S3 的子服务 |
| WebDAV | **7333** | — | WebDAV 接口 |
| Admin UI | **23646** | 33646 | 管理界面 + 维护任务 |
| Metrics | — | — | Prometheus 指标 |

可通过 flag 禁用部分服务：`-s3=false`、`-webdav=false`、`-admin.ui=false`

---

## 严格的启动顺序（`mini.go:879`）

```
1. Master 启动        → 等待 :9333 就绪（每 200ms 轮询，最多 30 次）
2. Volume Server 启动 → 等待 :9340 就绪
3. Filer 启动         → 等待 :8888 就绪
4. S3 + WebDAV 启动   → 并行启动，等待 :8333 / :8181 / :7333
5. Admin UI 启动      → 等待 /health 通过（最多 30s）
   ├─ Maintenance Worker（vacuum / ec / balance，MaxConcurrent=2）
   └─ Plugin Worker（job types="all"）
```

每一步依赖前一步就绪才启动：Volume 需要先注册到 Master，Filer 需要连接 Master，S3 需要连接 Filer。

---

## 示例命令解析

```bash
export AWS_ACCESS_KEY_ID=admin
export AWS_SECRET_ACCESS_KEY=Root@123!
mkdir -p /tmp/seaweed
weed mini -dir=/tmp/seaweed
```

- `-dir=/tmp/seaweed` 同时作为 volume 数据目录和 filer 元数据目录（LevelDB 存于 `/tmp/seaweed/filerldb2`）
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` 由 `startS3Service`（`mini.go:969`）读取，用来初始化 S3 IAM 的初始账号（`admin` / `Root@123!`），S3 IAM 默认为只读模式（`-s3.iam.readOnly=true`）
- volume 大小根据 `/tmp/seaweed` 所在磁盘总容量自动计算：取容量的 1%，向上取整到 2 的幂次，夹在 **64MB ~ 1024MB** 之间
- 配置持久化到 `/tmp/seaweed/mini.options`，下次启动自动读取（CLI flag 始终优先）
- 若默认端口被占用，`ensureAllPortsAvailableOnIP`（`mini.go:465`）会自动探测并分配下一个可用端口

---

## 各组件初始化细节

### Master（`weed/command/master.go:172`）
- 创建 `weed_server.NewMasterServer`，启动 Raft 选主
- 单 master 模式下立即执行 `DoJoinCommand()`（`master.go:228`）
- 同时监听 HTTP（9333）和 gRPC（19333）

### Volume Server（`weed/command/volume.go:172`）
- `maxVolumeCounts="0"`（自动），`minFreeSpace="1"`（保留 1% 可用空间）
- 索引类型默认 `"memory"`，读模式默认 `"proxy"`
- 支持 `-dir` 逗号分隔多目录

### Filer（`weed/command/filer.go:313`）
- LevelDB 元数据存于 master meta folder 下的 `filerldb2` 子目录
- 初始化 `CredentialManager` 供 IAM gRPC 使用

### S3 + IAM（`weed/command/s3.go:235`）
- 通过 gRPC 连接 Filer，获取 bucket 路径、filer group、master 地址
- 创建 `s3api.NewS3ApiServer`，内嵌 IAM（`enableIam: true`）

### WebDAV（`weed/command/webdav.go:76`）
- 直接指向 filer 地址（`mini.go:824`）

### Admin UI（`weed/command/admin.go:267`）
- gorilla/mux HTTP server，带 session-cookie 认证，内嵌静态文件
- 健康检查就绪后启动两个 worker goroutine

---

## 关键源文件

| 文件 | 作用 |
|------|------|
| `weed/command/mini.go` | 命令定义、flag 初始化、端口分配、启动编排 |
| `weed/command/master.go:172` | `startMaster` — Raft + gRPC master server |
| `weed/command/volume.go:172` | `startVolumeServer` — volume 存储引擎 |
| `weed/command/filer.go:313` | `startFiler` — 元数据 filer + LevelDB |
| `weed/command/s3.go:235` | `startS3Server` — S3 + IAM 网关 |
| `weed/command/webdav.go:76` | `startWebDav` — WebDAV 网关 |
| `weed/command/admin.go:267` | `startAdminServer` — Admin UI + gRPC server |
