# DolphinDB 技能库 v3.x

> **版本**: 3.x（破坏式更新） | **DolphinDB 版本**: 3.00.4+ | **分支**: v3-dev

专为 AI 助手和开发者设计的 DolphinDB 完整知识库。从脚本语法到生产运维，从流计算到量化回测，一站式解决所有 DolphinDB 相关问题。

---

## 🔄 v3.x 重大变更

**破坏式更新警告**：v3.x 与 v2.0.0 不兼容！

| 对比项 | v2.0.0 | v3.x |
|--------|--------|------|
| 组织方式 | 1493 个文档堆放 | 20+ 个模块化知识文件 |
| 导航方式 | 按 ID 查找 | 按问题场景定位 |
| 内容深度 | API 参考为主 | 实战导向 + 陷阱提示 |
| 新增内容 | - | Python/JS 差异对比、错误码排查、日志监控 |

**获取 v2.0.0**：`git checkout v2.0.0`

---

## 📁 模块化知识导航

### 🎯 按问题场景快速定位

| 你遇到的问题 | 首选文档 |
|-------------|---------|
| **DolphinDB 脚本语法 / 与 Python 有何不同** | [programming.md](references/programming.md) ⭐ |
| **报错了 / 日志里有 S0XXXX 错误码** | [error_codes.md](references/error_codes.md) ⭐ |
| **如何写日志 / 配置日志级别 / 审计日志** | [logging.md](logging.md) ⭐ |
| **写流计算 / K 线合成 / 实时因子** | [streaming.md](references/streaming.md) + [tutorials_streaming.md](references/tutorials_streaming.md) |
| **量化回测策略开发** | [backtest.md](references/backtest.md) |
| **Python 连接 / 上传数据** | [python_api.md](references/python_api.md) |
| **SQL GROUP BY / CONTEXT BY / PIVOT BY** | [sql.md](references/sql.md) |
| **建库建表 / 分区策略 / TSDB vs OLAP** | [database.md](references/database.md) + [tutorials_database.md](references/tutorials_database.md) |
| **用户权限 / 作业调度 / 集群监控** | [system_admin.md](references/system_admin.md) |
| **安装 / 集群部署 / Docker** | [deployment.md](references/deployment.md) |
| **Kafka / MySQL 等插件** | [plugins.md](references/plugins.md) |
| **函数名忘了 / 找内置函数** | [functions_overview.md](references/functions_overview.md) |
| **OOM / 节点崩溃 / 启动失败** | [tutorials_ops.md](references/tutorials_ops.md) |
| **因子计算 / 窗口函数 / 面板数据** | [tutorials_quant.md](references/tutorials_quant.md) |

### 📚 完整文档分类

#### 一、核心语言

| 文件 | 大小 | 内容 |
|------|------|------|
| [programming.md](references/programming.md) | 15KB | ⭐ 数据类型、字典 API、向量操作、与 Python 差异对比 12 大陷阱 |
| [sql.md](references/sql.md) | 5.6KB | SELECT/GROUP BY/context by/pivot by/ASOF JOIN/UPDATE/CREATE TABLE |
| [functions_overview.md](references/functions_overview.md) | 5.4KB | 数学/字符串/时间/向量/表/数据库/流数据/系统函数速查表 |
| [modules.md](references/modules.md) | 3KB | use/include/loadModule、内置模块、自定义模块开发 |

#### 二、流计算与回测

| 文件 | 大小 | 内容 |
|------|------|------|
| [streaming.md](references/streaming.md) | 6.7KB | ⭐ 发布订阅、5 类流引擎完整代码示例、流批一体 |
| [backtest.md](references/backtest.md) | 6.2KB | ⭐ 回测插件架构、策略回调函数、下单 API、模拟撮合引擎 |

#### 三、API 连接器

| 文件 | 大小 | 内容 |
|------|------|------|
| [python_api.md](references/python_api.md) | 5.7KB | ⭐ Session 连接、数据上传、tableAppender、流订阅、类型映射 |
| [cpp_api.md](references/cpp_api.md) | 7.5KB | 编译配置、数据类型操作、MultithreadedTableWriter、流订阅 |
| [api_connectors.md](references/api_connectors.md) | 4.3KB | Java JDBC、多语言 API 概览 |

#### 四、数据库与运维

| 文件 | 大小 | 内容 |
|------|------|------|
| [quick_start.md](references/quick_start.md) | 3.7KB | 安装→建库→建表→写入→查询完整入门 |
| [database.md](references/database.md) | 5.5KB | 分区类型、OLAP/TSDB/PKEY 引擎、内存表种类 |
| [deployment.md](references/deployment.md) | 4.3KB | 单节点/集群/Docker/HA 部署 |
| [configuration.md](references/configuration.md) | 3.9KB | 配置参数速查、生产/开发环境模板 |
| [system_admin.md](references/system_admin.md) | 3.8KB | 用户权限、作业调度、监控、备份恢复 |
| [logging.md](logging.md) | 11KB | ⭐ writeLog 完整用法、Loki+Promtail+Grafana 监控告警 |
| [error_codes.md](references/error_codes.md) | 14KB | ⭐ 错误码体系、5 大高频故障排查、具体解决方法 |

#### 五、插件与其他

| 文件 | 大小 | 内容 |
|------|------|------|
| [plugins.md](references/plugins.md) | 3.7KB | Kafka/MySQL/ODBC/HTTP/Parquet 插件 |
| [mcp.md](references/mcp.md) | 2.7KB | DolphinDB MCP 配置（AI 工具接入） |

#### 六、深度教程（官方教程转换）

| 文件 | 大小 | 内容 |
|------|------|------|
| [tutorials_streaming.md](references/tutorials_streaming.md) | 152KB | ⭐ K 线合成/响应式引擎/CEP/流聚合完整实战 |
| [tutorials_database.md](references/tutorials_database.md) | 162KB | ⭐ TSDB 引擎深度、分区存储最佳实践 |
| [tutorials_quant.md](references/tutorials_quant.md) | 224KB | ⭐ 因子计算、窗口函数、SQL 实战、函数式编程 |
| [tutorials_ops.md](references/tutorials_ops.md) | 130KB | ⭐ OOM 处理、内存管理、节点崩溃恢复 |

---

## 🚀 快速代码模板

### 连接（Python）

```python
import dolphindb as ddb
s = ddb.session()
s.connect("localhost", 8848, "admin", "123456")
```

### 建库建表（TSDB + 组合分区）

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

### 查询（DolphinDB 特有语法）

```dolphindb
-- context by（按组保留行，类似窗口函数）
select SecurityID, TradeTime, Price, mavg(Price, 5) as ma5
from loadTable("dfs://mydb", "trades")
where TradeTime >= 2024.01.01
context by SecurityID

-- PIVOT BY（二维交叉表）
select Price from t
pivot by TradeTime, SecurityID
```

### 流数据订阅

```dolphindb
subscribeTable(tableName="trades", actionName="demo",
    handler=outputTable, msgAsTable=true, batchSize=1000)
```

---

## ⚠️ DolphinDB 编程陷阱

DolphinDB 是小众专业语言，**不能套用 Python/JavaScript 的习惯**！

| 陷阱 | Python 风格 | DolphinDB 正确写法 |
|------|------------|-------------------|
| 字典方法 | `d.get(key)` | `find(d, key)` |
| 向量切片 | `v[0:3]` | `v[0..2]` 或 `subarray(v, 0, 3)` |
| 比较运算 | `=` 是比较 | `==` 才是比较 |
| NULL 处理 | 自动传播 | 聚合函数自动忽略 NULL |
| 类型差异 | `"AAPL"` 字符串 | `` `AAPL `` 是 SYMBOL 类型 |

详见 → **[programming.md](references/programming.md)** 完整对比

---

## 🔧 AI 助手集成

本技能库专为 AI 助手（Claude Code、GitHub Copilot、Cursor 等）设计，支持自然语言问答：

**示例提问**：
- "帮我写一个实时 K 线合成的脚本"
- "DolphinDB 字典怎么遍历？"
- "报错 S0003 是什么意思？"
- "Python 怎么批量写入数据到 DolphinDB？"

**激活条件**：
- 提及 DolphinDB、DDB、时序数据库
- 使用 DolphinDB 函数（writeLog/subscribeTable/createTimeSeriesEngine/loadTable 等）
- 涉及流计算、量化回测、集群运维

---

## 📊 版本历史

| 版本 | 分支 | 说明 |
|------|------|------|
| **v3.x** | `v3-dev` | 破坏式更新：模块化重构、实战导向 |
| **v2.0.0** | `main` | 稳定版：1493 个文档 + 3 份白皮书 |

**切换版本**：
```bash
# 查看 v3.x 开发版
git checkout v3-dev

# 回到 v2.0.0 稳定版
git checkout v2.0.0
```

---

## 🔗 相关资源

- **官方文档**: https://docs.dolphindb.cn
- **官网**: https://www.dolphindb.com
- **社区**: https://ask.dolphindb.cn/
- **GitHub**: https://github.com/dolphindb

---

**维护者**: Hugo | **许可**: MIT | **DolphinDB 版本**: 3.00.4+
