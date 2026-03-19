# DolphinDB 参考：MCP（Model Context Protocol）

> 来源：`mcp/mcp_introduction.html`

## 概述

DolphinDB MCP（Model Context Protocol）让 AI 模型（如 Claude、GPT 等）能够通过标准化协议访问和操作 DolphinDB 数据库。

---

## MCP 服务安装

### 方式一：通过 npm 安装（推荐）

```bash
npm install -g @dolphindb/mcp-server
```

### 方式二：通过 Python pip 安装

```bash
pip install dolphindb-mcp
```

### 方式三：查看 DolphinDB MCP 源码

```
https://github.com/dolphindb/dolphindb-mcp
```

---

## MCP 配置

### 在 Claude Desktop 中配置

编辑 `claude_desktop_config.json`（MacOS: `~/Library/Application Support/Claude/`，Windows: `%APPDATA%\Claude\`）：

```json
{
  "mcpServers": {
    "dolphindb": {
      "command": "dolphindb-mcp",
      "args": [],
      "env": {
        "DDB_HOST": "localhost",
        "DDB_PORT": "8848",
        "DDB_USER": "admin",
        "DDB_PASSWORD": "123456"
      }
    }
  }
}
```

### 在 Cursor / VS Code 中配置

```json
{
  "mcp": {
    "servers": {
      "dolphindb": {
        "command": "dolphindb-mcp",
        "env": {
          "DDB_HOST": "localhost",
          "DDB_PORT": "8848",
          "DDB_USER": "admin",
          "DDB_PASSWORD": "123456"
        }
      }
    }
  }
}
```

---

## MCP 工具（Tools）

DolphinDB MCP 服务暴露以下工具供 AI 调用：

| 工具名 | 说明 |
|--------|------|
| `execute_script` | 执行 DolphinDB 脚本，返回结果 |
| `query_table` | 查询分布式表（SQL 语句） |
| `list_databases` | 列出所有数据库 |
| `list_tables` | 列出指定数据库中的表 |
| `table_schema` | 获取表结构（列名/类型/分区信息） |
| `get_sample_data` | 获取表前 N 行样本数据 |
| `upload_data` | 上传 DataFrame/表格数据到 DolphinDB |
| `get_cluster_info` | 获取集群节点状态 |

---

## 使用示例

**对 AI 说：**
> "查看 dfs://trade_db 中 trades 表的结构，并统计每只股票最近 7 天的平均价格"

**AI 会通过 MCP 调用：**
1. `table_schema("dfs://trade_db", "trades")` — 获取表结构
2. `query_table("SELECT SecurityID, avg(Price) as avgPrice FROM loadTable('dfs://trade_db','trades') WHERE TradeTime >= today()-7 GROUP BY SecurityID")` — 查询统计

---

## 自定义 MCP 服务（Python）

```python
from mcp.server import Server
from mcp.types import Tool
import dolphindb as ddb

server = Server("dolphindb-custom-mcp")

@server.tool()
async def execute_ddb(script: str) -> str:
    """执行 DolphinDB 脚本"""
    s = ddb.session()
    s.connect("localhost", 8848, "admin", "123456")
    result = s.run(script)
    s.close()
    return str(result)

if __name__ == "__main__":
    server.run()
```
