# DolphinDB 参考：SQL 语句

> 来源：`progr/sql/sql_intro.html`, `progr/sql/select.html`, `progr/sql/contextBy.html`, `progr/sql/pivotBy.html`

## SQL 基础语法

DolphinDB 的 SQL 在标准 SQL 基础上扩展了 `context by`、`pivot by`、`csort` 等特有子句，且**大小写不敏感**（SQL 关键词）但表列名大小写敏感。

---

## SELECT 语句

### 基础查询

```sql
-- 查询所有列
SELECT * FROM t

-- 查询指定列
SELECT col1, col2 FROM t

-- 条件筛选
SELECT * FROM t WHERE price > 100 AND volume > 1000

-- 排序
SELECT * FROM t ORDER BY price ASC, volume DESC

-- 限制行数（前 N 行）
SELECT * FROM t LIMIT 10

-- 限制行数（从 offset 开始）
SELECT * FROM t LIMIT 10 OFFSET 100
```

### 聚合与分组

```sql
-- 聚合
SELECT avg(price), max(volume), count(*) FROM t

-- group by
SELECT SecurityID, avg(price), sum(volume)
FROM t
GROUP BY SecurityID

-- having
SELECT SecurityID, avg(price) AS avgPrice
FROM t
GROUP BY SecurityID
HAVING avgPrice > 50

-- group by 带时间分组（bar 函数）
SELECT bar(TradeTime, 1m) AS minute, avg(price)
FROM t
GROUP BY bar(TradeTime, 1m)
```

### 条件操作

```sql
-- IN
WHERE SecurityID IN (`AAPL, `MSFT, `GOOG)

-- BETWEEN
WHERE TradeTime BETWEEN 2024.01.01 AND 2024.12.31

-- IS NULL / IS NOT NULL
WHERE price IS NOT NULL

-- LIKE（字符串模式匹配，支持%和_通配符）
WHERE name LIKE 'Apple%'

-- case when
SELECT CASE
    WHEN price > 100 THEN 'high'
    WHEN price > 50 THEN 'mid'
    ELSE 'low'
END AS level
FROM t
```

---

## DolphinDB 特有子句

### context by（按组保留行数）

`context by` 对数据按组计算但**保留原始行结构**（类似 SQL 窗口函数，但更简洁）：

```sql
-- 在每组内计算移动平均（保留每行）
SELECT SecurityID, TradeTime, price,
       mavg(price, 5) AS ma5
FROM t
CONTEXT BY SecurityID

-- 按组取最近 2 行（负数表示倒数）
SELECT * FROM t CONTEXT BY SecurityID LIMIT -2

-- 按组排序内取前 3
SELECT * FROM t CONTEXT BY SecurityID CSORT price DESC LIMIT 3
```

#### `context by` vs `group by` 函数行为速查

| 函数 | `group by` 行为 | `context by` 行为 |
|------|----------------|------------------|
| `avg(price)` | 标量（每组一行） | ✅ 每行返回组内均值 |
| `sum(volume)` | 标量（每组一行） | ✅ 每行返回组内累积总和 |
| `mavg(price, 5)` | ❌ 不适用 | ✅ 每行返回 5 周期移动均值 |
| `msum(volume, 10)` | ❌ 不适用 | ✅ 每行返回 10 周期移动和 |
| `mcorr(a, b, 20)` | ❌ 不适用 | ✅ 每行返回 20 周期滚动相关系数 |
| `cumsum(volume)` | ❌ 不适用 | ✅ 每行返回组内截至当前的累积和 |
| `rank(price)` | ❌ 不适用 | ✅ 每行返回组内排名（从 0 开始） |
| `percentile(price, 95)` | 标量 | ✅ 每行返回组内 95 分位数 |
| `first(price)` | 标量（组首值） | ✅ 每行返回组内第一个值（常量列） |
| `count(*)` | 标量（组行数） | ✅ 每行返回组内行数 |

> ⚠️ **核心区别**：`group by` 把每组压缩为一行；`context by` 保留所有行，仅添加组内计算结果列。



### pivot by（二维交叉表）

```sql
-- 行 = TradeTime，列 = SecurityID，值 = price
SELECT price
FROM t
PIVOT BY TradeTime, SecurityID

-- 行 = date，列 = factor，值 = val
SELECT val FROM factor_table
PIVOT BY date, factor
```

### window（窗口函数）

```sql
-- 排名
SELECT SecurityID, price,
       rank(price) OVER (PARTITION BY date ORDER BY price DESC) AS rnk
FROM t
```

---

## INSERT / UPDATE / DELETE

```sql
-- INSERT INTO（追加数据，分布式表使用 append!）
INSERT INTO t VALUES (2024.01.01, `AAPL, 150.0, 10000)

-- 批量插入
INSERT INTO t VALUES
    (2024.01.01, `AAPL, 150.0, 10000),
    (2024.01.01, `MSFT, 380.0, 5000)

-- UPDATE（仅内存表，分布式表用 tableUpsert!）
UPDATE t SET price = price * 1.1 WHERE SecurityID = `AAPL

-- DELETE
DELETE FROM t WHERE TradeTime < 2020.01.01
```

---

## 表连接

```sql
-- 内连接（INNER JOIN）
SELECT a.TradeTime, a.SecurityID, a.price, b.companyName
FROM trades a
INNER JOIN company b ON a.SecurityID = b.SecurityID

-- 左连接（LEFT JOIN）
SELECT a.*, b.factor
FROM t a LEFT JOIN factor b
ON a.SecurityID = b.SecurityID AND a.date = b.date

-- 特有：asof join（时间序列异步连接）
SELECT a.TradeTime, a.price, b.rate
FROM trades a
ASOF JOIN rates b ON a.SecurityID = b.SecurityID, a.TradeTime

-- 特有：window join（时间窗口连接）
SELECT a.TradeTime, a.price, avg(b.volume) AS avgVol
FROM trades a
WINDOW JOIN quotes b
ON a.SecurityID = b.SecurityID, a.TradeTime - 1m : a.TradeTime
```

---

## 子查询与 WITH

```sql
-- 子查询
SELECT * FROM (
    SELECT SecurityID, avg(price) AS avgPrice
    FROM t GROUP BY SecurityID
) WHERE avgPrice > 50

-- WITH（CTE 公共表表达式）
WITH daily AS (
    SELECT date, SecurityID, avg(price) AS avgPrice
    FROM t GROUP BY date, SecurityID
)
SELECT SecurityID, max(avgPrice) FROM daily GROUP BY SecurityID
```

---

## 创建/删除表和数据库

```sql
-- 创建数据库（VALUE + HASH 组合分区）
CREATE DATABASE "dfs://mydb"
PARTITIONED BY VALUE(2020.01.01..2030.12.31), HASH([SYMBOL, 5])
ENGINE='TSDB'  -- 可选：OLAP（默认）、TSDB

-- 创建数据表
CREATE TABLE "dfs://mydb"."trades" (
    TradeTime   TIMESTAMP,
    SecurityID  SYMBOL,
    Price       DOUBLE,
    Volume      LONG
) PARTITIONED BY TradeTime, SecurityID
SORTBY [SecurityID, TradeTime]  -- TSDB 引擎才有此选项

-- DROP TABLE
DROP TABLE "dfs://mydb"."trades"

-- DROP DATABASE
DROP DATABASE "dfs://mydb"
```
