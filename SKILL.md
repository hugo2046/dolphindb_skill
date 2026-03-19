---
name: DolphinDB
description: >
  DolphinDB 高性能时序数据库完整技术技能。涵盖 DolphinDB 脚本语言（含 Python/Java/C++ 差异陷阱）、SQL 特有语法（context by/pivot by/asof join）、分布式分区数据库、流计算引擎（TimeSeriesEngine/ReactiveStateEngine/CEP）、量化回测引擎、Python API/C++ API、MCP 接入、Kafka 等插件、错误码排查、日志配置、集群运维等所有 DolphinDB 相关问题。
  【重要】每当用户提及 DolphinDB、问 DDB、问时序数据库写代码/报错/配置、或使用 writeLog/subscribeTable/createTimeSeriesEngine/loadTable 等 DolphinDB 函数时，必须激活本技能。即使用户没有明确说「使用 DolphinDB 技能」，只要问题与 DolphinDB 脚本、API、部署、运维、量化有关，都应激活。
version: "3.x"
---

# DolphinDB 技能

## ⚠️ 核心注意事项

DolphinDB 是小众专业语言，**不能套用 Python/JavaScript 的习惯**！常见陷阱：
- 字典无 `.get()` / `.items()` / `.keys()`，用 `find(d, key)` / `keys(d)` / `values(d)`
- 向量没有切片语法 `v[0:3]`，用 `v[0..2]` 或 `subarray(v, 0, 3)`
- `=` 在非 SQL 环境是赋值，比较用 `==`
- NULL 会传播，但聚合函数（sum/avg）自动忽略 NULL
- 函数内变量是局部的，修改参数不影响外部（值传递）
- SYMBOL 类型（反引号 `` `AAPL ``）≠ STRING 类型（`"AAPL"`）

详见 → **[编程语言深度指南](references/programming.md)**（含 Python/JS 差异对比 12 大陷阱）

---

## 📁 参考文件导航（按 6 大类）

### 一、原生语言语法

| 文件 | 内容 |
|------|------|
| **[programming.md](references/programming.md)** | ⭐ 数据类型、字典完整API、向量操作、控制语句、函数定义、函数化编程、与Python差异对比 |
| **[sql.md](references/sql.md)** | SELECT/GROUP BY/context by/pivot by/ASOF JOIN/UPDATE/CREATE TABLE |
| **[functions_overview.md](references/functions_overview.md)** | 数学/字符串/时间/向量/表/数据库/流数据/系统函数速查表、窗口函数、TA函数 |
| **[modules.md](references/modules.md)** | use/include/loadModule、内置模块(ta/mytt/wq101alpha/ops)、自定义模块开发 |

### 二、流计算引擎

| 文件 | 内容 |
|------|------|
| **[streaming.md](references/streaming.md)** | ⭐ 发布订阅、5类流引擎(时序/响应式/横截面/会话/CEP)完整代码示例、流批一体 |

### 三、回测引擎

| 文件 | 内容 |
|------|------|
| **[backtest.md](references/backtest.md)** | ⭐ 回测插件架构、策略5大回调函数、下单API、结果分析、模拟撮合引擎(MES) |

### 四、Python API

| 文件 | 内容 |
|------|------|
| **[python_api.md](references/python_api.md)** | ⭐ Session连接、执行脚本、上传数据、tableAppender/tableUpsert、SQL查询、流订阅、连接池、Python↔DolphinDB类型映射 |

### 五、C++ API

| 文件 | 内容 |
|------|------|
| **[cpp_api.md](references/cpp_api.md)** | 编译配置、连接执行、数据类型(创建/读取)、建表写入、MultithreadedTableWriter、StreamingClient流订阅 |

### 六、插件与其他

| 文件 | 内容 |
|------|------|
| **[plugins.md](references/plugins.md)** | Kafka/MySQL/ODBC/HTTP Client/Parquet等插件安装与使用 |
| **[api_connectors.md](references/api_connectors.md)** | Java JDBC连接器、多语言API概览 |
| **[mcp.md](references/mcp.md)** | DolphinDB MCP配置（Claude/Cursor等AI工具接入） |

### 七、深度教程（完整转换自官方 HTML）

| 文件 | 内容 |
|------|------|
| **[tutorials_streaming.md](references/tutorials_streaming.md)** | ⭐ K线合成/响应式引擎/CEP入门/流聚合器完整实战教程 |
| **[tutorials_database.md](references/tutorials_database.md)** | ⭐ TSDB引擎深度解析、分区存储最佳实践、内存表使用 |
| **[tutorials_quant.md](references/tutorials_quant.md)** | ⭐ 因子计算最佳实践、窗口函数精讲、SQL实战案例、函数式编程案例 |
| **[tutorials_ops.md](references/tutorials_ops.md)** | OOM处理、内存管理、节点启动异常、崩溃恢复完整指南 |

### 附：数据库与运维

| 文件 | 内容 |
|------|------|
| **[quick_start.md](references/quick_start.md)** | 安装→建库→建表→写入→查询的完整入门流程 |
| **[database.md](references/database.md)** | 分区类型、OLAP/TSDB/PKEY引擎建表、内存表种类 |
| **[deployment.md](references/deployment.md)** | 单节点/单机集群/多机/Docker/HA部署 |
| **[configuration.md](references/configuration.md)** | 所有配置参数速查，生产/开发环境模板 |
| **[system_admin.md](references/system_admin.md)** | 用户权限、作业调度、监控、备份恢复、故障排查 |
| **[logging.md](references/logging.md)** | ⭐ writeLog/writeLogLevel/setLogLevel/getAuditLog 完整用法、日志格式、Loki+Promtail+Grafana 监控告警方案、LogQL 告警规则速查 |
| **[error_codes.md](references/error_codes.md)** | ⭐ 错误码体系（S00-S06）、5大高频故障排查、具体错误码原因+解决方法、调试命令速查 |

---

## 🧭 按问题场景快速选参考文件

| 我的问题是… | 先读这个文件 |
|-------------|-------------|
| DolphinDB 字典/向量/字符串用法，与 Python 有何区别 | **[programming.md](references/programming.md)** |
| 报错了，日志里有 `S0XXXX` 错误码 | **[error_codes.md](references/error_codes.md)** |
| 如何写日志 / 配置日志级别 / 审计日志 | **[logging.md](references/logging.md)** |
| 写流计算 / K 线合成 / 实时因子 | **[streaming.md](references/streaming.md)** + **[tutorials_streaming.md](references/tutorials_streaming.md)** |
| 量化回测策略开发 | **[backtest.md](references/backtest.md)** |
| Python 连接 DolphinDB / 上传数据 | **[python_api.md](references/python_api.md)** |
| SQL GROUP BY / CONTEXT BY / PIVOT BY | **[sql.md](references/sql.md)** |
| 建库建表 / 分区策略 / TSDB vs OLAP | **[database.md](references/database.md)** + **[tutorials_database.md](references/tutorials_database.md)** |
| 用户权限 / 作业调度 / 集群监控 | **[system_admin.md](references/system_admin.md)** |
| 安装/集群部署/Docker | **[deployment.md](references/deployment.md)** |
| Kafka/MySQL 等插件 | **[plugins.md](references/plugins.md)** |
| 函数名忘了 / 找内置函数 | **[functions_overview.md](references/functions_overview.md)** |
| OOM / 节点崩溃 / 启动失败 | **[tutorials_ops.md](references/tutorials_ops.md)** |
| 因子计算 / 窗口函数 / 面板数据 | **[tutorials_quant.md](references/tutorials_quant.md)** |

---

## 快速代码模板

### 连接（DolphinDB 原生客户端）

```dolphindb
// 通过终端或 VS Code 扩展连接
// 服务端地址：localhost:8848，账密：admin/123456
```

### 连接（Python）

```python
import dolphindb as ddb
s = ddb.session()
s.connect("localhost", 8848, "admin", "123456")
```

### 建库建表

```dolphindb
CREATE DATABASE "dfs://mydb"
PARTITIONED BY VALUE(2020.01.01..2025.12.31), HASH([SYMBOL, 5])
ENGINE='TSDB'

CREATE TABLE "dfs://mydb"."trades" (
    TradeTime  TIMESTAMP,
    SecurityID SYMBOL,
    Price      DOUBLE,
    Volume     LONG
) PARTITIONED BY TradeTime, SecurityID
SORTBY [SecurityID, TradeTime]
```

### 写入数据

```dolphindb
t = loadTable("dfs://mydb", "trades")
t.append!(data)
```

### 查询（含 DolphinDB 特有语法）

```dolphindb
// 带 context by（按组保留行，类似窗口函数）
select SecurityID, TradeTime, Price, mavg(Price, 5) as ma5
from loadTable("dfs://mydb", "trades")
where TradeTime >= 2024.01.01
context by SecurityID

// PIVOT BY（二维交叉表）
select Price from t
pivot by TradeTime, SecurityID
```

### 流数据

```dolphindb
// 订阅发布
subscribeTable(tableName="trades", actionName="demo",
    handler=outputTable, msgAsTable=true, batchSize=1000)
```
