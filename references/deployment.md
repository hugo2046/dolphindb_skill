# DolphinDB 参考：部署

> 来源：`deploy/deploy_intro.html`, `tutorials/standalone_server.html`, `tutorials/single_machine_cluster_deploy.html`

## 部署模式总览

| 模式 | 适用场景 | 节点数 |
|------|----------|--------|
| 单节点 | 开发测试、小规模生产 | 1 |
| 单机集群 | 中小规模生产（同一台服务器多核） | 多 |
| 多机集群 | 大规模生产（多台服务器） | 多 |
| 高可用集群 | 生产高可用（需 Raft） | 3+ |
| Docker/K8s | 云原生部署 | 多 |

---

## 单节点部署

### 启动步骤

```bash
# 1. 下载并解压
unzip DolphinDB_Linux64_Vx.xx.xx.zip

# 2. 进入 server 目录
cd server/

# 3. 修改权限（Linux 首次）
chmod +x dolphindb

# 4. 前台启动
./dolphindb

# 5. 后台启动（推荐）
nohup ./dolphindb -console 0 &
```

**Windows:** 双击 `dolphindb.exe`，或以管理员身份运行

### 基础配置（dolphindb.cfg）

```ini
# 节点本地地址和端口
localSite=localhost:8848:local8848

# 许可证文件路径（相对 server 目录）
license=dolphindb.lic

# 最大内存（字节）
maxMemSize=16

# 最多工作线程数
workerNum=8

# 持久化路径
volumes=/data/dolphindb/volumes

# 日志路径
logFile=/var/log/dolphindb.log

# Web 管理端口（默认和 localSite 相同）
# dataPort=8848
```

---

## 单机集群部署

单机集群包含：**控制节点（controller）、代理节点（agent）、数据节点（datanode）**

### 目录结构

```
server/
├── controller.cfg      # 控制节点配置
├── cluster.cfg         # 集群全局配置
├── cluster.nodes       # 节点列表
├── agent.cfg           # 代理节点配置
└── StartCluster.sh     # 启动脚本（Linux）
```

### cluster.nodes 示例

```
localSite,mode
localhost:8848:controller,controller
localhost:8849:agent1,agent
localhost:8850:datanode1,datanode
localhost:8851:datanode2,datanode
localhost:8852:datanode3,datanode
```

### controller.cfg 示例

```ini
localSite=localhost:8848:controller
localExecutors=3
maxConnections=128
maxMemSize=4
logFile=/log/controller.log
dfsReplicationFactor=2
dfsReplicaReliabilityLevel=1
dataSync=1
```

### cluster.cfg 示例（数据节点通用配置）

```ini
maxConnections=32
maxMemSize=8
volumes=/data/ddb/volumes
batchJobWorkerNum=4
```

### 启动命令

```bash
# 1. 启动控制节点
nohup ./dolphindb -mode controller -home ./controller -config ./controller.cfg &

# 2. 启动代理节点
nohup ./dolphindb -mode agent -home ./agent -config ./agent.cfg &

# 3. 通过 Web 或脚本启动数据节点
// Web: http://localhost:8848 → 集群管理 → 启动节点
// 脚本:
rpc("controller:8848", startDataNode, ["datanode1", "datanode2"])
```

---

## 多机集群部署

多机部署与单机集群类似，区别：
- 每台机器部署对应角色（controller/agent/datanode）
- 节点配置文件中 `localhost` 改为实际 IP
- 需要保证各节点间网络互通、防火墙开对应端口

```ini
# 示例：datanode 节点的 cluster.nodes
192.168.1.100:8848:controller,controller
192.168.1.100:8849:agent1,agent
192.168.1.101:8850:datanode1,datanode
192.168.1.102:8851:datanode2,datanode
```

---

## Docker 部署

```bash
# 拉取镜像（官方）
docker pull dolphindb/dolphindb:latest

# 单节点启动
docker run -d \
  -p 8848:8848 \
  -v /data/ddb:/dolphindb/server \
  --name ddb \
  dolphindb/dolphindb:latest

# 进入容器
docker exec -it ddb /bin/bash
```

---

## 高可用（HA）部署

高可用需要至少 3 个控制节点（Raft 协议）

```ini
# controller.cfg（高可用）
localSite=192.168.1.100:8848:controller1
mode=controller
dfsReplicationFactor=2
dataSync=1
# HA 配置
raftElectionTick=1000
raftHeartbeatTick=100
```

```ini
# cluster.nodes（高可用）
192.168.1.100:8848:controller1,controller
192.168.1.101:8848:controller2,controller
192.168.1.102:8848:controller3,controller
```

---

## 管理与维护

```dolphindb
// 查看集群节点状态
getClusterDFSNodes()

// 查看正在运行的作业
getRecentJobs()

// 取消作业
cancelJob("jobId")

// 查看内存使用
getSessionMemoryStat()

// 查看分区分布
getClusterDFSNodes()[`volumes]

// 热升级（滚动升级data节点）
stopDataNode("datanode1")
// 更换 dolphindb 可执行文件后
startDataNode("datanode1")
```
