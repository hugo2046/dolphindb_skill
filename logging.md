# DolphinDB 日志系统参考

## 概述

DolphinDB 日志系统涵盖三个层面：
1. **运行日志**：记录节点运行状态、错误、警告等系统事件
2. **用户自定义日志**：通过 `writeLog` / `writeLogLevel` 写入业务逻辑日志
3. **审计日志**：通过 `getAuditLog` 查询 DDL 操作记录

---

## 一、日志级别配置

### `setLogLevel` — 动态调整日志级别

**语法：**
```
setLogLevel(level)
```

**参数：**
| 参数 | 说明 |
|------|------|
| `level` | 字符串，可选值：`"DEBUG"`, `"INFO"`, `"WARNING"`, `"ERROR"` |

**权限要求：** 仅管理员（admin）可调用。

**说明：**
- 对当前节点生效，**在线调整**无需重启
- 日志级别由低到高：`DEBUG < INFO < WARNING < ERROR`
- 高于等于设定级别的日志才会被记录
- 节点重启后恢复配置文件中的默认级别

**示例：**
```dolphindb
// 切换到 DEBUG 模式以定位问题
setLogLevel("DEBUG")

// 生产环境设为 WARNING，减少日志量
setLogLevel("WARNING")

// 恢复默认 INFO 级别
setLogLevel("INFO")
```

**⚠️ 注意：**  
`DEBUG` 级别会产生大量日志，影响性能，仅在排查问题时临时开启。

---

## 二、写入自定义日志

### `writeLog` — 写入 INFO 级别日志

**语法：**
```
writeLog(obj1, [obj2, ...])
```

**参数：**
| 参数 | 说明 |
|------|------|
| `obj1, obj2, ...` | 字符串，每个参数写入一行日志 |

**权限要求：** 用户需先登录（login）。

**返回值：** 无（NULL）

**说明：**
- 每个参数写入独立一行，级别固定为 `INFO`
- 日志写入当前节点的运行日志文件（`dolphindb.log`）
- 如需指定级别，使用 `writeLogLevel`

**示例：**
```dolphindb
// 写入单条日志
writeLog("策略初始化完成")

// 写入多条日志（每条独立一行）
writeLog("=== 开始回测 ===", "起始日期: 2024-01-01", "结束日期: 2024-12-31")

// 在函数中记录关键步骤
def processData(t) {
    writeLog("processData 开始, 行数: " + string(t.size()))
    // ... 业务逻辑 ...
    writeLog("processData 完成")
}
```

### `writeLogLevel` — 写入指定级别日志

**语法：**
```
writeLogLevel(level, obj)
```

**参数：**
| 参数 | 说明 |
|------|------|
| `level` | 日志级别常量：`DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `obj` | 字符串，日志内容 |

**示例：**
```dolphindb
// 写入不同级别的日志
writeLogLevel(DEBUG, "进入函数 calcFactor, 参数: " + string(param))
writeLogLevel(INFO,  "因子计算完成，耗时: " + string(elapsed) + "ms")
writeLogLevel(WARNING, "数据延迟超过阈值: " + string(delay) + "s")
writeLogLevel(ERROR, "StockID 缺失，跳过记录")
```

**实践模式 — 业务日志封装：**
```dolphindb
// 封装带模块前缀的日志函数
def logInfo(module, msg) {
    writeLogLevel(INFO, "[" + module + "] " + msg)
}

def logError(module, msg) {
    writeLogLevel(ERROR, "[" + module + "] " + msg)
}

// 使用示例
logInfo("StreamEngine", "订阅表已创建: result_stream")
logError("DataLoader", "文件读取失败: " + path)
```

---

## 三、日志格式

DolphinDB 日志文件的标准格式：
```
2024-01-15 09:30:25.123456 <INFO> : 策略初始化完成
2024-01-15 09:30:26.456789 <ERROR> : StockID 缺失，跳过记录
```

字段含义：
- `时间戳`：精确到微秒
- `<级别>`：`<DEBUG>`, `<INFO>`, `<WARNING>`, `<ERROR>`
- `消息内容`：日志正文

---

## 四、审计日志查询

### `getAuditLog` — 查询 DDL 操作记录

**语法：**
```
getAuditLog([userId], [startTime], [endTime], [opType])
```

**参数：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `userId` | STRING | 用户名，为空则查询所有用户 |
| `startTime` | DATETIME | 查询起始时间 |
| `endTime` | DATETIME | 查询结束时间 |
| `opType` | INT | 操作类型代码，为空则查询所有类型 |

**返回列：**
| 列名 | 含义 |
|------|------|
| `userId` | 操作用户 |
| `timestamp` | 操作时间 |
| `opType` | 操作类型代码 |
| `opDetail` | 操作详情 |
| `clientIP` | 客户端 IP |

**常用 `opType` 值：**

| opType | 操作类型 | opDetail 含义 |
|--------|----------|---------------|
| 1 | 创建数据库 | `database=<dbPath>` |
| 2 | 删除数据库 | `database=<dbPath>` |
| 3 | 创建表 | `database=<dbPath>, table=<tbName>` |
| 4 | 删除表 | `database=<dbPath>, table=<tbName>` |
| 5 | 添加列 | `database=<dbPath>, table=<tbName>, column=<colName>, type=<type>` |
| 6 | 删除列 | `database=<dbPath>, table=<tbName>, column=<colName>` |
| 7 | 重命名列 | `database=<dbPath>, table=<tbName>, oldColumn=<old>, newColumn=<new>` |
| 10 | 添加分区 | `database=<dbPath>, partition=<partitionStr>` |
| 11 | 删除分区 | `database=<dbPath>, partition=<partitionStr>` |
| 20 | 用户登录 | `user=<userId>` |
| 21 | 用户登出 | `user=<userId>` |
| 30 | 创建用户 | `user=<userId>` |
| 31 | 删除用户 | `user=<userId>` |
| 32 | 修改密码 | `user=<userId>` |
| 40 | 创建函数视图 | `functionView=<name>` |
| 41 | 删除函数视图 | `functionView=<name>` |

**示例：**
```dolphindb
// 查询所有用户最近一天的审计日志
t = getAuditLog(, datetime(today()) - 1d, datetime(today()))
select * from t order by timestamp desc

// 查询特定用户的 DDL 操作
t = getAuditLog("alice", 2024.01.01T00:00:00, 2024.01.31T23:59:59)
select * from t

// 查询所有数据库删除操作（opType=2）
t = getAuditLog(, , , 2)
select userId, timestamp, opDetail from t

// 查询今天的登录记录（opType=20）
t = getAuditLog(, datetime(today()), datetime(today() + 1), 20)
select userId, timestamp, clientIP from t order by timestamp
```

---

## 五、日志文件位置与配置

### 日志文件路径

| 节点类型 | 默认路径 |
|----------|----------|
| 单节点 | `<HomeDir>/dolphindb.log` |
| Controller | `<HomeDir>/log/controller.log` |
| Agent | `<HomeDir>/log/agent.log` |
| DataNode | `<HomeDir>/log/<nodeAlias>.log` |

配置参数（`dolphindb.cfg`）：
```ini
# 日志文件路径
logFile=/path/to/dolphindb.log

# 日志级别（默认 INFO）
logLevel=INFO

# 日志文件最大大小（MB），超出时自动分割
maxLogSize=1024
```

### Shell 命令查看日志

```bash
# 实时追踪日志（推荐生产环境使用）
tail -f /path/to/dolphindb.log

# 过滤 ERROR 级别
grep "<ERROR>" /path/to/dolphindb.log

# 过滤特定时间段
grep "2024-01-15 09:" /path/to/dolphindb.log | grep "<ERROR>"

# 查看最后 200 行
tail -200 /path/to/dolphindb.log
```

---

## 六、日志监控方案（Loki + Promtail + Grafana）

适用于**多节点高可用集群**的轻量级日志监控方案。

### 架构

```
DolphinDB 节点日志 → Promtail（采集） → Loki（存储） → Grafana（可视化+告警）
```

### 告警规则 LogQL 速查

| 场景 | LogQL 查询语句 | 评估参数 |
|------|--------------|---------|
| 任意 ERROR 日志 | `count_over_time({job="dolphinDB"} \|= "ERROR"[5m])` | Every 1m / For 2m / Above 0 |
| 连接失败 | `count_over_time({job="dolphinDB"} \|~ "(?i)failed to connect"[5m])` | Every 30s / For 1m / Above 5 |
| OOM 异常 | `count_over_time({job="dolphinDB"} \|= "Out of memory"[5m])` | Every 1m / For 2m / Above 2 |
| 节点停机 | `count_over_time({job="dolphinDB"} \|= "MainServer shutdown."[5m])` | Every 15s / For 30s / Above 0 |
| 磁盘空间不足 | `count_over_time({job="dolphinDB"} \|~ "(?i)No space left on device"[5m])` | Every 1m / For 2m / Above 0 |
| Core 文件生成 | `count_over_time({job="core_files"}[5m])` | Every 1m / For 2m / Above 0 |
| 登录失败暴力破解 | `sum by (remoteIP) (count_over_time({job="dolphinDB"} \|~ "failed.*The user name or password is incorrect"\|logfmt\|remoteIP!=""[1h]))` | Every 1m / For 2m / Above 10 |
| Chunk 恢复失败 | `count_over_time({job="dolphinDB"} \|~ "(?i)failed to incrementally recover chunk"[1m])` | Every 15s / For 30s / Above 10 |
| 日志长时间无输出 | `count_over_time({job="dolphinDB"}[10m])` | Every 1m / For 2m / Below 1 |
| 心跳异常（网络波动）| `count_over_time({job="dolphinDB"} \|~ "(?i)HeartBeatSender exception"[3m])` | Every 15s / For 30s / Above 10 |

### 在 DolphinDB 中写入结构化日志供 Loki 检索

```dolphindb
// 写入带关键词的业务告警日志
writeLogLevel(ERROR, "wrong applseqnum: channel=1001, expected=500, received=502")

// 写入用户登录失败日志（格式需与 LogQL 匹配）
// 系统自动写入，格式：failed ... The user name or password is incorrect
```

### Promtail 日志标签解析配置

```yaml
pipeline_stages:
  - regex:
      expression: '^(?P<ts>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}\.\d+)\s(?P<level><\w+>)\s:(?P<message>.*)$'
  - timestamp:
      source: ts
      format: 2006-01-02 15:04:05.000000
      timezone: "China/Beijing"
  - labels:
      level:    # 将日志级别作为可过滤标签
  - output:
      source: message
```

解析后可按级别查询（注意：解析 level 标签后不能再以关键词搜索级别）：
```logql
# 正确（已配置 level 标签解析后）
count_over_time({job="dolphinDB", level="<ERROR>"}[5m])

# 错误（已配置 level 标签后不能用关键词）
count_over_time({job="dolphinDB"} |= "ERROR"[5m])
```

---

## 七、最佳实践

### 1. 分级记录，避免日志污染
```dolphindb
// ❌ 错误：所有信息都用同一级别
writeLog("开始处理")
writeLog("发生了一个小问题但不影响运行")
writeLog("严重错误，数据丢失！")

// ✅ 正确：区分不同级别
writeLogLevel(INFO, "开始处理")
writeLogLevel(WARNING, "发生了一个小问题但不影响运行")
writeLogLevel(ERROR, "严重错误，数据丢失！")
```

### 2. 日志内容结构化，便于检索
```dolphindb
// ✅ 包含关键字段：模块、操作、参数
writeLogLevel(ERROR, "DataLoader|load|file=" + path + ",error=" + msg)
writeLogLevel(INFO,  "StreamEngine|subscribe|topic=" + topic + ",start=" + string(startTime))
```

### 3. 生产环境日志级别管理
```dolphindb
// 生产环境默认 WARNING，问题排查时临时改为 DEBUG
setLogLevel("WARNING")    // 生产
setLogLevel("DEBUG")      // 排查时
setLogLevel("WARNING")    // 排查完恢复
```

### 4. 定期检查审计日志发现异常
```dolphindb
// 检查近7天是否有非预期的表删除操作
t = getAuditLog(, datetime(today()) - 7d, datetime(today()), 4)
if (t.size() > 0) {
    writeLogLevel(WARNING, "发现表删除操作: " + string(t.size()) + " 条")
}
```

### 5. DolphinDB 语言注意事项（避免常见错误）
```dolphindb
// ✅ 字符串拼接：用 string() 转换非字符串类型
writeLog("行数: " + string(1000))          // 正确
writeLog("时间: " + string(now()))          // 正确

// ❌ 错误：直接拼接数值会报类型错误
// writeLog("行数: " + 1000)               // 错误！

// ✅ 多参数写入多行（不是拼接）
writeLog("第一行", "第二行", "第三行")      // 写入3行独立日志
```
