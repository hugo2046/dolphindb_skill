# DolphinDB 参考：回测引擎

> 来源：`backtest/backtest_intro.html`, `plugins/backtest.html`, `plugins/matchingEngineSimulator/mes.html`

## 架构概览

DolphinDB 回测框架由两个插件组成：

| 组件 | 插件名 | 说明 |
|------|--------|------|
| **回测引擎** | `backtest` | 事件驱动回测框架，负责策略逻辑调度 |
| **模拟撮合引擎** | `matchingEngineSimulator` (mes) | 模拟订单撮合，产生成交回报 |

**支持资产类型：** 股票、期货、期权、债券（银行间）、数字货币、多资产组合

---

## 安装插件

```dolphindb
// 安装
installPlugin("backtest")
installPlugin("matchingEngineSimulator")

// 加载
loadPlugin("backtest")
loadPlugin("matchingEngineSimulator")
```

---

## 回测引擎完整流程

### 1. 初始化（策略配置）

```dolphindb
// 回测配置字典
config = dict(STRING, ANY)
config["startDate"] = 2023.01.01
config["endDate"] = 2023.12.31
config["frequency"] = "1m"             // bar 频率：tick/1m/5m/日线等
config["cash"] = 10000000.0            // 初始资金（元）
config["commission"] = 0.0003          // 手续费（万三）
config["slippage"] = 0.0               // 滑点
config["matchingMode"] = "next"        // 以下一 bar 开盘价成交

// 创建回测引擎
engine = Backtest::createBacktestEngine(
    name="myStrategy",
    config=config,
    initialize=initialize,             // 初始化回调
    beforeTrading=beforeTrading,       // 盘前回调
    onBar=onBar,                       // Bar 数据回调
    onTick=onTick,                     // Tick 回调（可选）
    onOrder=onOrder,                   // 订单状态回调（可选）
    onTrade=onTrade,                   // 成交回调（可选）
    afterTrading=afterTrading          // 盘后回调（可选）
)
```

### 2. 策略回调函数

```dolphindb
// ①初始化（仅调用一次）
def initialize(mutable context) {
    // 设置股票池
    context["universe"] = ["600519.SH", "000858.SZ", "600036.SH"]
    // 初始化持仓
    context["positions"] = dict(STRING, DOUBLE)
}

// ②每日盘前（可用于选股、生成信号）
def beforeTrading(mutable context) {
    context["signals"] = generateSignals()
}

// ③核心：每个 Bar 调用一次
def onBar(mutable context, bar) {
    // bar 是一个 dict，包含 symbol/open/high/low/close/volume/tradeTime 等
    sym = bar["symbol"]
    close = bar["close"]
    
    // 买入
    if (context["signals"][sym] == 1) {
        Backtest::order(context, sym, "buy", 100, close)   // 市价买入 100 股
    }
    // 卖出
    if (context["signals"][sym] == -1) {
        pos = Backtest::getPosition(context, sym)
        if (pos > 0) {
            Backtest::order(context, sym, "sell", pos, close)
        }
    }
}

// ④订单状态变化回调
def onOrder(mutable context, order) {
    // order["orderId"], order["status"], order["symbol"], order["qty"] 等
    print("订单状态: " + order["status"])
}

// ⑤成交回调
def onTrade(mutable context, trade) {
    print("成交: " + trade["symbol"] + " " + string(trade["qty"]) + " @ " + string(trade["price"]))
}

// ⑥每日盘后
def afterTrading(mutable context) {
    // 记录当日净值
    nav = Backtest::getPortfolioValue(context)
    print("当日净值: " + string(nav))
}
```

### 3. 下单 API

```dolphindb
// 市价单（以当前 bar close 或下一 bar open 成交，取决于 matchingMode）
Backtest::order(context, symbol, side, qty, price)
// side: "buy" / "sell"

// 限价单
Backtest::limitOrder(context, symbol, side, qty, limitPrice)

// 按目标仓位调仓（自动计算买卖数量）
Backtest::orderTargetQty(context, symbol, targetQty)

// 按目标金额调仓
Backtest::orderTargetValue(context, symbol, targetValue)

// 按目标仓位占比调仓
Backtest::orderTargetPercent(context, symbol, targetPercent)  // 0.1 = 10%

// 撤单
Backtest::cancelOrder(context, orderId)
```

### 4. 查询 API

```dolphindb
// 查询持仓（返回 dict：symbol → qty）
pos = Backtest::getPosition(context)
pos["600519.SH"]   // 某股持仓

// 查询账户净值
nav = Backtest::getPortfolioValue(context)

// 查询可用资金
cash = Backtest::getCash(context)

// 查询当前时间
t = Backtest::getCurrentTime(context)

// 查询历史 K 线
hist = Backtest::getHistoricalData(context, "600519.SH", 20)  // 最近20根bar
```

### 5. 运行回测

```dolphindb
// 加载数据（可以是内存表或分布式表）
marketData = loadTable("dfs://astock", "daily_bar")

// 运行
Backtest::run(engine, marketData)

// 获取回测结果
result = Backtest::getResult(engine)
```

### 6. 分析结果

```dolphindb
// result 是一个字典，包含以下字段：
result["dailyPnL"]      // 每日盈亏（Table）
result["returns"]       // 每日收益率向量
result["trades"]        // 成交详情（Table）
result["orders"]        // 订单详情（Table）
result["positions"]     // 持仓历史（Table）

// 计算绩效指标
annReturn = avg(result["returns"]) * 252
annVol = std(result["returns"]) * sqrt(252)
sharpe = annReturn / annVol

// 最大回撤
nav = result["dailyPnL"]["nav"]
mdd = max((cummax(nav) - nav) / cummax(nav))

print("年化收益: " + string(annReturn))
print("年化波动: " + string(annVol))
print("夏普比率: " + string(sharpe))
print("最大回撤: " + string(mdd))
```

---

## 模拟撮合引擎（单独使用）

```dolphindb
loadPlugin("matchingEngineSimulator")

// 创建撮合引擎
mes = MES::createMatchingEngine(
    name="mesEngine",
    assetType="stock",        // stock/futures/crypto
    matchingRule="priceTime"  // 价格优先时间优先
)

// 提交订单
orderId = MES::submitOrder(mes,
    symbol="600519.SH",
    side="buy",
    qty=100,
    price=1800.0,
    orderType="limit"
)

// 推入行情数据（触发撮合）
MES::processTick(mes, {
    "symbol": "600519.SH",
    "bidPrice": [1799.0, 1798.0],
    "askPrice": [1800.0, 1801.0],
    "bidQty": [500, 1000],
    "askQty": [200, 300],
    "tradeTime": now()
})

// 获取成交回报
fills = MES::getFills(mes)

// 查询订单状态
order = MES::getOrder(mes, orderId)
```
