# DolphinDB 参考：系统运维

> 来源：`sys_man/om_intro.html`，系统管理文档

## 用户与权限管理

```dolphindb
// 创建用户
createUser("alice", "passwdXxx")

// 创建用户组
createGroup("analysts", ["alice", "bob"])

// 授予权限
grant("alice", TABLE_READ, "dfs://trade_db/trades")
grant("analysts", DB_READ, "dfs://trade_db")
grant("alice", SCRIPT_EXEC, "%")      // 允许执行脚本

// 撤销权限
revoke("alice", TABLE_READ, "dfs://trade_db/trades")

// 查看用户权限
getUserAccess("alice")

// 删除用户
deleteUser("alice")

// 修改密码
changePassword("alice", "newpass")

// 以 admin 重置密码
resetPwd("alice", "newpass")
```

### 权限类型速查

| 权限常量 | 粒度 | 说明 |
|----------|:----:|------|
| `TABLE_READ` | 表 | 查询数据 |
| `TABLE_WRITE` | 表 | append!/insert into |
| `TABLE_UPDATE` | 表 | update 语句 |
| `TABLE_DELETE` | 表 | delete 语句 |
| `TABLE_INSERT` | 表 | insert into（细粒度） |
| `DB_READ` | 库 | 读该库所有表 |
| `DB_WRITE` | 库 | 写该库所有表 |
| `DB_CREATE` | 库 | 在库中建表 |
| `DB_DELETE` | 库 | 删库/删表 |
| `DB_MANAGE` | 库 | 库管理（含 DDL） |
| `SCRIPT_EXEC` | 全局 | 执行任意脚本 |
| `TEST_EXEC` | 全局 | 执行测试脚本 |
| `VIEW_EXEC` | 视图 | 执行指定视图 |
| `FUNCTION_EXEC` | 函数 | 执行指定函数视图 |
| `COMPUTE_GROUP_EXEC` | 计算组 | 使用计算组 |

### 权限通配符与批量授权

```dolphindb
// "%" 通配符：授予对所有对象的权限
grant("alice", TABLE_READ, "%")       // 所有表可读
grant("alice", DB_WRITE, "%")         // 所有库可写

// 指定单表
grant("alice", TABLE_READ, "dfs://trade_db/trades")

// 指定整个库（库下所有表）
grant("alice", DB_READ, "dfs://trade_db")

// 函数视图权限（最小权限原则）
// 1. 先定义函数视图
addFunctionView(getTrades)    // 向系统注册函数视图
// 2. 授予用户执行该视图的权限（用户无需 TABLE_READ）
grant("alice", VIEW_EXEC, "getTrades")

// 用户组批量管理
grant("analysts", TABLE_READ, "dfs://trade_db/%")  // 分析员组全部表可读
revoke("analysts", TABLE_WRITE, "%")               // 撤销所有写权限
```

### 权限诊断函数

```dolphindb
// 查看用户已有权限
getUserAccess("alice")

// 列出所有用户
getUserList()

// 列出所有用户组
getGroupList()

// 查看用户组成员
exec member from getGroupList() where groupID="analysts"

// 查看函数视图列表
getFunctionViews()

// 测试某用户是否有权限（模拟该用户执行）
// 用 rpc 以该用户身份执行，观察是否报 Not allowed 错误
```

> **⚠️ 常见权限报错**：
> - `Not allowed to read...` → 缺少 `TABLE_READ`，检查 `getUserAccess()`
> - `No privilege to create...` → 缺少 `DB_CREATE` 或 `DB_MANAGE`
> - `Permission denied` on script → 缺少 `SCRIPT_EXEC`，考虑改用函数视图

---

## 作业与任务管理

```dolphindb
// 查询当前作业
getJobStatus()

// 查看历史作业
getRecentJobs()

// 取消作业
cancelJob("jobId")

// 批处理作业（后台运行）
submitJob("myJob", "每日计算因子", calcFactorDef)

// 定时作业（Scheduler）
// 每天 9:00 执行
scheduleJob("dailyCalc", "每日计算", factorCalcFunc, 
    09:00m, 2024.01.01, 2099.12.31, 'D')

// 查看定时作业
getScheduledJobs()

// 取消定时作业
deleteScheduledJob("dailyCalc")
```

---

## 监控与诊断

```dolphindb
// 查看节点状态
getClusterDFSNodes()

// 查看当前内存使用
getSessionMemoryStat()

// 查看表的内存占用
objs(true)

// 查看正在运行的查询
getQueryStatus()

// 查看流数据订阅状态
getStreamingStat()

// 查看持久化流表状态
getPersistenceTableMap()

// 查看 DFS 使用情况
getDFSStat()

// 查看分区信息
getTabletsMeta("<dfs://trade_db>", "<trades>", true)
```

---

## 性能优化

```dolphindb
// 清空 chunk 缓存（强制刷盘）
flushOLAPCache()
flushTSDBCache()

// 清空查询缓存
clearCachedDatabase()

// 查看慢查询日志
getSlowQueryLog()

// 触发 GC
gc()

// 查看 session 变量
objs()

// 释放大变量
undef("bigData")
```

---

## 备份与恢复

```dolphindb
// 备份整张表
backup(
    backupDir="/backup/2024",
    sqlObj=<select * from loadTable("dfs://trade_db","trades")>,
    force=true
)

// 按分区备份
backup(
    backupDir="/backup/2024",
    sqlObj=<select * from loadTable("dfs://trade_db","trades") 
            where TradeTime between 2024.01.01 and 2024.01.31>,
    force=true
)

// 恢复数据
restore(
    backupDir="/backup/2024",
    dbPath="dfs://trade_db",
    tableName="trades",
    partition="%",    // 恢复全部分区
    force=true
)

// 查看备份信息
backupStatus("/backup/2024")
```

---

## 故障排查

```dolphindb
// 查看错误日志（最近 N 行）
getServerLog(100)

// 查看节点日志
rpc("datanode1", getServerLog, 50)

// 检查分区完整性
checkPartitionData("dfs://trade_db", "trades")

// 查看死锁信息
getTransactionMeta()

// 强制释放表锁
releaseTableLock("dfs://trade_db", "trades")

// 检查 chunk 状态
getChunksMeta()
```

### 常见错误及处理

| 错误 | 可能原因 | 解决方式 |
|------|----------|---------|
| Out of memory | maxMemSize 太小 | 调大 maxMemSize 或分批查询 |
| partition 数量超限 | maxPartitionNumPerQuery 过小 | 调大该参数或添加分区过滤 |
| License expired | License 过期 | 更新 dolphindb.lic 文件 |
| Connection refused | 端口未开放或节点未启动 | 检查防火墙和节点状态 |
| Symbol size exceeded | SYMBOL 分区超 2^21 | 减少分区内去重值 |
