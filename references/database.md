# DolphinDB 参考：数据库操作

> 来源：`db_distr_comp/cfg/db_intro.html`, `db_distr_comp/db/db_partitioning.html`

## 核心概念

DolphinDB 是分布式时序数据库，数据存储在**分布式文件系统（DFS）**中。

| 概念 | 说明 |
|------|------|
| DFS 路径 | `"dfs://库名"` 格式 |
| 分区 | 数据物理分割单元，每个分区存储在一个节点 |
| 存储引擎 | OLAP（列式）/ TSDB（时序）/ PKEY（主键）/ VectorDB / TextDB |
| chunk | 分布式数据块，分区的存储实体 |

---

## 存储引擎对比

| 引擎 | 特点 | 适用场景 |
|------|------|----------|
| OLAP | 列式存储，高压缩比 | 大规模历史数据分析 |
| TSDB | 时序优化，支持 sortKey | 时序数据（tick/bar/因子） |
| PKEY | 主键引擎，支持 upsert | 需要按主键更新的场景 |
| VectorDB | 向量存储，支持向量搜索 | AI/嵌入向量检索 |
| TextDB | 文本存储 | 文本检索场景 |

---

## 分区类型

### VALUE 分区（按值分区）
```dolphindb
// 按日期值分区
CREATE DATABASE "dfs://trade_db"
PARTITIONED BY VALUE(2020.01.01..2025.12.31)
```

### RANGE 分区（按范围分区）
```dolphindb
// 按价格范围分区
CREATE DATABASE "dfs://price_db"
PARTITIONED BY RANGE([0, 50, 100, 500, 2000])
```

### HASH 分区（按哈希分区）
```dolphindb
// 按 SYMBOL 哈希分为 10 个分区
CREATE DATABASE "dfs://sym_db"
PARTITIONED BY HASH([SYMBOL, 10])
```

### LIST 分区（按列表分区）
```dolphindb
// 按市场类型分区
CREATE DATABASE "dfs://mkt_db"
PARTITIONED BY LIST(["SH", "SZ", "BJ"])
```

### COMPO 组合分区（最常用）
```dolphindb
// 日期（一级）+ 股票 HASH（二级）
CREATE DATABASE "dfs://trade_db"
PARTITIONED BY VALUE(2020.01.01..2025.12.31), HASH([SYMBOL, 5])

// 日期（一级）+ 股票列表（二级）
CREATE DATABASE "dfs://astock"
PARTITIONED BY VALUE(2020.01M..2025.12M), HASH([SYMBOL, 20])
ENGINE='TSDB'
```

---

## 建库建表示例

### OLAP 引擎建表

```dolphindb
// 1. 创建数据库
CREATE DATABASE "dfs://trade_olap"
PARTITIONED BY VALUE(2020.01.01..2025.12.31), HASH([SYMBOL, 5])

// 2. 创建分区表
CREATE TABLE "dfs://trade_olap"."trades" (
    TradeTime  TIMESTAMP,
    SecurityID SYMBOL,
    Price      DOUBLE,
    Volume     LONG
) PARTITIONED BY TradeTime, SecurityID
```

### TSDB 引擎建表

```dolphindb
// 创建 TSDB 数据库
CREATE DATABASE "dfs://trade_tsdb"
PARTITIONED BY VALUE(2020.01.01..2025.12.31), HASH([SYMBOL, 5])
ENGINE='TSDB'

// 创建 TSDB 分区表（需指定 sortKey 或 sortColumns）
CREATE TABLE "dfs://trade_tsdb"."trades" (
    TradeTime  TIMESTAMP,
    SecurityID SYMBOL,
    Price      DOUBLE,
    Volume     LONG
) PARTITIONED BY TradeTime, SecurityID
SORTBY [SecurityID, TradeTime]   -- TSDB 排序列
KEEPDUPLICATES ALL               -- 重复处理方式（ALL/LAST/FIRST）
```

### 使用脚本 API 建库（等价方式）

```dolphindb
// 建数据库
db = database("dfs://trade_db",
    VALUE,
    2020.01.01..2025.12.31,
    engine="TSDB")

// 建分区表
t = table(1:0,
    `TradeTime`SecurityID`Price`Volume,
    [TIMESTAMP, SYMBOL, DOUBLE, LONG])

pt = db.createPartitionedTable(t, "trades", `TradeTime,
    sortColumns=`SecurityID`TradeTime)
```

---

## 数据写入

```dolphindb
// 基本写入（append!）
t = loadTable("dfs://trade_db", "trades")
data = table(
    2024.01.01T09:30:00.000 2024.01.01T09:30:01.000 AS TradeTime,
    `AAPL`MSFT AS SecurityID,
    150.0 380.0 AS Price,
    10000 5000 AS Volume
)
t.append!(data)

// upsert!（PKEY 引擎 / TSDB keepDuplicates=LAST 支持）
t.upsert!(data, ignoreNull=true, keyColNames=`SecurityID`TradeTime)

// SQL INSERT INTO
INSERT INTO loadTable("dfs://trade_db","trades")
VALUES (2024.01.01T09:30:00.000, "AAPL", 150.0, 10000)

// ── CSV 导入 ──────────────────────────────────────────────
// loadText：导入为内存表（适合小文件）
t = loadText(
    filename="/data/trades.csv",
    [delimiter=","],           // 分隔符（默认逗号）
    [schema],                  // 自定义 schema 表（见下）
    [skipRows=0],              // 跳过头部行数
    [transform]                // 可选数据变换函数
)

// loadTextEx：直接写入分布式表（推荐大文件）
loadTextEx(
    dbHandle=database("dfs://trade_db"),
    tableName="trades",
    partitionColumns=`TradeDate`SecurityID,
    filename="/data/trades.csv",
    [delimiter=","],
    [schema],
    [skipRows=1],              // 跳过标题行
    [sortColumns=`SecurityID`TradeTime]  // TSDB 时需要
)

// ── Schema 自定义（解决类型推断错误）──────────────────────
// 先让系统自动推断一次
schema = extractTextSchema("/data/trades.csv")
// 手动修正列类型（修改 type 列）
update schema set type="SYMBOL" where name="SecurityID"
update schema set type="TIMESTAMP" where name="TradeTime"
update schema set type="DOUBLE" where name="Price"
// 再用修正后的 schema 加载
loadTextEx(database("dfs://trade_db"), "trades", `TradeDate, "/data/trades.csv", schema=schema)

// ── 常见陷阱 ──────────────────────────────────────────────
// 1. 编码问题（GBK → UTF-8 转换）
//    Linux: iconv -f GBK -t UTF-8 input.csv > output.csv
//    或在 loadText 后手动处理字符串列

// 2. NULL 值处理（CSV 中的空格或特定字符）
//    默认空字符串被识别为 NULL，如需保留：
//    update schema set format="%s" where name="StringCol"

// 3. 日期格式（非标准格式需指定 format）
//    update schema set format="MM/dd/yyyy" where name="TradeDate"

// 4. 跳过表头（若 CSV 第一行是列名）
//    skipRows=1（不填 schema 时 DolphinDB 会自动把第一行当列名）

// ── 多文件批量导入 ──────────────────────────────────────────
fileList = files("/data/2024/").filename   // 列出目录下文件
each(def(f){
    loadTextEx(database("dfs://trade_db"), "trades",
               `TradeDate`SecurityID, "/data/2024/" + f)
}, fileList)
```

---

## 数据查询

```dolphindb
// 加载表并查询
t = loadTable("dfs://trade_db", "trades")
select * from t limit 10

// 带分区修枝的查询（利用分区加速）
select avg(Price) from t
where TradeTime >= 2024.01.01 and TradeTime < 2024.12.31
group by SecurityID

// 分区信息
schema(t).partitionColumnName
schema(t).chunkPath
```

---

## 内存表类型

| 类型 | 函数 | 说明 |
|------|------|------|
| 普通内存表 | `table(...)` | 无主键，可重复 |
| 键控表 | `keyedTable(keyCol, ...)` | 主键唯一，自动 upsert |
| 索引表 | `indexedTable(...)` | 按列索引，快速查找 |
| 流数据表 | `streamTable(...)` | 专用于流计算 |
| 持久化流表 | `enableTableShareAndPersist(...)` | 支持持久化的共享流表 |

```dolphindb
// 键控表（keyedTable）— 自动按主键 upsert
kt = keyedTable(`id, 1000:0, `id`val`ts, [INT, DOUBLE, TIMESTAMP])
kt.append!(table(1 2 3 AS id, 0.1 0.2 0.3 AS val, now()-1..3 AS ts))

// 索引表（indexedTable）
it = indexedTable(`id`date, 1000:0, 
    `id`date`val, [SYMBOL, DATE, DOUBLE])
```

---

## 数据迁移与备份

```dolphindb
// ODBC 导入（mysql 等）
loadPlugin("mysql")
conn = mysql::connect("127.0.0.1", 3306, "root", "password", "mydb")
data = mysql::load(conn, "SELECT * FROM trades")

// 备份数据库
backup("/backup/path", <select * from loadTable("dfs://trade_db","trades")>, force=true)

// 恢复
restore("/backup/path", "dfs://trade_db", "trades", %)
```
