# DolphinDB 参考：错误码与故障排查

> 来源：`error_codes/err_codes.html`, `error_codes/troubleshooting.html` 和各具体错误码文件

---

## 错误码格式说明

```
S + <模块编号(2位)> + <类别内编号(3位)>
示例：S01001 → S(Server) + 01(存储模块) + 001(第1个存储错误)
```

### 模块分类

| 模块编号 | 模块 | 涵盖范围 |
|----------|------|---------|
| S00 | 系统 | 网络错误、磁盘错误、文件操作、内存错误 |
| S01 | 存储 | DFS、OLAP、TSDB、事务、Raft、Recovery、备份恢复、异步复制 |
| S02 | SQL | SQL 解析与执行相关错误 |
| S03 | 流数据 | 流计算、流计算引擎、流数据引擎解释器 |
| S04 | 管理 | 数据库管理、权限管理、函数视图、集群管理 |
| S05 | 通用 | 基础数据结构相关的难以归类错误 |
| S06 | DolphinDB 解释器 | 语法错误和语法解析 |
| S07 | Swordfish | Swordfish 存储引擎 |
| S09 | JIT | JIT 即时编译 |
| S10 | GPU | GPU 计算 |
| S12 | PrimaryKeyEngine | PKEY 主键引擎 |

> **提示**：报错信息末尾的 `RefId: SxxxYY` 就是对应的错误码，可直接在文档中查找。
>
> 报错中如有 `with error 13` 等数字，参考底部的**原始报错码对照表**。

---

## 常见故障速查（按现象分类）

### 🔌 无法连接服务器

**常见原因与检查步骤：**

1. **检查服务端进程是否运行**
   ```bash
   # Linux
   ps aux | grep dolphindb
   # 查看日志
   tail -f server/logs/dolphindb.log
   ```

2. **检查端口是否正常**
   ```bash
   netstat -tlnp | grep 8848
   telnet localhost 8848
   ```

3. **防火墙检查**
   ```bash
   firewall-cmd --list-ports   # Linux
   # Windows: 检查防火墙是否开放 8848 端口
   ```

4. **许可证问题**
   - 检查 `dolphindb.lic` 是否存在且未过期
   - 许可证绑定的 IP/MAC 与当前机器是否匹配

---

### 🚀 节点启动异常

**常见原因：**

| 错误现象 | 可能原因 | 解决方法 |
|---------|---------|---------|
| 启动后立刻退出 | 端口被占用 | 修改 `dolphindb.cfg` 中的 `port` 参数 |
| `License is expired` | 许可证过期 | 联系 DolphinDB 更新许可证 |
| `Cannot read config file` | 配置文件路径错误 | 检查 `-c` 参数或默认路径 |
| `Failed to bind the port` | 端口冲突 | `lsof -i:8848` 查找占用进程并释放 |
| 集群控制节点连不上 | controller.cfg 中地址配置错误 | 检查 `controllerSite` 配置 |
| 存储不可用 | 磁盘满了 | `df -h` 检查磁盘空间 |

---

### 💥 节点宕机

**排查步骤：**

1. 查看崩溃日志
   ```dolphindb
   // 在 GUI 中查看
   getServerLog(100)   // 最近 100 行
   ```

2. 常见崩溃原因
   - **OOM（内存不足）**：见「Out of Memory」章节
   - **磁盘 IO 错误**：硬件故障或 NFS 挂载问题
   - **Segment Fault**：通常是 bug，上报给 DolphinDB 官方

---

### 🐌 查询/写入慢

**诊断命令：**

```dolphindb
// 查看当前运行的作业
getRecentJobs()

// 查看等待中的任务
getPendingJobs()

// 查看内存使用
getMemoryStat()

// 查看节点负载
rpc("datanoe1", getClusterMetrics, "CPU_USAGE,MEM_USED")
```

**常见原因：**

| 原因 | 诊断 | 解决 |
|------|------|------|
| 分区过多/过大 | `getAllChunks()` 查看分区分布 | 重新规划分区方案 |
| 未走分区剪枝 | 查看 explain 输出 | WHERE 条件加入分区列 |
| cacheEngine 积压 | `getCacheEngineStat()` | 调大 `cacheEngineSize` 配置 |
| 并发写入冲突 | 日志中有大量 WAIT 事务 | 减少并发写入，或换 TSDB 引擎 |
| 磁盘 IO 瓶颈 | `iotop` 查看磁盘 | 换 SSD，或调整 RAID 配置 |

---

### 💾 Out of Memory（OOM）

```dolphindb
// 查看内存使用详情
mem()                    // 节点内存占用
getMemoryStat()          // 各类内存统计
objs()                   // 列出所有变量及大小
```

**常见原因与解决：**

```dolphindb
// 1. 查询返回数据过多 - 限制行数
select top 10000 * from t where ...

// 2. 全表扫描 - 加分区条件
select * from t where TradeDate = 2024.01.01  // ← 走分区

// 3. 清理无用变量
undef(`largeVar)         // 删除变量
undef all                // 清除所有变量

// 4. 内存泄漏检查
// 查看是否有大的流表/内存表占用
select name, rows, bytes from objs() order by bytes desc
```

**配置调整（重启生效）：**
```ini
# dolphindb.cfg
maxMemSize=64            # 最大内存（GB），建议为物理内存的 85%
warningMemSize=56        # 触发警告的内存阈值（GB）
```

---

## S00 系统错误（高频）

### S00001 - 流数据订阅/发布未启用

**报错：**
```
The feature of publish is not enabled. RefId: S00001
Can't complete subscription because subscription is not enabled. RefId: S00001
```
**原因：** 默认未启用流数据发布/订阅功能。

**解决：** 在配置文件中添加：
```ini
# 发布端（服务端配置）
maxPubConnections=10     # 允许的最大订阅连接数

# 订阅端（客户端配置）
subPort=8849             # 订阅监听端口
```

---

## S01 存储错误（高频）

### S01001 - VALUE 分区列含 NULL

**报错：** `The column used for value-partitioning cannot contain NULL values. RefId: S01001`

**原因：** VALUE 分区的分区列不能有 NULL。

**解决：**
```dolphindb
// 写入前过滤 NULL
t = select * from rawData where !isNull(TradeDate)
t.append!(t)
```

### S01007 - 分区列数据超出分区范围

**报错：** `Data [xxx] out of the range of partition [xxx]. RefId: S01007`（日期/值不在分区范围内）

**解决：**
```dolphindb
// 方法1：扩展分区范围（重新建库）
// 方法2：使用 addValuePartitions 动态扩展 VALUE 分区
addValuePartitions("dfs://mydb", 2025.01.01..2025.12.31)
```

### S01012 - 写入数据列数与表结构不匹配

**报错：** `The number of columns of the data doesn't match the table. RefId: S01012`

**解决：**
```dolphindb
// 检查列数
schema(loadTable("dfs://mydb", "t")).colDefs    // 表结构
cols = colNames(data)                             // 数据列名
```

### S01014 - TSDB 排序键列值不能含 NULL

**报错：** `The sort column [xxx] cannot contain NULL values. RefId: S01014`

**解决：** 检查 `SORTBY` 定义的排序列，确保不含 NULL。

### S01035 - 超出 SYMBOL 类型最大不同值数量

**报错：** `The number of symbol values exceeded the limit. RefId: S01035`

**原因：** 每个分区中 SYMBOL 列不同值不能超过 2^21 = 2097152。

**解决：**
```dolphindb
// 选项1：使用 STRING 类型代替 SYMBOL
// 选项2：重新规划分区，减少每个分区的数据量
// 检查 SYMBOL 列的不同值数量
select count(distinct sym) from t
```

---

## S02 SQL 错误（高频）

### S02000 - unpivot 列类型不一致

**报错：** `Columns specified in valueColNames of function unpivot must be of the same data type. RefId: S02000`

**解决：** 确保 `valueColNames` 指定的列都是同一类型：
```dolphindb
// 先类型转换再 unpivot
t2 = select id, double(col1) as col1, double(col2) as col2 from t
t2.unpivot(keyColNames=`id, valueColNames=`col1`col2)
```

### S02015 - SELECT 的列不在 GROUP BY 中

**报错：** `xxx is not an aggregation function or in the group-by fields. RefId: S02015`

**解决：**
```dolphindb
// 错误
select sym, price, sum(vol) from t group by sym
// 正确：price 必须是聚合函数或在 group by 中
select sym, last(price) as price, sum(vol) from t group by sym
```

### S02025 - 分布式表不支持的操作

**报错：** `The table does not allow updating. RefId: S02025`（或类似分布式操作限制）

**解决：** 分布式表（dfs://）有些操作限制，使用对应解决方法：
```dolphindb
// 更新分布式表（OLAP 引擎不支持 UPDATE）
// 正确方式：使用 sqlUpdate
sqlUpdate(t, <price=price*1.1>, <where sym=`AAPL>)

// 或重新写入（OLAP 引擎的标准做法）
delete from t where TradeDate = 2024.01.01
t.append!(newData)
```

---

## S03 流数据错误（高频）

### S03000 - 流表不存在或名称错误

**报错：** `Stream table [xxx] does not exist. RefId: S03000`

**解决：**
```dolphindb
// 检查流表是否存在
objs()
// 重新建立流表
share streamTable(1000000:0, colNames, colTypes) as myStream
```

### S03001 - 重复订阅（相同 actionName）

**报错：** `Subscription [actionName] already exists. RefId: S03001`

**解决：**
```dolphindb
// 先取消已有订阅再重新订阅
unsubscribeTable(tableName="myStream", actionName="sub1")
subscribeTable(tableName="myStream", actionName="sub1", offset=-1,
    handler=handler, msgAsTable=true)
```

### S03005 - 订阅 offset 超出流表范围

**报错：** `The offset [xxx] is out of range. RefId: S03005`

**原因：** 指定的 offset 超出了流表当前已有的消息数量（如流表已被清空或重建）。

**解决：**
```dolphindb
// 使用 -1 从最新数据开始，避免 offset 越界
subscribeTable(tableName="myStream", actionName="sub1",
    offset=-1, handler=handler, msgAsTable=true)

// 或使用 0 从头订阅
subscribeTable(tableName="myStream", actionName="sub1",
    offset=0, handler=handler, msgAsTable=true)
```

### S03006 - 发布队列溢出

**报错：** `Publishing queue is full. RefId: S03006` 或订阅端接收异常

**原因：** 发布节点的 `maxPubQueueDepthPerSite` 配置的发布队列已满，通常因订阅端处理速度过慢或网络拥塞导致。

**解决：**
```dolphindb
// 查看发布队列状态
getStreamingStat().pubTables          // 发布表列表
getStreamingStat().pubWorkers         // 发布工作线程
// 参考 tutorials_ops.md「流数据消息缓存队列」章节：
// → 增大配置：maxPubQueueDepthPerSite=10000000
// → 优化订阅端 handler，提升消费速度
```

### S03007 - 订阅接收队列溢出

**报错：** `Subscription queue is full for [tableName]. RefId: S03007`

**原因：** 订阅端 `maxSubQueueDepth` 超限，handler 处理速度跟不上数据推送速度。

**解决：**
```dolphindb
// 查看订阅队列状态
getStreamingStat().subWorkers         // 各订阅线程队列深度
// 解决方案：
// 1. 优化 handler，减少单条处理耗时
// 2. 增大配置：maxSubQueueDepth=10000000
// 3. 使用 batchSize 批量处理，减少处理次数
subscribeTable(tableName="myStream", actionName="sub1",
    offset=-1, handler=handler, msgAsTable=true,
    batchSize=1000, throttle=1.0)   // 每 1000 条或 1 秒触发一次
```

### S03008 - 流计算引擎输入输出表结构不匹配

**报错：** `The schema of the input/output table doesn't match. RefId: S03008`

**原因：** 创建流计算引擎时，`dummyTable`（输入表结构定义）或 `outputTable` 的列名/类型与实际流表不一致。

**解决：**
```dolphindb
// 检查流表结构
schema(myInputStream)
// 确认 dummyTable 与流表一致
// 确认 metrics 输出列数与 outputTable 列数匹配
```

> 💡 **流数据内存排查**：如遇到流数据引起的内存压力，参考
> `tutorials_ops.md` →「流数据消息缓存队列」一节，
> 使用 `getStreamingStat()` 和 `getStreamEngineStat()` 监控队列深度与引擎内存。



## S04 管理错误（高频）

### S04001 - 权限不足

**报错：** `No access to [xxx]. RefId: S04001` 或 `Not authorized. RefId: S04001`

**解决：**
```dolphindb
// 以 admin 运行，授权
login("admin", "123456")
grant("myUser", TABLE_READ, "dfs://mydb/*")
grant("myUser", TABLE_WRITE, "dfs://mydb/*")
grant("myUser", DB_MANAGE, "dfs://mydb")
```

---

## S05 通用错误（高频）

### S05000 - 类型不匹配

**报错：** `Incompatible type. RefId: S05000`

**解决：**
```dolphindb
// 显式类型转换
int(x)        // 转 INT
double(x)     // 转 DOUBLE
string(x)     // 转 STRING
temporalParse(s, "yyyy.MM.dd")  // 字符串 → 日期
```

### S05001 - 索引越界

**报错：** `Index out of range. RefId: S05001`

```dolphindb
// 检查向量大小
size(v)
// 安全访问
if (i < size(v)) { val = v[i] }
```

---

## S06 解释器错误（高频）

### S06000 - 变量未定义

**报错：** `[xxx] is not defined. RefId: S06000`

**常见原因：**
```dolphindb
// 1. 函数名拼写错误（DolphinDB 区分大小写！）
avg(v)    // ✅ 正确
Avg(v)    // ❌ 错误
AVG(v)    // ❌ 错误

// 2. 模块未加载
use ta         // 需要先 use 模块
ta::ma(v, 5)

// 3. 在函数内使用了外部变量
// 函数内部要使用外部变量，需要先赋值到局部
def myFunc() {
    localV = globalV   // ← 先拷贝到局部
    avg(localV)
}
```

### S06001 - 语法错误

**报错：** `Syntax Error. RefId: S06001`

**检查：**
```dolphindb
// 常见语法问题
// 1. 忘记 return 语句
def f(x) {
    x * 2       // ❌ 没有 return，返回 NULL
}
def f(x) {
    return x * 2  // ✅
}

// 2. 括号不匹配
// 3. 分号在不应该的地方
// 4. 字符串引号不匹配
```

---

## 原始报错码对照表

报错信息中出现 `with error N` 时对应：

| 码 | 含义 |
|----|------|
| 1 | Socket 断开或文件已关闭 |
| 2 | 非阻塞模式下无数据可读 |
| 3 | 内存不足或磁盘空间不足 |
| 4 | 字符串超过 64K 或代码超过 1MB |
| 5 | 非阻塞模式下连接正在进行 |
| 6 | 消息格式错误 |
| 7 | 文件/Buffer 已到末尾 |
| 8 | 文件只读不可写 |
| 9 | 文件只写不可读 |
| 10 | 文件不存在或 socket 不可达 |
| 11 | 数据库文件损坏 |
| 12 | 不是 Raft 协议的 Leader 节点 |
| 13 | 未知 IO 错误 |

---

## 通用调试命令

```dolphindb
// 查看服务端日志（最近N行）
getServerLog(100)
getServerLog(100, false)  // 不含时间戳

// 查看当前作业
getJobStatus()
getRecentJobs()
cancelJob("jobId")

// 查看集群状态
getClusterPortInfo()
getClusterStat()

// 检查数据库元数据
getTabletsMeta("dfs://mydb")
getPartitionInfo("dfs://mydb", `trades, 2024.01.01)

// 查看内存/性能
mem()
getMemoryStat()
getPerf()
```
