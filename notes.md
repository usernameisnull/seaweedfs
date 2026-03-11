## 三大核心组件
Master Server（管理元数据）
Volume Server（存储实际数据）
Filer（提供文件系统接口）

### Master Server：
查看 weed/command/master.go 和 weed/server/master_server.go
理解集群管理和元数据存储机制
### Volume Server：
查看 weed/command/volume.go 和 weed/server/volume_server.go
理解数据存储和检索机制
### Filer：
查看 weed/command/filer.go 和 weed/server/filer_server.go
理解文件系统接口和元数据管理

## 程序入口/架构
weed/weed.go - 主程序入口
weed/command/ - 各个子命令的实现
weed/server/ - 服务器相关代码
weed/filer/ - 文件系统接口实现
weed/storage/ - 数据存储相关代码
weed/pb/ - Protocol Buffer 定义

## dashboard  
- 管理界面: weed/admin目录下  
- 执行命令: weed admin
- 访问: localhost:23646
  
## 流水线
[binaries_release1.yml](.github/workflows/binaries_release1.yml)里打包了linux的二进制

### 重点解读
build 目录: 进入weed目录执行build   
静态编译:  
```txt
CGO_ENABLED=0 → 静态编译（确保最终二进制最大兼容性）
```
禁用HTTP/2:  
- GODEBUG 是 Go 官方支持的调试环境变量集合，用于开启/关闭运行时和标准库中的一些调试行为。
- HTTP/2 客户端属于 Go 标准库的 net/http 包，但实际实现位于: `vendor/golang.org/x/net/http2
`
```txt
GODEBUG=http2client=0 → 禁用 Go 的 HTTP/2 客户端（SeaweedFS 的某些 client 兼容性用）
```
build标志位
```txt
-s -w → 删除调试信息 → 二进制更小
-extldflags -static → 完全静态链接
-X pkg.Var=value → 注入版本信息
```
Large Disk 版本（增加 build tags）  
开启 SeaweedFS 的大磁盘模式（Large Disk Mode）这会启用 SeaweedFS 中的另一个编译路径，支持更大的 file offset。  
```txt
build_flags: -tags 5BytesOffset
```

## 其他
### 为啥version子命令会输出30GB？
```cgo
 weed version
version 30GB 4.07 bc853bdee5c2b34ae0423e5c2c9f1b8b30196bcb linux amd64

For enterprise users, please visit https://seaweedfs.com for SeaweedFS Enterprise Edition,
which has a self-healing storage format with better data protection
```

### github的release页面多个版本有什么区别
每个CPU和操作系统都有这4种版本, [What's the difference between large_disk and large_disk_full?](https://github.com/seaweedfs/seaweedfs/wiki/FAQ#whats-the-difference-between-large_disk-and-large_disk_full)
```cgo
linux_amd64.tar.gz
linux_amd64_full.tar.gz
linux_amd64_full_large_disk.tar.gz
linux_amd64_large_disk.tar.gz
```