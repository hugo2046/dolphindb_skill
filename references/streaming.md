# DolphinDB 参考：流数据

> 来源：`stream/str_intro.html`, 流数据引擎文档

## 核心概念

DolphinDB 流计算基于**发布-订阅（Pub/Sub）**模型，支持流批一体化计算。

| 组件 | 说明 |
|------|------|
| 流表（Stream Table）| 数据源，生产者向其写入数据 |
| 订阅（Subscribe）| 消费者从流表读取数据流 |
| 流计算引擎 | 对订阅的数据实时计算，支持多种算子 |
| 输出表（Output Table）| 引擎计算结果写入的目标表 |

---

## 创建流表

```dolphindb
// 共享持久化流表（最常用方式）
t = streamTable(1000000:0,
    `TradeTime`SecurityID`Price`Volume,
    [TIMESTAMP, SYMBOL, DOUBLE, LONG])

// 启用持久化流表（支持重启恢复、订阅保留）
enableTableShareAndPersist(t, "trades", preCache=1000000, 
    cacheSize=2000000, asynWrite=true, compress=true, 
    flushMode=0, retentionMinutes=1440)

// 向流表写入数据
objByName("trades").append!(data)
```

---

## 发布订阅

```dolphindb
// 订阅流表（接收消息并处理）
subscribeTable(
    tableName="trades",
    actionName="processAction",
    handler=outputTable,     // 消息处理器（表或函数）
    msgAsTable=true,
    batchSize=2000,           // 批量大小
    throttle=1                // 最大等待时间（秒）
)

// 取消订阅
unsubscribeTable(tableName="trades", actionName="processAction")

// 查看订阅状态
getStreamingStat()

// 查看所有流表
objs(true)
```

---

## 流计算引擎

### 1. 时序聚合引擎（TimeSeriesEngine）

```dolphindb
// 按时间窗口聚合（常规示例）
engine = createTimeSeriesEngine(
    name="myEngine",
    windowSize=60000,         // 60 秒窗口
    step=60000,               // 步长 60 秒
    metrics=<[avg(Price), sum(Volume)]>,
    dummyTable=inputTable,
    outputTable=outputTable,
    timeColumn="TradeTime",
    keyColumn="SecurityID",
    fill=[0, 0]               // NULL 填充值
)

// 向引擎输入数据（通过订阅触发）
subscribeTable(
    tableName="trades",
    actionName="tsEngine",
    handler=engine,
    msgAsTable=true
)
```

#### 实战：完整 1 分钟 OHLCV K 线合成

```dolphindb
// ── 1. 创建 tick 输入流表 ──
share streamTable(1000000:0,
    `TradeTime`SecurityID`Price`Volume,
    [TIMESTAMP, SYMBOL, DOUBLE, LONG]) as tick_stream

// ── 2. 创建 K 线输出表 ──
share table(1000:0,
    `bar_time`SecurityID`open`high`low`close`volume,
    [TIMESTAMP, SYMBOL, DOUBLE, DOUBLE, DOUBLE, DOUBLE, LONG]) as kline_stream

// ── 3. 创建时序聚合引擎（1 分钟窗口）──
klineEngine = createTimeSeriesEngine(
    name      = "kline_1min",
    windowSize= 60000,          // 1 分钟 = 60000 ms
    step      = 60000,
    metrics   = <[first(Price) as open,     // 开盘价
                  max(Price)   as high,     // 最高价
                  min(Price)   as low,      // 最低价
                  last(Price)  as close,    // 收盘价
                  sum(Volume)  as volume]>, // 总成交量
    dummyTable  = tick_stream,
    outputTable = kline_stream,
    timeColumn  = `TradeTime,
    keyColumn   = `SecurityID,
    useSystemTime = false       // 使用数据自带时间戳，非系统时间
)

// ── 4. 订阅 tick 流表，驱动引擎计算 ──
subscribeTable(
    tableName  = "tick_stream",
    actionName = "kline_calc",
    handler    = klineEngine,
    msgAsTable = true,
    batchSize  = 2000,
    throttle   = 0.2           // 每批数据最大等待 0.2 秒
)

// ── 5. 模拟写入测试数据 ──
test_data = table(
    (2024.01.02T09:30:00.000 + (0..9)*3000) as TradeTime,
    take(`000001, 10)          as SecurityID,
    10.0 + rand(1.0, 10)       as Price,
    rand(1000, 10)             as Volume)
tick_stream.append!(test_data)

// ── 6. 查看 K 线结果 ──
select * from kline_stream
```


### 2. 响应式状态引擎（ReactiveStateEngine）

```dolphindb
// 实时计算增量因子（无状态丢失）
metrics = <[
    mavg(Price, 20),          // 20 期移动平均
    mstd(Price, 20),          // 20 期移动标准差
    msum(Volume, 5)           // 5 期总量
]>
engine = createReactiveStateEngine(
    name="rseEngine",
    metrics=metrics,
    dummyTable=inputTable,
    outputTable=outputTable,
    keyColumn="SecurityID"
)
```

### 3. 横截面引擎（CrossSectionalEngine）

```dolphindb
// 按时间截面聚合多只股票
engine = createCrossSectionalEngine(
    name="csEngine",
    metrics=<[
        contextby(rank, Price, SecurityID),   // 横截面排名
        avg(Price)
    ]>,
    dummyTable=inputTable,
    outputTable=outputTable,
    keyColumn="SecurityID",
    triggeringPattern="keyCount",
    triggeringInterval=100
)
```

### 4. 会话窗口引擎（SessionWindowEngine）

```dolphindb
// 按会话（无数据间隔）划分窗口
engine = createSessionWindowEngine(
    name="swEngine",
    sessionGap=10000,         // 10 秒无数据即认为会话结束
    metrics=<[sum(Volume), max(Price)]>,
    dummyTable=inputTable,
    outputTable=outputTable,
    timeColumn="TradeTime",
    keyColumn="SecurityID"
)
```

### 5. 异常检测引擎 / CEP 引擎

```dolphindb
// CEP（复杂事件处理）引擎
// 定义事件模式（pattern）
createCEPEngine(
    name="cepEngine",
    monitors=<[
        PatternMonitor(
            pattern="A -> B within 60s",  // A 后 60 秒内出现 B
            outputTable=alertTable
        )
    ]>,
    dummyTable=inputTable
)
```

---

## 流批一体化

```dolphindb
// 同一套 metrics 既用于批计算也用于流计算
metrics = <[mavg(Price, 5), mstd(Price, 5)]>

// 批计算（历史数据）
result = select TradeTime, SecurityID, mavg(Price, 5) as ma5
from trades
context by SecurityID

// 流计算（实时数据）
engine = createReactiveStateEngine("rs", metrics, ...)
```

---

## 实战：完整流数据链路

```dolphindb
// 1. 创建输出表
outputTable = table(1000:0, 
    `TradeTime`SecurityID`ma5`vol5, 
    [TIMESTAMP, SYMBOL, DOUBLE, LONG])
share outputTable as sharedOutput

// 2. 创建引擎
engine = createReactiveStateEngine(
    name="tickEngine",
    metrics=<[mavg(Price, 5) as ma5, msum(Volume, 5) as vol5]>,
    dummyTable=trades,
    outputTable=sharedOutput,
    keyColumn=`SecurityID
)

// 3. 订阅流表
subscribeTable(
    tableName="trades",
    actionName="calcTick",
    handler=engine,
    msgAsTable=true,
    batchSize=1000,
    throttle=0.5
)

// 4. 模拟写入数据
data = table(now()+(1..10)*1000 AS TradeTime, 
    take(`AAPL, 10) AS SecurityID,
    rand(150.0, 10) AS Price,
    rand(10000, 10) AS Volume)
objByName("trades").append!(data)

---

## 历史数据回放（replay / replayDS）

> 将历史分布式表数据按时序"回放"到流表，常用于量化回测和策略调试。

### replay 基础用法

```dolphindb
// 语法
replay(inputTables, outputTables, dateColumn, timeColumn,
       [replayRate=-1], [absoluteRate=true], [parallelLevel=1])
```

| 参数 | 说明 |
|------|------|
| `inputTables` | 数据源表（内存表或分布式表） |
| `outputTables` | 目标流表（结构必须与输入一致） |
| `dateColumn` | 日期列名 |
| `timeColumn` | 时间列名 |
| `replayRate` | 回放速度：`-1`=极速（无延迟），`1000`=每秒1000条 |
| `absoluteRate` | `true`=按条数/秒限速，`false`=按时间加速倍率 |

```dolphindb
// 创建目标流表
share streamTable(1:0, `TradeDate`TradeTime`SecurityID`Price`Volume,
    [DATE, TIME, SYMBOL, DOUBLE, INT]) as tickStream

// 加载历史数据
histData = loadTable("dfs://stock_data", "trades")

// 回放（极速模式）
replay(
    inputTables=histData,
    outputTables=tickStream,
    dateColumn=`TradeDate,
    timeColumn=`TradeTime,
    replayRate=-1        // -1=极速，不等待
)

// 回放（限速模式，每秒1000条）
replay(histData, tickStream, `TradeDate, `TradeTime, replayRate=1000)
```

---

### replayDS 分段回放（大数据推荐）

`replayDS` 将大查询分割为多个数据段（DataSource），避免一次性加载全量数据到内存。

```dolphindb
// replayDS 语法
ds = replayDS(
    sqlObj=<select * from histData where TradeDate=2024.01.01>,
    dateColumn=`TradeDate,
    timeColumn=`TradeTime,
    [timeRepartitionSchema]   // 可选：时间切片方案
)

// 将 replayDS 结果传给 replay
replay(
    inputTables=ds,
    outputTables=tickStream,
    dateColumn=`TradeDate,
    timeColumn=`TradeTime,
    replayRate=1000
)
```

---

### 与回测引擎联动（完整端到端）

```dolphindb
// ① 清理旧环境（防止重复注册报错）
try { unsubscribeTable(tableName="tickStream", actionName="backtest") } catch(ex) {}
try { dropStreamEngine("backtestEngine") } catch(ex) {}
try { dropStreamTable(`tickStream) } catch(ex) {}

// ② 创建回放流表（结构与历史表一致）
share streamTable(1:0, `TradeDate`TradeTime`SecurityID`Price`Volume,
    [DATE, TIME, SYMBOL, DOUBLE, INT]) as tickStream

// ③ 加载历史数据
histData = loadTable("dfs://stock_data", "trades")

// ④ 创建策略处理引擎（示例：简单统计）
outputTable = table(1000:0, `SecurityID`TradeTime`Price, [SYMBOL, TIME, DOUBLE])
engine = createReactiveStateEngine(
    name="backtestEngine",
    metrics=<[TradeTime, Price]>,
    dummyTable=tickStream,
    outputTable=outputTable,
    keyColumn=`SecurityID
)

// ⑤ 订阅流表 → 触发引擎
subscribeTable(
    tableName="tickStream",
    actionName="backtest",
    handler=engine,
    msgAsTable=true
)

// ⑥ 执行回放（按日期切片）
ds = replayDS(
    sqlObj=<select * from histData where TradeDate between 2024.01.01 and 2024.01.31>,
    dateColumn=`TradeDate,
    timeColumn=`TradeTime
)
replay(ds, tickStream, `TradeDate, `TradeTime, replayRate=-1)

// ⑦ 查看结果
select * from outputTable limit 10
```

---

### 常用运维函数

```dolphindb
// 查看回放/订阅状态
getStreamingStat().pubTables     // 发布表状态
getStreamingStat().subWorkers    // 订阅消费线程状态

// 清理
unsubscribeTable(tableName="tickStream", actionName="backtest")
dropStreamEngine("backtestEngine")
dropStreamTable(`tickStream)
```
```
