# DolphinDB 参考：Python API

> 来源：`api/py/Py_intro.html`，官方 Python API 文档

## 安装

```bash
pip install dolphindb
# 推荐版本与 DolphinDB Server 配套
# 查看兼容版本：pip index versions dolphindb
```

---

## 1. 会话（Session）

```python
import dolphindb as ddb
import numpy as np
import pandas as pd

# 创建会话
s = ddb.session()

# 连接（支持 SSL/加密）
s.connect("localhost", 8848, "admin", "123456")

# 启用压缩（大数据传输时推荐）
s.connect("localhost", 8848, "admin", "123456", compress=True)

# 关闭
s.close()
```

---

## 2. 执行脚本

```python
# 执行任意 DolphinDB 脚本，返回 Python 对象
result = s.run("1 + 1")               # int → 2
result = s.run("1..10")               # numpy.ndarray
result = s.run("now()")               # numpy.datetime64
result = s.run("table(1..3 as id)")   # pandas.DataFrame

# 带参数执行（防止注入，性能更好）
s.run("{0} + {1}", 1, 2)    # 旧版参数方式
# 新版推荐：
s.run("def f(a, b) {return a + b}", pickleTableToList=False)
f = s.run("f")              # 获取函数对象引用
```

---

## 3. 上传数据（Python → DolphinDB）

```python
import pandas as pd
import numpy as np

# 上传变量（一次上传多个）
s.upload({'x': 3.14, 'v': np.array([1, 2, 3])})

# 上传 DataFrame（会自动映射为内存表）
df = pd.DataFrame({
    'TradeTime': pd.date_range('2024-01-01', periods=5, freq='s'),
    'SecurityID': ['AAPL'] * 5,
    'Price': np.random.uniform(100, 200, 5),
    'Volume': np.random.randint(1000, 10000, 5)
})
s.upload({'data': df})

# 然后在服务端使用
s.run("loadTable('dfs://trade_db','trades').append!(data)")
```

---

## 4. 写入分布式表（tableAppender / tableUpsert）

```python
# tableAppender：append 到分区表（最常用）
appender = ddb.tableAppender(
    dbPath="dfs://trade_db",
    tableName="trades",
    ddbSession=s,
    action="fitColumnType"    # 自动类型转换
)
appender.append(df)

# tableUpsert（PKEY/TSDB dedup 场景）
upserter = ddb.tableUpsert(
    dbPath="dfs://trade_db",
    tableName="trades",
    ddbSession=s,
    keyColNames=["SecurityID", "TradeTime"]
)
upserter.upsert(df)

# PartitionedTableAppender（多线程并发写入，大数据量推荐）
appender = ddb.PartitionedTableAppender(
    dbPath="dfs://trade_db",
    tableName="trades",
    partitionColName="TradeTime",
    dbConnectionPool=ddb.DBConnectionPool("localhost", 8848, 4, "admin", "123456")
)
appender.append(df)
```

---

## 5. 查询数据（DolphinDB → Python）

```python
# SQL 查询 → pandas DataFrame
df = s.run("""
    SELECT SecurityID, TradeTime, Price, Volume
    FROM loadTable('dfs://trade_db', 'trades')
    WHERE TradeTime >= 2024.01.01 AND SecurityID = `AAPL
    LIMIT 1000
""")
print(df.dtypes)

# 使用 loadTableBySQL（分区下推，内存友好）
t = s.loadTableBySQL(
    tableName="trades",
    dbPath="dfs://trade_db",
    sql="select * from t where TradeTime between 2024.01.01 and 2024.12.31"
)
df = t.toDF()

# 大表分批读取（分区迭代）
s.run("""
    partitions = getTabletsMeta('<dfs://trade_db>', '<trades>', true).chunkPath
""")
```

---

## 6. 流数据订阅

```python
# 创建新会话用于订阅（避免阻塞主会话）
s_stream = ddb.session()
s_stream.connect("localhost", 8848, "admin", "123456")

# 启用流数据接收端口
s_stream.enableStreaming(port=8849)   # 本地端口，需与服务端协商

# 定义消息处理函数
def handler(msg):
    """msg 是 pandas DataFrame（msgAsTable=True 时）"""
    print(f"收到 {len(msg)} 行数据")
    print(msg.head())

# 订阅流表
s_stream.subscribe(
    host="localhost",
    port=8848,
    handler=handler,
    tableName="trades",      # 服务端流表名
    actionName="mySub",      # 订阅动作名（唯一标识）
    offset=0,                # 起始偏移量（-1=最新）
    msgAsTable=True,         # 以 DataFrame 方式接收
    batchSize=100,           # 批量 N 条触发一次
    throttle=0.5             # 最大等待时间（秒）
)

# 取消订阅
s_stream.unsubscribe("localhost", 8848, "trades", "mySub")
```

---

## 7. 连接池（高并发生产环境）

```python
# 创建连接池
pool = ddb.DBConnectionPool(
    host="localhost",
    port=8848,
    threadNum=8,          # 并发线程数
    userid="admin",
    password="123456",
    compress=True
)

# 使用连接池（基于 asyncio）
import asyncio

async def query():
    result = await pool.run("select count(*) from loadTable('dfs://t','t')")
    return result

asyncio.run(query())
```

---

## 8. 类型映射（Python ↔ DolphinDB）

| Python / NumPy / Pandas | DolphinDB |
|--------------------------|-----------|
| `int`, `np.int32` | INT |
| `np.int64` | LONG |
| `float`, `np.float64` | DOUBLE |
| `np.float32` | FLOAT |
| `str` | STRING |
| `np.bytes_` | SYMBOL（自动） |
| `pd.Timestamp`, `np.datetime64[ns]` | NANOTIMESTAMP |
| `pd.Timestamp[us]` | TIMESTAMP |
| `pd.Series` (datetime.date) | DATE |
| `bool` | BOOL |
| `None` / `np.nan` | NULL |
| `pd.DataFrame` | Table |
| `np.ndarray` 1D | Vector |
| `np.ndarray` 2D | Matrix |
| `dict` | Dictionary |

---

## 9. 常见高级用法

```python
# 执行用户定义函数（从服务端获取函数引用）
s.run("def myFactor(prices, n) { return mavg(prices, n) / prices - 1 }")
# 直接调用
result = s.run("myFactor", pd.Series([100.0, 101, 99, 102, 105]), 3)

# 调用内置函数
result = s.run("mavg", np.array([100.0, 101, 99, 102, 105]), 3)

# 远程调用另一个数据节点
result = s.run("rpc('datanode1', getServerLog, 10)")

# 异步写入（不等返回结果）
s.runAsync("loadTable('dfs://t','t').append!(data)", data=df)
```
