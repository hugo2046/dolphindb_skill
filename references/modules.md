# DolphinDB 参考：内置模块

> 来源：`modules/` 目录，官方模块文档

## 模块使用方式

```dolphindb
// 方式一：use 导入（推荐，直接使用函数名）
use mytt
ema(close, 12)          // 直接调用，无需前缀

// 方式二：include 导入（函数需加模块前缀）
include "/path/to/mytt.dos"
mytt::ema(close, 12)

// 方式三：loadModule（系统模块路径）
loadModule("mytt")

// 查看已加载模块
module status
```

---

## 官方内置模块

### ta（技术分析）

```dolphindb
use ta

// ── 移动平均 ──
ta::ma(close, 5)                 // 简单移动平均（SMA）
ta::ema(close, 12)               // 指数移动平均（EMA）
ta::dema(close, 12)              // 双指数移动平均（DEMA）
ta::tema(close, 12)              // 三指数移动平均（TEMA）
ta::wma(close, 5)                // 加权移动平均
ta::kama(close, 10)              // 考夫曼自适应均线

// ── 动量/震荡指标 ──
ta::rsi(close, 14)               // RSI（相对强弱指数）
result = ta::macd(close, 12, 26, 9)
// result[0]=MACD线，result[1]=信号线，result[2]=柱状图

ta::cci(high, low, close, 20)    // 商品通道指数

// KDJ（随机指标）
kdj = ta::kdj(high, low, close, 9, 3, 3)
// kdj[0]=K，kdj[1]=D，kdj[2]=J

// Stochastic（慢速随机）
stoch = ta::stoch(high, low, close, 5, 3, 3)
// stoch[0]=slowK，stoch[1]=slowD

ta::mom(close, 10)               // 动量（Momentum）
ta::roc(close, 10)               // 变化率（ROC）

// ── 波动率 ──
ta::atr(high, low, close, 14)    // 真实波幅（ATR）
bband = ta::bBands(close, 20, 2.0, 2.0)
// bband[0]=upper，bband[1]=middle，bband[2]=lower

// ── 趋势 ──
ta::adx(high, low, close, 14)    // 平均趋向指标
ta::sar(high, low, 0.02, 0.2)    // 抛物线转向

// ── 成交量指标 ──
ta::obv(close, volume)           // 能量潮（OBV）
ta::vwap(price, volume)          // 成交量加权均价（VWAP）
ta::adosc(high, low, close, volume, 3, 10)  // A/D摆动指标

// ── K 线形态识别 ──
ta::doji(open, close, high, low)  // 十字星
ta::hammer(open, close, high, low) // 锤子线
```

> **⚠️ 注意**：`ta` 模块函数返回**向量**（与输入等长），适合在 SQL `context by` 中使用：
> ```dolphindb
> use ta
> select SecurityID, TradeTime, close,
>     ta::ema(close, 12) as ema12,
>     ta::rsi(close, 14) as rsi14
> from loadTable("dfs://trade_db", "kline")
> context by SecurityID
> ```

### mytt（My Technical Tools）

```dolphindb
use mytt

// 更多技术指标（类似文华财经公式）
mytt::ema(close, 12)
mytt::kdj(high, low, close, 9, 3, 3)   // KDJ
mytt::adx(high, low, close, 14)         // ADX
mytt::atr(high, low, close, 14)         // ATR 真实波幅
mytt::sma(close, 5, 1)                  // 文华 SMA（加权 MA）
```

### wq101alpha（WorldQuant 101 Alpha 因子）

```dolphindb
use wq101alpha

// Alpha 因子计算
wq101alpha::alpha001(close, returns)
wq101alpha::alpha002(open, close, volume)
// ... 共 101 个 Alpha 因子
```

### ops（运维工具）

```dolphindb
use ops

// 查看表空间占用
ops::getTableSize("dfs://trade_db", "trades")

// 一键备份
ops::backup(dbPath, tableName, backupDir)

// 查看慢查询
ops::getSlowQueries(n=10)
```

---

## 自定义模块

### 创建模块文件（myModule.dos）

```dolphindb
// myModule.dos

// 定义模块命名空间
module myModule

// 函数定义
def rollingReturn(prices, n=5) {
    return ratios(prices, n) - 1
}

def sharpe(returns, rf=0.0, annFactor=252) {
    excess = returns - rf / annFactor
    return avg(excess) / std(excess) * sqrt(annFactor)
}
```

### 使用自定义模块

```dolphindb
// 加载模块（指定路径）
use "/path/to/myModule.dos"

// 调用函数
prices = loadTable("dfs://trade_db","trades").Price
ret5 = rollingReturn(prices, 5)
sr = sharpe(ret5)
```

---

## 模块开发最佳实践

```dolphindb
// 1. 模块头部声明
module com.mycompany.finance

// 2. 使用命名空间避免冲突
def myEma(data, n) { ... }

// 3. 模块可以 include 其他模块
include "/shared/utils.dos"

// 4. 推荐存放路径：
// server/modules/myModule.dos（会自动搜索）

// 5. 模块路径配置（dolphindb.cfg）
// moduleDir=/path/to/modules
```
