# DolphinDB 参考：配置参数

> 来源：`db_distr_comp/cfg/function_configuration.html`

## 配置文件位置

| 文件 | 位置 | 用途 |
|------|------|------|
| `dolphindb.cfg` | server 目录 | 单节点配置 |
| `controller.cfg` | server 目录 | 控制节点配置 |
| `agent.cfg` | server 目录 | 代理节点配置 |
| `cluster.cfg` | server 目录 | 集群数据节点通用配置 |
| `cluster.nodes` | server 目录 | 节点清单 |

---

## 核心配置参数

### 网络与连接

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `localSite` | 节点地址（ip:port:alias） | localhost:8848:local8848 |
| `maxConnections` | 最大连接数 | 64 |
| `maxConnectionPoolSize` | 连接池大小 | 0（不限） |
| `workerNum` | 工作线程数 | CPU 核数 |
| `localExecutors` | 本地执行线程数 | CPU核数-1 |
| `webWorkerNum` | Web 服务线程数 | 4 |

### 内存配置

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `maxMemSize` | 最大内存用量（GB） | 0（系统内存80%） |
| `maxTempTableSize` | 临时表最大内存（MB） | 1024 |
| `cacheEngineSize` | 写入缓存大小（GB） | 1 |
| `maxPartitionNumPerQuery` | 单查询最多扫描分区数 | 65536 |

### 存储与数据

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `volumes` | 数据存储路径（多路径用,分隔）| `home/volumes` |
| `dfsReplicationFactor` | DFS 副本数 | 1 |
| `dfsReplicaReliabilityLevel` | 副本可靠性级别（0/1/2） | 0 |
| `chunkCacheEngineMemSize` | chunk 写入缓存（GB） | 0 |
| `dataSync` | 数据同步到磁盘（0/1）| 0 |
| `redoLogDir` | Redo log 目录 | |
| `TSDBRedoLogDir` | TSDB Redo log 目录 | |

### 日志配置

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `logFile` | 日志文件路径 | dolphindb.log |
| `logLevel` | 日志级别（DEBUG/INFO/WARNING/ERROR） | INFO |
| `maxLogSize` | 单日志文件最大尺寸（MB） | 1024 |
| `logRetentionTime` | 日志保留时间（小时） | 7d=168 |

### 安全配置

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `enableSSL` | 启用 SSL | false |
| `enableHTTPS` | 启用 HTTPS | false |
| `enableChunkGranularityConfig` | 分块粒度权限控制 | false |
| `enableEncryption` | 数据加密 | false |

### 性能调优

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `batchJobWorkerNum` | 批处理作业工作线程数 | CPU 核数 |
| `asyncIOWorkerNum` | 异步 IO 线程数 | 1 |
| `subWorkersNum` | 流数据订阅工作线程数 | 1 |
| `maxAsyncTask` | 最大异步任务数 | 0 |
| `parallelNum` | 并行执行线程数 | CPU 核数 |

### 流数据配置

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `subPort` | 流数据订阅端口 | 0（禁用） |
| `persistenceDir` | 流表持久化目录 | |
| `persistenceOffsetDir` | 订阅偏移量持久化目录 | |
| `maxPubConnections` | 最大发布连接数 | 0（不限） |
| `maxSubConnections` | 最大订阅连接数 | 64 |

---

## 常用配置模板

### 开发测试环境（8核16GB）

```ini
localSite=localhost:8848:dev
workerNum=4
localExecutors=3
maxMemSize=12
volumes=/data/dolphindb/volumes
logFile=/var/log/dolphindb.log
subPort=8849
persistenceDir=/data/dolphindb/persist
```

### 生产数据节点（32核128GB/NVMe SSD）

```ini
localSite=192.168.1.101:8850:datanode1
workerNum=16
localExecutors=15
maxMemSize=96
volumes=/ssd1/dolphindb/volumes,/ssd2/dolphindb/volumes
cacheEngineSize=8
dfsReplicationFactor=2
dataSync=1
logLevel=WARNING
maxLogSize=2048
batchJobWorkerNum=8
asyncIOWorkerNum=4
```

---

## 运行时查看配置

```dolphindb
// 查看单个配置
getConfig("maxMemSize")

// 查看所有配置
getConfigContent()

// 修改配置（不重启，部分参数支持）
setConfig("maxConnectionPoolSize", 100)

// 查看当前节点信息
getNodeType()    // "controller" / "datanode" / "standalone"
getNodeAlias()   // 节点别名
```
