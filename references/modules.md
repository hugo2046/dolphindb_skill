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

// 移动平均
ta::ma(close, 5)               // 简单移动平均（SMA）
ta::ema(close, 12)             // 指数移动平均（EMA）
ta::dema(close, 12)            // 双指数移动平均
ta::wma(close, 5)              // 加权移动平均

// 动量指标
ta::rsi(close, 14)             // RSI
ta::macd(close, 12, 26, 9)    // MACD（返回 macd/signal/histogram）
ta::cci(high, low, close, 20)  // CCI

// 布林带
result = ta::bBands(close, 20, 2.0)   // 返回 [upper, middle, lower]

// 成交量指标
ta::obv(close, volume)         // OBV
ta::vwap(price, volume)        // VWAP（成交量加权均价）

// K 线形态
ta::doji(open, close, high, low)  // 十字星形态检测
```

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
