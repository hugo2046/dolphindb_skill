# DolphinDB 参考：量化计算与因子工程



---

> 来源: `best_practice_for_factor_calculation.html`


# 因子计算最佳实践

因子挖掘是量化交易的基础。除传统的基本面因子外，从中高频行情数据中挖掘有价值的因子，并进一步建模和回测以构建交易系统，是一个量化团队的必经之路。金融或者量化金融是一个高度市场化、多方机构高度博弈的领域。因子的有效时间会随着博弈程度的加剧而缩短，如何使用更加高效的工具和流程，更快的找到新的有效的因子，是每一个交易团队必须面对的问题。
近年来，DolphinDB 越来越成为国内乃至国际上大量基金（私募和公募）、资管机构、券商自营团队进行因子挖掘的利器。基于大量客户的反馈，我们特撰写此白皮书， 总结使用 DolphinDB 进行因子挖掘的最佳实践。

## 1. 概述

交易团队用于因子挖掘的常见技术栈有几个大的类别:
- 使用 python、matlab 等数据分析工具
- 委托第三方开发有图形界面的因子挖掘工具
- 使用 java、c++ 等编程语言自行开发挖掘工具
- 在 DolphinDB 等专业工具上进行二次开发
我们暂且不讨论各个技术栈的优缺点。但不管使用何种技术栈，都必须解决以下几个问题:
- 能处理不同频率不同规模的数据集
- 能计算不同风格的因子
- 能处理因子数量不断增长的问题
- 能高效的存取原始数据和因子数据
- 能提升因子挖掘的开发效率
- 能提升因子计算的运行效率（高吞吐，低延时）
- 能解决研究的因子用于生产（实盘交易）的问题
- 能解决多个交易员（研究员）或交易团队一起使用时的各种工程问题，如代码管理、单元测试、权限管理、大规模计算等
DolphinDB 作为分布式计算、实时流计算及分布式存储一体化的高性能时序数据库，非常适合因子的存储、计算、建模、回测和实盘交易。通过部署 DolphinDB 单机或集群环境，用户可以快速地处理 GB 级别甚至 PB 级别的海量数据集，日级、分钟级、快照和逐笔委托数据均能高效响应。
DolphinDB 内置了多范式的编程语言（函数式，命令式，向量式、SQL式），可以帮助研发人员高效开发不同风格的因子。此外，DolphinDB 还提供了丰富且性能高效的函数库（超1400个内置函数），尤其是窗口处理方面经过优化的内置算子，大大缩短了因子计算的延时。
DolphinDB 自带的数据回放和流式增量计算引擎可以方便地解决因子挖掘中研发和生产一体化的问题。DolphinDB 的分布式存储和计算框架，天生便于解决工程中的可靠性、扩展性等问题。
本文基于国内 A 股市场各个频率的数据来演示 DolphinDB 计算和规划因子库存储的方案。根据批量因子计算、实时因子计算、多因子建模、因子库存储规划、因子计算工程化等各个场景的实操演练，以及针对不同方案的对比分析，本文总结出了在 DolphinDB 中进行因子计算的最佳实践。

## 2. 测试数据集

本文的因子计算基于三类国内 A 股行情数据集：逐笔数据、快照数据和 K 线数据（分钟 K 线和日 K 线）。快照数据以两种形式存储：（1）各档数据分别存储为一列；（2）用 array vector 将所有档位的数据存储为一列。
| 数据集 | 简称 | 代码样例中的分区数据库路径 | 代码样例中的表名 | 分区机制 |
| --- | --- | --- | --- | --- |
| 逐笔成交 | level2_tick | dfs://tick_SH_L2_TSDB | tick_SH_L2_TSDB | VALUE:每交易日, HASH: [SYMBOL, 20] |
| 快照 | level2_snapshot | dfs://snapshot_SH_L2_TSDB | snapshot_SH_L2_TSDB | VALUE:每交易日, HASH: [SYMBOL, 20] |
| 快照(向量存储) | level2_snapshot | dfs://LEVEL2_Snapshot_ArrayVector | Snap | VALUE:每交易日, HASH: [SYMBOL, 20] |
| 分钟K线 | k_line | dfs://k_minute_level | k_minute | VALUE:交易月, HASH: [SYMBOL, 3] |
| 日K线 | k_line | dfs://k_day_level | k_day | VALUE:年 |

### 2.1. 逐笔成交数据

逐笔成交是交易所公布买卖双方具体成交的每一笔数据，每3秒发布一次，每次包含这3秒内的所有成交记录。每一笔成交撮合，都由买方和卖方的一笔具体委托组成。上述数据样例采用字段 BuyNo 和 SellNo 标注买卖双方的委托单号，其它关键字段分别为： SecurityID（标的物代码），TradeTime（成交时刻），TradePrice（成交价格），TradeQty（本笔成交量）和 TradeAmount（本笔成交金额）。
每个交易日的原始数据量在 8 GB 上下。根据上表的分区机制进行建库建表，点击查看对应脚本： 逐笔成交数据建库建表完整代码 。

### 2.2. 快照数据

股票交易所每3秒发布一次，每次涵盖这3秒结束时的日内累计成交量(TotalVolumeTrade)，日内累计成交金额(TotalValueTrade)，3秒终了时的盘口买卖双方挂单（买方为 Bid，卖方在有些数据源字段为 Offer，在有些数据源字段为 Ask，其余字段以此类推：BidPrice 为买方各档价格，OfferPrice 为卖方各档价格，OrderQty 为买卖双方各档的委托单总量， Orders 为买卖双方委托单数），3秒终了时的最近一笔成交价格（LastPx），全天开盘价（OpenPx），日内截止当下最高价（HighPx），日内截止当下最低价（LowPx）等各字段。其他和逐笔成交一致的字段不再赘述，涵义一致，详情可参见交易所数据说明字典。
每个交易日的原始数据量约在 10G 左右。
在 DolphinDB 2.0版本的 TSDB 存储引擎中，支持 array vector 的存储机制，即可以允许数据表中一个 cell 存储一个向量。在本白皮书的案例中，后面文章会详细介绍 array vector 存储方案和普通存储方案的区别。快照数据的买10档或卖10档在本例中作为一个 vector 存入单个 cell 中，其他各字段和普通快照数据表都相同。
两种存储模式的建库建表可以参考 Snapshot普通及arrayVector形式建库和建表完整代码
快照数据的 array_vector 存储形式：

### 2.3. 分钟K数据

包含每只股票，每分钟的开盘价、最高价、最低价、收盘价，四个价格字段，同时记录本分钟的成交量和成交金额。另外，数据 K 线可以依据基本字段计算衍生字段，比如：k 线均价(vwap 价格)。k 线数据是由逐笔成交数据聚合产生，具体代码可以参考 基于快照数据的分钟聚合 。
日 K 数据，存储形式和字段跟分钟 k 线一致，可以由分钟 k 线或高频数据聚合产生，这里不作赘述。
日 K 数据，分钟数据的建库建表可以参考： k 线数据建库建表完整代码

## 3. 投研阶段的因子计算

在投研阶段，会通过历史数据批量计算生成因子。通常，推荐研究员将每一种因子的计算都封装成自定义函数。根据因子类型和使用者习惯的不同，DolphinDB 提供了面板和 SQL 两种计算方式。
在面板计算中，自定义函数的参数一般为向量，矩阵或表，输出一般为向量，矩阵或表；在 SQL 模式中，自定义函数的参数一般为向量（列），输出一般为向量。因子函数的粒度尽可能小，只包含计算单个因子的业务逻辑，也不用考虑并行计算加速等问题。这样做的优点包括：（1）易于实现流批一体，（2）便于团队的因子代码提交和管理，（3）方便用统一的框架运行因子计算作业。

### 3.1. 面板数据模式

面板数据(panel data)是以时间为索引，标的为列，指标作为内容的一种数据载体，它非常适用于以标的集合为单位的指标计算，将数据以面板作为载体，可以大大简化脚本的复杂度，通常最后的计算表达式可以从原始的数学公式中一对一的翻译过来。除此之外，可以充分利用DolphinDB矩阵计算的高效能。
在因子计算中，面板数据通常可以通过 panel 函数，或者 exec 搭配 pivot by 得到，具体样例如下表：每一行是一个时间点，每一列是一个股票。

```
000001     000002  000003  000004  ...
                        --------- ---------- ------- ------ ---
2020.01.02T09:29:00.000|3066.336 3212.982  257.523 2400.042 ...
2020.01.02T09:30:00.000|3070.247  3217.087 258.696 2402.221 ...
2020.01.02T09:31:00.000|3070.381 3217.170  259.066 2402.029 ...
```

在面板数据上，由于是以时间为索引，标的为列，因子可以方便地在截面上做各类运算。DolphinDB 包含 row 系列函数以及各类滑动窗口函数，在下面两个因子计算例子中，原本复杂的计算逻辑，在面板数据中，可以用一行代码轻松实现。
- Alpha 1 因子计算中，下例使用了rowRank 函数，可以在面板数据中的每一个时间截面对各标的进行排名；iif 条件运算，可以在标的向量层面直接筛选及计算；mimax 及 mstd 等滑动窗口函数也是在标的层面垂直计算的。因此，在面板计算中合理应用 DolphinDB 的内置函数，可以从不同维度进行计算。

```
//alpha 1 
//Alpha#001公式：rank(Ts_ArgMax(SignedPower((returns<0?stddev(returns,20):close), 2), 5))-0.5

@state
def alpha1TS(close){
	return mimax(pow(iif(ratios(close) - 1 < 0, mstd(ratios(close) - 1, 20),close), 2.0), 5)
}

def alpha1Panel(close){
	return rowRank(X=alpha1TS(close), percent=true) - 0.5
}

input = exec close from loadTable("dfs://k_minute","k_minute") where date(tradetime) between 2020.01.01 : 2020.01.31 pivot by tradetime, securityid
res = alpha1Panel(input)
```

- Alpha 98 因子计算中，同时使用了三个面板数据，分别是vwap, open和vol。不仅各矩阵内部运用了rowRank函数横向截面运算以及m系列垂直滑动窗口计算，矩阵之间也进行了二元运算。用一行代码解决了多维度的复杂的嵌套计算逻辑。

```
//alpha 98
//Alpha #98计算公式:
(rank(decay_linear(correlation(vwap, sum(adv5, 26.4719), 4.58418), 7.18088)) -
rank(decay_linear(Ts_Rank(Ts_ArgMin(correlation(rank(open), rank(adv15), 20.8187), 8.62571), 6.95668), 8.07206)))

def prepareDataForDDBPanel(raw_data, start_time, end_time){
	t = select tradetime,securityid, vwap,vol,open from raw_data where date(tradetime) between start_time : end_time
	return dict(`vwap`open`vol, panel(t.tradetime, t.securityid, [t.vwap, t.open, t.vol]))
}

@state
def alpha98Panel(vwap, open, vol){
	return rowRank(X = mavg(mcorr(vwap, msum(mavg(vol, 5), 26), 5), 1..7),percent=true) - rowRank(X=mavg(mrank(9 - mimin(mcorr(rowRank(X=open,percent=true), rowRank(X=mavg(vol, 15),percent=true), 21), 9), true, 7), 1..8),percent=true)
}

raw_data = loadTable("dfs://k_minute","k_day")
start_time = 2020.01.01
end_time = 2020.12.31
input = prepareDataForDDBPanel(raw_data, start_time, end_time)
timer alpha98DDBPanel = alpha98Panel(input.vwap, input.open, input.vol)
```

基于面板数据的因子计算，耗时主要在面板数据准备和因子计算两个阶段。在很多场景下，面板数据准备的耗时可能超过因子计算本身。为解决这个问题，DolphinDB的TSDB引擎提供了宽表存储，即把面板数据直接存储在数据库表中（面板中每一个列存储为表中的每一个列），这样通过SQL查询可以直接获取面板数据，而不需要通过转置行列来获取，从而大大缩短准备面板数据的时间。在本文的第5章中，我们有详细的宽表和竖表存储性能的对比。

### 3.2. SQL模式

DolphinDB在存储和计算框架上都是基于列式结构，表中的一个列可以直接作为一个向量化函数的输入参数。因此如果一个因子的计算逻辑只涉及股票自身的时间序列数据，不涉及多个股票横截面上的信息，可以直接在SQL中按股票分组，然后在select中调用因子函数计算每个股票在一段时间内的因子值。如果数据在数据库中本身是按股票分区存储的，那么可以非常高效地实现数据库内并行计算。

```
def sum_diff(x, y){
    return (x-y)\(x+y)
}

@state
def factorDoubleEMA(price){
    ema_2 = ema(price, 2)
    ema_4 = ema(price, 4)
    sum_diff_1000 = 1000 * sum_diff(ema_2, ema_4)
    return ema(sum_diff_1000, 2) - ema(sum_diff_1000, 3)
}

res = select tradetime, securityid, `doubleEMA as factorname, factorDoubleEMA(close) as val from loadTable("dfs://k_minute","k_minute") where  tradetime between 2020.01.01 : 2020.01.31 context by securityid
```

在上面的例子中，我们定义了一个因子函数 factorDoubleEMA，只需要用到股票的价格序列信息。我们在 SQL 中通过 context by 子句按股票代码分组，然后调用factorDoubleEMA函数，计算每个股票的因子序列。值得注意的是， context by 是 DolphinDB SQL 对 group by 的扩展，是 DolphinDB 特有的 SQL 语句。 group by 只适用于聚合计算，也就是说输入长度为n，输出长度是1。 context by 适用于向量计算，输入长度是n，输出长度也是n。另外因子函数 factorDOubleEMA 除了可以接受一个向量作为输入，也可以接受一个面板数据作为输入。这也是我们前面强调的，因子函数的粒度尽可能细，这样可以应用于很多场景。
时间序列的因子函数非常普遍，talib 中的所有技术分析指标都属于此类函数，因此都可以使用上述SQL方式或面板数据模式来调用。但是3.1中提到的 alpha1 和 alpha98 等因子，涉及到时间序列和横截面两个维度的计算，我们称之为截面因子，无法将因子逻辑封装在一个自定义函数中，然后在一个 SQL 语句中被调用。通常面对截面因子，我们建议将表作为自定义因子函数的入参，内部用 SQL 进行操作，函数最后返回一个表。

```
//alpha1
def alpha1SQL(t){
	res = select tradetime, securityid, mimax(pow(iif(ratios(close) - 1 < 0, mstd(ratios(close) - 1, 20), close), 2.0), 5) as val from t context by securityid
	return select tradetime, securityid, rank(val, percent=true) - 0.5 as val from res context by tradetime
}
input = select tradetime,securityid, close from loadTable("dfs://k_day_level","k_day") where tradetime between 2010.01.01 : 2010.12.31
alpha1DDBSql = alpha1SQL(input)

//alpha98
def alpha98SQL(mutable t){
	update t set adv5 = mavg(vol, 5), adv15 = mavg(vol, 15) context by securityid
	update t set rank_open = rank(X = open,percent=true), rank_adv15 =rank(X=adv15,percent=true) context by date(tradetime)
	update t set decay7 = mavg(mcorr(vwap, msum(adv5, 26), 5), 1..7), decay8 = mavg(mrank(9 - mimin(mcorr(rank_open, rank_adv15, 21), 9), true, 7), 1..8) context by securityid
	return select tradetime,securityid, `alpha98 as factorname, rank(X =decay7,percent=true)-rank(X =decay8,percent=true) as val from t context by date(tradetime)
}
input = select tradetime,securityid, vwap,vol,open from  loadTable("dfs://k_day_level","k_day") where tradetime between 2010.01.01 : 2010.12.31
alpha98DDBSql = alpha98SQL(input)
```


### 3.3. 不同频率的因子开发举例

不同频率数据的因子，有着不同的特点。本章节将分别举例分钟频、日频、快照、逐笔数据的特点因子，阐述不同频率数据计算因子的最佳实践。

#### 3.3.1. 分钟级和日级数据

日级数据的计算，通常是涉及多个截面的复杂计算，在 面板数据模式 中已展现。对于稍简单的计算，则与分钟级数据的因子相像。
针对分钟级数据，下面的例子是日内收益率偏度的因子计算，对于这类只涉及表内字段的计算，通常使用 SQL 模式，配合 group by 语句将计算分组：

```
defg dayReturnSkew(close){
	return skew(ratios(close))	
}

minReturn = select `dayReturnSkew as factorname, dayReturnSkew(close) as val from loadTable("dfs://k_minute_level", "k_minute") where date(tradetime) between 2020.01.02 : 2020.01.31 group by date(tradetime) as tradetime, securityid

// output
tradetime  securityid factorname    val               
---------- ---------- ------------- -------
2020.01.02 000019     dayReturnSkew 11.8328
2020.01.02 000048     dayReturnSkew 11.0544
2020.01.02 000050     dayReturnSkew 10.6186
```


#### 3.3.2. 基于快照数据的有状态因子计算

有状态的因子，意为因子的计算需要基于之前的计算结果，如一般的滑动窗口计算，聚合计算等，都是有状态的因子计算。
下例flow这个自定义函数中，参数为四个列字段，运用 mavg 滑动平均函数以及 iif 条件运算函数，可以直接在SQL中得到因子结果：

```
@state
def flow(buy_vol, sell_vol, askPrice1, bidPrice1){
        buy_vol_ma = round(mavg(buy_vol, 5*60), 5)
        sell_vol_ma = round(mavg(sell_vol, 5*60), 5)
        buy_prop = iif(abs(buy_vol_ma+sell_vol_ma) < 0, 0.5 , buy_vol_ma/ (buy_vol_ma+sell_vol_ma))
        spd = askPrice1 - bidPrice1
        spd = iif(spd < 0, 0, spd)
        spd_ma = round(mavg(spd, 5*60), 5)
        return iif(spd_ma == 0, 0, buy_prop / spd_ma)
}

res_flow = select TradeTime, SecurityID, `flow as factorname, flow(BidOrderQty[1],OfferOrderQty[1], OfferPrice[1], BidPrice[1]) as val from loadTable("dfs://LEVEL2_Snapshot_ArrayVector","Snap") where date(TradeTime) <= 2020.01.30 and date(TradeTime) >= 2020.01.01 context by SecurityID

// output sample
TradeTime               SecurityID factorname val              
----------------------- ---------- ---------- -----------------
2020.01.22T14:46:27.000 110065     flow       3.7587
2020.01.22T14:46:30.000 110065     flow       3.7515
2020.01.22T14:46:33.000 110065     flow       3.7443
...
```


#### 3.3.3. 快照数据的多档赋权无状态因子计算

计算Level 2的多档快照数据，传统的方式是将多档量价数据存储成为多个列, 再将多档挂单或者报价用 matrix 转换与权重做计算。 更推荐的做法是，将多档数据存储为 array vector，仍旧可以用原来的自定义函数，但是资源消耗包括效率都有提升。 下面的例子是计算多档报价的权重偏度因子，使用 array vector 后计算时间从4秒缩短到2秒。

```
def mathWghtCovar(x, y, w){
	v = (x - rowWavg(x, w)) * (y - rowWavg(y, w))
	return rowWavg(v, w)
}

@state
def mathWghtSkew(x, w){
	x_var = mathWghtCovar(x, x, w)
	x_std = sqrt(x_var)
	x_1 = x - rowWavg(x, w)
	x_2 = x_1*x_1
	len = size(w)
	adj = sqrt((len - 1) * len) \ (len - 2)
	skew = rowWsum(x_2, x_1) \ (x_var * x_std) * adj \ len
	return iif(x_std==0, 0, skew)
}

//weights:
w = 10 9 8 7 6 5 4 3 2 1

//权重偏度因子：
resWeight =  select TradeTime, SecurityID, `mathWghtSkew as factorname, mathWghtSkew(BidPrice, w)  as val from loadTable("dfs://LEVEL2_Snapshot_ArrayVector","Snap")  where date(TradeTime) = 2020.01.02 map
resWeight1 =  select TradeTime, SecurityID, `mathWghtSkew as factorname, mathWghtSkew(matrix(BidPrice0,BidPrice1,BidPrice2,BidPrice3,BidPrice4,BidPrice5,BidPrice6,BidPrice7,BidPrice8,BidPrice9), w)  as val from loadTable("dfs://snapshot_SH_L2_TSDB", "snapshot_SH_L2_TSDB")  where date(TradeTime) = 2020.01.02 map

// output
TradeTime               SecurityID factorname val               
----------------------- ---------- ---------- ------
...
2020.01.02T09:30:09.000 113537     array_1    -0.8828 
2020.01.02T09:30:12.000 113537     array_1    0.7371 
2020.01.02T09:30:15.000 113537     array_1    0.6041 
...
```


#### 3.3.4. 基于快照数据的分钟聚合

投研中经常需要基于快照数据聚合分钟线的 OHLC ，下例就是这一场景中的通用做法：

```
//基于快照因子的分钟聚合OHLC，vwap
tick_aggr = select first(LastPx) as open, max(LastPx) as high, min(LastPx) as low, last(LastPx) as close, sum(totalvolumetrade) as vol,sum(lastpx*totalvolumetrade) as val,wavg(lastpx, totalvolumetrade) as vwap from loadTable("dfs://LEVEL2_Snapshot_ArrayVector","Snap") where date(TradeTime) <= 2020.01.30 and date(TradeTime) >= 2020.01.01 group by SecurityID, bar(TradeTime,1m)
```


#### 3.3.5. 逐笔数据

逐笔数据量较大，一般会针对成交量等字段进行计算，下面的例子计算了每天主买成交量占全部成交量的比例，同样使用 SQL 模式，发挥库内并行计算的优势，并使用 csort 语句用来对组内数据按照时间顺序排序：

```
@state
def buyTradeRatio(buyNo, sellNo, tradeQty){
    return cumsum(iif(buyNo>sellNo, tradeQty, 0))\cumsum(tradeQty)
}

factor = select TradeTime, SecurityID, `buyTradeRatio as factorname, buyTradeRatio(BuyNo, SellNo, TradeQty) as val from loadTable("dfs://tick_SH_L2_TSDB","tick_SH_L2_TSDB") where date(TradeTime)<2020.01.31 and time(TradeTime)>=09:30:00.000 context by SecurityID, date(TradeTime) csort TradeTime

// output
TradeTime           SecurityID factorname val              
------------------- ---------- ---------- ------
2020.01.08T09:30:07 511850     buyTradeRatio    0.0086
2020.01.08T09:30:31 511850     buyTradeRatio    0.0574
2020.01.08T09:30:36 511850     buyTradeRatio    0.0569
...
```


## 4. 生产环境的流式因子计算

在生产环境中，DolphinDB 提供了实时流计算框架。在流计算框架下，用户在投研阶段封装好的基于批量数据开发的因子函数，可以无缝投入交易和投资方面的生产程序中，这就是通常所说的批流一体。使用流批一体可以加速用户的开发和部署。同时流计算框架还在算法的路径上，做了极致的优化，在具有高效开发的优势的同时，又兼顾了计算的高效性能。在这一章中，将会基于实际的状态因子案例，展示实时流计算的使用方法。
DolphinDB 流计算解决方案的核心部件是流计算引擎和流数据表。流计算引擎用于时间序列处理、横截面处理、窗口处理、表关联、异常检测等操作。流数据表可以看作是一个简化版的消息中间件，或者说是消息中间件中的一个主题（topic），可以往其发布（publish）数据，也可以从其订阅（subscribe）数据。流计算引擎和流数据表均继承于 DolphinDB 的数据表（table），因此都可以通过 append! 函数往其注入数据。流计算引擎的输出也是数据表的形式，因此多个计算引擎可以跟搭积木一样自由组合，形成流式处理的流水线。

### 4.1. 流式增量计算

金融方面的原始数据和计算指标，在时间上通常有延续性的关系。以最简单的五周期移动均线 mavg(close,5) 为例，当新一个周期的数据传入模型时，可以将之前最远的第五周期值从 sum 中减出，再把最新一个周期的值加入 sum ，这样就不必每个周期只更新一个值时都重算一遍 sum 。这种增量计算是流计算的核心，可以大大降低实时计算的延时。DolphinDB内置了大量量化金融中需要用到的基本算子，并为这些算子实现了高效的增量算法。不仅如此，DolphinDB还支持自定义函数的增量实现。在前一章节中，部分自定义的因子函数加了修饰符 @state ，表示该函数支持增量计算。

#### 4.1.1. 主买成交量占比因子的流式处理

逐笔数据 的逐笔数据因子的例子展示了主买成交量占比因子（buyTradeRatio）的批量实现方式。以下代码演示如何使用响应式状态引擎（reactive state engine）来实现该因子的流式增量计算。
上述代码创建了一个名为demo的响应式状态引擎，SecurityID作为分组键，输入的消息格式同内存表tickStream。需要计算的指标定义在factors中，其中1个是输入表中的原始字段TradeTime，另一个是我们需要计算的因子的函数表示。输出到内存表result，除了在factors中定义的指标外，输出结果还会添加分组键。请注意，自定义的因子函数跟批计算中的完全一致！创建完引擎之后，我们即可往引擎中插入几条数据，并观察计算结果。

```
insert into demoEngine values(`000155, 2020.01.01T09:30:00, 30.85, 100, 3085, 4951, 0)
insert into demoEngine values(`000155, 2020.01.01T09:30:01, 30.86, 100, 3086, 4951, 1)
insert into demoEngine values(`000155, 2020.01.01T09:30:02, 30.80, 200, 6160, 5501, 5600)

select * from result

SecurityID TradeTime           Factor
---------- ------------------- ------
000155     2020.01.01T09:30:00 1
000155     2020.01.01T09:30:01 1
000155     2020.01.01T09:30:02 0.5
```

从这个例子可以看出，在DolphinDB中实现因子的流式增量计算非常简单。如果在投研阶段，已经用我们推荐的方式自定义了一个因子函数，在生产阶段只要程序性的创建一个流式计算引擎即可实现目标。这也是为什么我们一再强调，自定义的因子函数必须使用规范的接口，而且只包含核心的因子逻辑，不用考虑并行计算等问题。

#### 4.1.2. 大小单的流式处理

资金流分析是逐笔委托数据的一个重要应用场景。在实时处理逐笔数据时，大小单的统计是资金流分析的一个具体应用。大小单在一定程度上能反映主力、散户的动向。但在实时场景中，大小单的生成有很多难点：（1） 大小单的计算涉及历史状态，如若不能实现增量计算，当计算下午的数据时，可能需要回溯有关这笔订单上午的数据，效率会非常低下。 （2）计算涉及至少两个阶段。在第一阶段需要根据订单分组，根据订单的累计成交量判断大小单，在第二阶段要根据股票来分组，统计每个股票的大小单数量及金额。
大小单是一个动态的概念。一个小单在成交量增加后可能变成一个大单。DolphinDB的两个内置函数 dynamicGroupCumsum 和 dynamicGroupCumcount 用于对动态组的增量计算。完整的代码请参考： 大小单的流式处理 。

```
@state
def factorSmallOrderNetAmountRatio(tradeAmount, sellCumAmount, sellOrderFlag, prevSellCumAmount, prevSellOrderFlag, buyCumAmount, buyOrderFlag, prevBuyCumAmount, prevBuyOrderFlag){
	cumsumTradeAmount = cumsum(tradeAmount)
	smallSellCumAmount, bigSellCumAmount = dynamicGroupCumsum(sellCumAmount, prevSellCumAmount, sellOrderFlag, prevSellOrderFlag, 2)
	smallBuyCumAmount, bigBuyCumAmount = dynamicGroupCumsum(buyCumAmount, prevBuyCumAmount, buyOrderFlag, prevBuyOrderFlag, 2) 
	f = (smallBuyCumAmount - smallSellCumAmount) \ cumsumTradeAmount
	return smallBuyCumAmount, smallSellCumAmount, cumsumTradeAmount, f
}

def createStreamEngine(result){
	tradeSchema = createTradeSchema()
	result1Schema = createResult1Schema()
	result2Schema = createResult2Schema()
	engineNames = ["rse1", "rse2", "res3"]
	cleanStreamEngines(engineNames)
	
	metrics3 = <[TradeTime, factorSmallOrderNetAmountRatio(tradeAmount, sellCumAmount, sellOrderFlag, prevSellCumAmount, prevSellOrderFlag, buyCumAmount, buyOrderFlag, prevBuyCumAmount, prevBuyOrderFlag)]>
	rse3 = createReactiveStateEngine(name=engineNames[2], metrics=metrics3, dummyTable=result2Schema, outputTable=result, keyColumn="SecurityID")
	
	metrics2 = <[BuyNo, SecurityID, TradeTime, TradeAmount, BuyCumAmount, PrevBuyCumAmount, BuyOrderFlag, PrevBuyOrderFlag, factorOrderCumAmount(TradeAmount)]>
	rse2 = createReactiveStateEngine(name=engineNames[1], metrics=metrics2, dummyTable=result1Schema, outputTable=rse3, keyColumn="SellNo")
	
	metrics1 = <[SecurityID, SellNo, TradeTime, TradeAmount, factorOrderCumAmount(TradeAmount)]>
	return createReactiveStateEngine(name=engineNames[0], metrics=metrics1, dummyTable=tradeSchema, outputTable=rse2, keyColumn="BuyNo")
}
```

自定义函数 factorSmallOrderNetAmountRatio 是一个状态因子函数，用于计算小单的净流入资金占总的交易资金的比例。 createStreamEngine 创建流式计算引擎。我们一共创建了3个级联的响应式状态引擎，后一个作为前一个的输出，因此从最后一个引擎开始创建。前两个计算引擎rse1和rse2分别以买方订单号（BuyNo)和卖方订单号（SellNo）作为分组键，计算每个订单的累计交易量，并以此区分是大单或小单。第三个引擎rse3把股票代码（SecurityID）作为分组键，统计每个股票的小单净流入资金占总交易资金的比例。下面我们输入一些样本数据来观察流计算引擎的运行。

```
result = createResultTable()
rse = createStreamEngine(result)
insert into rse values(`000155, 1000, 1001, 2020.01.01T09:30:00, 20000)
insert into rse values(`000155, 1000, 1002, 2020.01.01T09:30:01, 40000)
insert into rse values(`000155, 1000, 1003, 2020.01.01T09:30:02, 60000)
insert into rse values(`000155, 1004, 1003, 2020.01.01T09:30:03, 30000)

select * from result

SecurityID TradeTime smallBuyOrderAmount smallSellOrderAmount totalOrderAmount factor
---------- ------------------- ------------------- -------------------- ---------------- ------
000155     2020.01.01T09:30:00 20000               20000                20000            0
000155     2020.01.01T09:30:01 60000               60000                60000            0
000155     2020.01.01T09:30:02 0                   120000               120000           -1
000155     2020.01.01T09:30:03 30000               150000               150000           -0.8
```


#### 4.1.3. 复杂因子Alpha #1流式计算的快捷实现

从前一个大小单的例子可以看到，有些因子的流式实现比较复杂，需要创建多个引擎进行流水线处理来完成。完全用手工的方式来创建多个引擎其实是一件耗时的工作。如果输入的指标计算只涉及一个分组键，DolphinDB提供了一个解析引擎 streamEngineParser 来解决此问题。下面我们以面板数据模式一节的alpha #1因子为例，展示 streamEngineParser 的使用方法。完整代码参考 Alpha #1流式计算 。以下为核心代码。

```
@state
def alpha1TS(close){
	return mimax(pow(iif(ratios(close) - 1 < 0, mstd(ratios(close) - 1, 20),close), 2.0), 5)
}

def alpha1Panel(close){
	return rowRank(X=alpha1TS(close), percent=true) - 0.5
}

inputSchema = table(1:0, ["SecurityID","TradeTime","close"], [SYMBOL,TIMESTAMP,DOUBLE])
result = table(10000:0, ["TradeTime","SecurityID", "factor"], [TIMESTAMP,SYMBOL,DOUBLE])
metrics = <[SecurityID, alpha1Panel(close)]>
streamEngine = streamEngineParser(name="alpha1Parser", metrics=metrics, dummyTable=inputSchema, outputTable=result, keyColumn="SecurityID", timeColumn=`tradetime, triggeringPattern='keyCount', triggeringInterval=4000)
```

因子alpha1实际上包含了时间序列处理和横截面处理，需要响应式状态引擎和横截面引擎串联来处理才能完成。但以上代码仅仅使用了streamEngineParser就创建了全部引擎，大大简化了创建过程。
前面三个例子展示了DolphinDB如何通过流计算引擎实现因子在生产环境中的增量计算。值得注意的是，流式计算时直接使用了投研阶段生成的核心因子代码，这很好的解决了传统金融分析面临的批流一体问题。在传统的研究框架下，用户往往需要对同一个因子计算逻辑写两套代码，一套用于在历史数据上建模、回测，另外一套专门处理盘中传入的实时数据。这是因为数据传入程序的形状(机制)不统一，又甚至是编程语言也无法统一。比如研究分析使用了 python 或者 R，在 python 或 R 的研究程序确定模型和参数后，生产交易的程序必须用 C++ 再实现这套模型，才能保证交易时的执行效率。在两套代码完成后，还要再校验它们计算出来的结果是否一致。这样的业务流程毫无疑问加重了研究员和程序员们的负担，也让基金经理们没法更快地让新交易思路迭代上线。在DolphinDB的流式计算中，实时行情订阅、行情数据收录、交易实时计算、盘后研究建模，全都用同一套代码完成，保证在历史回放和生产交易当中数据完全一致。
除了三个例子中用到的响应式状态引擎(reactive state engine)和横截面引擎（cross sectional engine），DolphinDB 还提供了多种流数据处理引擎包括做流表连接的 asof join engine，equal join engine，lookup join engine，window join engine ，时间序列聚合引擎（time series engine），异常检测引擎(anomaly detection engine)，会话窗口引擎（session window engine）等。

### 4.2. 数据回放

前一节我们介绍了因子计算的批流一体实现方案，简单地说，就是一套代码（自定义的因子函数），两种引擎（批计算引擎和流计算引擎）。事实上，DolphinDB提供一种更为简洁的批流一体实现方案，那就是在历史数据建模时，通过数据回放，也用流引擎来实现计算。
在 SQL语句方式批处理计算factorDoubleEMA因子 的例子中，这里介绍如何使用流计算的方式回放数据，计算 factorDoubleEMA 的因子值。全部代码参考 流计算factorDoubleEMA因子

```
//创建流引擎，并传入因子算法factorDoubleEMA
factors = <[TradeTime, factorDoubleEMA(close)]>
demoEngine = createReactiveStateEngine(name=engineName, metrics=factors, dummyTable=inputDummyTable, outputTable=resultTable, keyColumn="SecurityID")
	
//demo_engine订阅snapshotStreamTable流数据表
subscribeTable(tableName=snapshotSharedTableName, actionName=actionName, handler=append!{demoEngine}, msgAsTable=true)

//创建播放数据源供replay函数历史回放；盘中的时候，改为行情数据直接写入snapshotStreamTable流数据表
inputDS = replayDS(<select SecurityID, TradeTime, LastPx from tableHandle where date(TradeTime)<2020.07.01>, `TradeTime, `TradeTime)
```


### 4.3. 对接交易系统

DolphinDB 本身具有多种常用编程语言的API，包括C++, java, javascript, c#, python, go等。使用这些语言的程序，都可以调用该语言的 DolphinDB 接口，订阅到 DolphinDB 服务器的流数据。本例提供一个简单的 python接口订阅流数据 样例。
DolphinDB-Python API订阅流数据例子：

```
current_ddb_session.subscribe(host=DDB_datanode_host,tableName=stream_table_shared_name,actionName=action_name,offset=0,resub=False,filter=None,port=DDB_server_port,
handler=python_callback_handler,#此处传入python端要接收消息的回调函数
)
```

在金融生产环境中，更常见的情况，是流数据实时的灌注到消息队列中，供下游的其他模块消费。DolphinDB 也支持将实时计算结果推送到消息中间件，与交易程序对接。示例中提供的样例，使用 DolphinDB 的开源 ZMQ 插件，将实时计算的结果推送到 ZMQ 消息队列，供下游ZMQ协议的订阅程序消费(交易或展示)。除ZMQ之外，其他支持的工具都在 DolphinDB 插件库 中提供。所有已有的 DolphinDB 插件都是开源的，插件的编写组件也是开源的，用户也可按自己的需要编写。
DolphinDB向ZMQ消息队列推送流数据代码样例：
- 启动下游的ZMQ数据消费程序，作为监听端(ZeroMQ消息队列的服务端)，完整代码见 向ZMQ推送流数据 zmq_context = Context() zmq_bingding_socket = zmq_context.socket(SUB)#见完整版代码设定socket选项 zmq_bingding_socket.bind("tcp://*:55556") async def loop_runner(): while True: msg=await zmq_bingding_socket.recv()#阻塞循环until收到流数据 print(msg)#在此编写下游消息处理代码 asyncio.run(loop_runner())

## 5. 因子的存储和查询

无论是批量计算还是实时计算，将 DolphinDB 中计算生成的因子保存下来提供给投研做后续的分析都是很有意义的。本章主要是根据存储、查询，使用方式等方面，来分析如何基于使用场景来选择更高效的存储模型。本章测试均为在 Linux 系统上部署的单节点集群，其中包含三个数据节点，每个节点设置16线程并采用三块固态硬盘进行数据存贮。
在实际考虑数据存储方案，我们需要从以下三个方面考虑：
- 选择 OLAP 引擎还是 TSDB 引擎。OLAP 最适合全量跑批计算，TSDB 则在序列查询上优势突出，性能和功能上比较全面。
- 因子的存储方式是单值纵表方式还是多值宽表方式。 单值方式的最大优点是灵活性强，增加因子和股票时，不用修改表结构；缺点是数据冗余度高。多值宽表的数据冗余度很低，配合 TSDB 引擎的 array vector ，使用宽表结构节省了行数，提高了存储效率，但若出现新因子或新股票，需要重新生成因子表。
- 分区方式选择。可用于分区的列包括时间列，股票代码列和因子列。OLAP 引擎推荐的分区大小为原始数据100MB左右。为保证最佳性能，TSDB 引擎推荐单分区数据量大小保持在 100MB-1GB 范围内性能最佳。
结合以上考虑因素，我们以4000只股票，1000个因子，存储分钟级因子库为例，有如下三种选择：
- 以纵表存储,使用 OLAP 引擎，每行按时间存储一只股票一个因子数据，分区方案 VALUE(月)+ VALUE(因子名)。
- 以纵表存储,使用 TSDB 引擎，每行按时间存储一只股票一个因子数据，分区方案 VALUE(月)+ VALUE(因子名)， 按股票代码+时间排序。
- 以宽表存储,使用 TSDB 引擎，每行按时间存储全部股票一个因子，或者一支股票全部因子数据，分区方案 VALUE(月)+ VALUE(因子名)，按因子名+时间排序。
OLAP 引擎是纯列式存储，不适合表过宽，若存储宽表的列数超过 80，写入性能会逐渐下降。本例中每行存储全部股票的一个因子，因此按股票代码作为宽表的列，列数过多，所以不使用宽表的方式存储。
| tradetime | securityid | factorname | factorval |
| --- | --- | --- | --- |
| 2020:12:31 09:30:00 | sz000001 | factor1 | 143.20 |
| 2020:12:31 09:30:00 | sz000001 | factor2 | 142.20 |
| 2020:12:31 09:30:00 | sz000002 | factor1 | 142.20 |
| 2020:12:31 09:30:00 | sz000002 | factor2 | 142.20 |
| tradetime | factorname | sz000001 | sz000002 | sz000003 | sz000004 | sz000005 | ...... | sz600203 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2020:12:31 09:30:00 | factor1 | 143.20 | 143.20 | 143.20 | 143.20 | 143.20 | ....... | 143.20 |
| 2020:12:31 09:30:00 | factor2 | 143.20 | 143.20 | 143.20 | 143.20 | 143.20 | ....... | 143.20 |

### 5.1. 因子存储

我们以存储5个因子一年的分钟级数据来进行测试，比对这三种存储模式在数据大小、实际使用的存储空间、写入速度等方面的优劣。
| 横纵方式 | 数据引擎 | 数据总行数 | 单行字节 | 数据大小(GB) | 数据落盘大小(GB) | 压缩比 | 写入磁盘耗时 | IO峰值(m/s) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 纵表 | OLAP | 1,268,080,000 | 24 | 28.34 | 9.62 | 0.34 | 150 | 430 |
| 纵表 | TSDB | 1,268,080,000 | 24 | 28.34 | 9.03 | 0.32 | 226 | 430 |
| 宽表 | TSDB | 317,020 | 32,012 | 9.45 | 8.50 | 0.90 | 38 | 430 |
从比对结果来看，宽表 TSDB 模式的写入速度是纵表 OLAP 的4倍，纵表 TSDB 的5倍，存储空间上 OLAP 纵表和 TSDB 纵表相近，TSDB 宽表略小于前二者，压缩比上纵表 TSDB 最优，纵表 OLAP 次之，宽表 TSDB 最差。原因如下：
- 实际产生的数据字节上，纵表模式是宽表模式的三倍，决定了宽表 TSDB 的的写入速度最优，磁盘使用空间最优，同时宽表 TSDB 模式的压缩比也会相对差一些。
- 模拟数据随机性很多大，也影响了 TSDB 引擎宽表的数据压缩。
- TSDB 引擎会进行数据排序，生成索引，所以同样是纵表，TSDB 引擎在存储空间、存储速度、压缩比方面都要略逊于 OLAP 引擎。
具体存储脚本参考 因子数据存储模拟脚本 。

### 5.2. 因子查询

下面我们模拟大数据量来进行查询测试，模拟4000支股票，200个因子，一年的分钟级数据，详细数据信息及分区信息见下面表格：
| 横纵方式 | 数据引擎 | 股票数 | 因子数 | 时间跨度 | 数据级别 | 数据总行数 | 每行字节 | 数据大小(GB) | 数据分区 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 纵表 | OLAP | 4000 | 200 | 一年 | 分钟级 | 50,723,200,000 | 24 | 1133.75 | 日(VALUE分区)+因子(VALUE分区) |
| 纵表 | TSDB | 4000 | 200 | 一年 | 分钟级 | 50,723,200,000 | 24 | 1133.75 | 月(VALUE分区)+因子(VALUE分区) |
| 宽表 | TSDB | 4000 | 200 | 一年 | 分钟级 | 12,680,800 | 32012 | 340.19 | 月(VALUE分区)+因子(VALUE分区) |
下面我们通过多个角度的查询测试来比对这三种存储方式的查询性能。 因子查询测试脚本
- 查询1个因子1只股票指定时间点数据 横纵方式 数据引擎 查询行数 字节数 耗时(ms) 纵表 OLAP 1 24 1100 纵表 TSDB 1 24 6 宽表 TSDB 1 20 2 在点查询上 TSDB 引擎优势明显，而宽表 TSDB 因为数据行数少，速度上还要快于纵表 TSDB 模式。
- 查询1个因子1只股票一年分钟级数据 横纵方式 数据引擎 数据大小(MB) 耗时(s) 纵表 OLAP 1.5 0.9 纵表 TSDB 1.5 0.03 宽表 TSDB 1.2 0.02 查询单因子单股票一年的分钟级数据宽表 TSDB 引擎速度最快，这是因为 TSDB 引擎分区较大，读取的文件少，且数据有排序，而 OLAP 引擎本身数据分区较小，需要扫描的行数又同样不少，所以速度最慢。
- 查询1个因子全市场股票一年分钟级数据 横纵方式 数据引擎 数据大小(GB) 耗时(s) 纵表 OLAP 5.7 8.9 纵表 TSDB 5.7 12.4 宽表 TSDB 1.9 3.8 宽表 TSDB 读取速度最快，读取的总数据量比较大时，这几种模式都会读取很多完整分区，而宽表 TSDB 模式因为实际数据比较小，所以速度上是纵表 OLAP 的一半，是纵表 TSDB 的三分之一左右。
- 查询3个因子全市场股票一年分钟级数据 横纵方式 数据引擎 数据大小(GB) 耗时(s) 纵表 OLAP 17.0 17.7 纵表 TSDB 17.0 25.9 宽表 TSDB 5.7 10.7 更大数据量的数据读取，查询耗时线性增长，同样原因，宽表 TSDB 读取速度仍然最快。
- 查询1只股票全部因子一年的分钟级数据 宽表在进行该查询时，查询 SQL 应只选择需要股票代码列，SQL 如下： //纵表查询sql, 查询全部字段，使用通配符* tsdb_symbol_all=select * from tsdb_min_factor where symbol=`sz000056 //宽表查询sql,只检索部分字段，详细列出 select mtime,factorname,sz000001 from tsdb_wide_min_factor 横纵方式 数据引擎 数据大小(MB) 耗时 纵表 OLAP 312 7min58s 纵表 TSDB 312 1.5s 宽表 TSDB 260 0.5s
以上结果可以看到，宽表 TSDB 引擎和纵表 TSDB 都可以很快的查出数据，而纵表模式 OLAP 则需要百倍以上的时间才能查询出数据。这是因为纵表模式 OLAP 的分区字段是时间和因子，这种情况下查询某只股票所有的因子需要扫描全部分区的全部列才能取出所需的数据；而宽表 TSDB 引擎只需要取三列数据，所以可以很快查出数据。对比纵表 TSDB 和 OLAP 引擎的耗时，可以发现纵表 TSDB 查询速度也比较快，这是因为 TSDB 引擎按股票代码维护了索引，以实现快速检索。
综上所述，因子的存储需根据用户的查询习惯去做规划。根据上述性能对比，本节涉及的查询推荐使用宽表 TSDB 的方式存储因子。

### 5.3. 在线获取面板数据

对于不同的存储模式，可通过以下 DolphinDB 脚本在线生成面板数据。
- 生成3个因子全市场股票一年分钟级面板数据 //纵表模式取面板数据sql olap_factor_year_pivot=select val from olap_min_factor where factorcode in ('f0001','f0002','f0003') pivot by tradetime,symbol ,factorcode //宽表模式取面板数据sql wide_tsdb_factor_year=select * from tsdb_wide_min_factor where factorname in ('f0001','f0002','f0003') 横纵方式 数据引擎 数据大小(GB) 耗时(s) 纵表 OLAP 5.7 10分钟以上 中止 纵表 TSDB 5.7 10分钟以上 中止 宽表 TSDB 5.7 9.5
宽表 TSDB 引擎具有最佳的查询性能，随着数据量上升，纵表数据列转行操作要额外增加 piovt by 的列，从而增加更多的去重、排序操作，导致生成面板数据的耗时进一步增加。
使用宽表 TSDB 模式存储在以下方面均有明显优势：
- 存储空间：虽然宽表 TSDB 在压缩比上相对逊色，但是由于宽表模式本书数据字节只有纵表模式的三分之一，所以在磁盘空间上宽表 TSDB 模式使用最小；
- 存储速度：宽表 TSDB 模式的在写入相同有效数据的情况下写入速度是纵表 OLAP 的4倍，纵表 TSDB 的5倍；
- 直接检索数据： 宽表 TSDB 模式在不同场景的查询速度至少是纵表 OLAP 和纵表 TSDB 的1.5倍，甚至可能达到100倍以上；
- 以面板模式检索数据：宽表 TSDB 模式的查询速度是纵表 OLAP 和纵表 TSDB 的至少10倍以上；
- 在以非分区维度检索数据：例如，按因子分区的按股票检索数据，此场景宽表 TSDB 模式查询速度是纵表 OLAP 和纵表 TSDB 的300倍和500倍。
综上，如果一定时期内股票和因子数量固定，因子存储的最佳选择方式为 TSDB 宽表的模式进行存储，用户可以根据实际场景的查询需求，来选择生成以股票名或因子名做为列的宽表。

## 6. 因子回测和建模

很多时候，计算因子只是投研阶段的第一部分，而最重要的部分其实在于如何挑选最为有效的因子。在本章节中，将会讲述如何在DolphinDB中做因子间的相关性分析，以及回归分析。

### 6.1. 因子回测

因子的建模和计算等，一旦从图表上分析出有方向性的结论，就要做成策略。按照确定的因子信号来设计出来的一套买卖条件，就是所谓的投资策略。把一套投资策略代入到历史数据当中，计算按照这样的策略条件去做交易是否长期有利可图的过程就是回测。
事件驱动型回测主要用来分析少量标的，中高频的交易策略。在按因子配置投资组合的策略类型中不是核心或重点，这里不做详细阐述。
本章主要介绍的案例是向量化的因子回测。
首先，在k线数据上，实现了一个按多日股票收益率连乘打分的因子。之后根据分值排序高低分配标的持仓权重。
得到分配持仓权重后，再与持仓股票的日收益率做矩阵乘法，最后按天相加，可得整个投资组合的回报率变化曲线。
完整实例代码参考： 向量化因子回测完整代码

### 6.2. 因子相关性分析

在之前的章节中，存储因子的库表可以是多值模型，也可以是单值模型。在求因子间相关性时，推荐利用 array vector 将同一股票同一时间的多个因子放在一个列中，这样可以避免枚举多个列名。下面以单值模型为例，演示如何有效地先在股票内求因子间相关性，然后根据股票个数求均值。
- 单值模型计算因子间自相关性矩阵 其原理是先将当天的因子根据时间和标的，转换成 array vector ，再对生成的小内存表进行计算求值。 day_data = select toArray(val) as factor_value from loadTable("dfs://MIN_FACTOR_OLAP_VERTICAL","min_factor") where date(tradetime) = 2020.01.03 group by tradetime, securityid result = select toArray(matrix(factor_value).corrMatrix()) as corr from day_data group by securityid corrMatrix = result.corr.matrix().avg().reshape(size(distinct(day_data.factorname)):size(distinct(day_data.factorname)))

### 6.3. 多因子建模

在大部分场景中，多因子投资模型的搭建可分为：1，简单加权法；2，回归法；两种方式均可以在 DolphinDB 中实现。
- 简单加权法 对不同的因子不同的权重，计算出所有因子预测的各只股票的预期回报率的加权平均值，然后选择预期回报率最高的股票。这类方法比较简单，故不在本小节赘述。
- 回归法 在DolphinDB中，有很多相关的内置函数。细节使用请参考文档： DolphinDB教程：机器学习 其中对于线性回归内置了多种模型，包括普通最小二乘法回归（OLS Regression），脊回归（Ridge Regression），广义线性模型（Generalized Linear Model）等。 目前，普通最小二乘法回归 olsEx ，脊回归 ridge 中的 'cholesky' 算法，广义线性模型 glm 都支持分布式并行计算。 其他回归模型，DolphinDB 支持 Lasso 回归，ElasticNet 回归，随机森林回归，AdaBoost 回归等。其中，AdaBoost 回归 adaBoostRegressor ， randomForestRegressor 支持分布式并行计算。

## 7. 因子计算的工程化

在实际量化投研过程，研究员要聚焦策略因子研发，而因子计算框架的开发维护通常是IT部门人员来负责，为了加强协作，通常要进行工程化管理。好的工程化管理能减少重复、冗余工作，极大的提高生产效率，使策略投研更加高效。本章节将会通过一些案例来介绍如何对因子计算进行工程化管理。

### 7.1. 代码管理

因子的开发往往涉及到QUANT团队和IT团队。QUANT团队主要负责因子开发和维护因子逻辑代码。IT团队负责因子计算框架的开发和运维。因此要把计算框架的代码和因子本身的逻辑代码做到有效的分离，降低耦合，并且可以支持因子开发团队单独提交因子逻辑代码，计算框架能够自动更新并进行因子重算等任务。本节我们主要讨论因子逻辑代码管理，计算框架和运维请参考7.3和7.6。
我们推荐用户使用自定义函数来封装核心的因子逻辑，每个因子对应一个自定义函数。DolphinDB对自定义函数的管理提供了两种方法，函数视图（Function View）和模块（Module）。函数视图的优点包括：（1）集中管理，添加到集群后，所有节点都可以使用；（2）支持权限管理。函数视图的主要缺点是无法进行模块化管理，当数量增加时，运维难度增加。模块的优缺点正好同函数视图相反。模块可以将大量函数按目录树结构组织在不同模块中。既可以在系统初始化时预加载，也可以在需要使用的时候使用use语句，引入这个模块。但是模块必须复制到每个需要使用的节点才可以使用，另外无法对模块中的函数进行权限管理。后续版本会统一函数视图和模块的优点。

### 7.2. 单元测试

遇到因子代码重构、计算框架调整、数据库升级等情况，必须对最基本的因子逻辑进行正确性测试。DolphinDB内置了单元测试框架，可用于自动化测试。
这个单元测试框架主要包含了以下内容：
- test 函数，可以测试一个单元测试文件或一个目录下的所有单元测试文件。
- @testing宏，用于描述一个测试case。
- assert语句，判断结果是否符合预期。
- eqObj等函数，用于测试结果是否符合预期。
下面通过对因子函数factorDoubleEMA的测试来展示单元测试的撰写。全部代码请点击 脚本 。下面的代码展示了三个测试cases，两个用于批处理，一个用于流计算处理。

```
@testing: case = "factorDoubleEMA_without_null"
re = factorDoubleEMA(0.1 0.1 0.2 0.2 0.15 0.3 0.2 0.5 0.1 0.2)
assert 1, eqObj(re, NULL NULL NULL NULL NULL 5.788743 -7.291889 7.031123 -24.039933 -16.766359, 6)

@testing: case = "factorDoubleEMA_with_null"
re = factorDoubleEMA(NULL 0.1 0.2 0.2 0.15 NULL 0.2 0.5 0.1 0.2)
assert 1, eqObj(re, NULL NULL NULL NULL NULL NULL 63.641310 60.256608  8.156385 -0.134531, 6)

@testing: case = "factorDoubleEMA_streaming"
try{dropStreamEngine("factorDoubleEMA")}catch(ex){}
input = table(take(1, 10) as id, 0.1 0.1 0.2 0.2 0.15 0.3 0.2 0.5 0.1 0.2 as price)
out = table(10:0, `id`price, [INT,DOUBLE])
rse = createReactiveStateEngine(name="factorDoubleEMA", metrics=<factorDoubleEMA(price)>, dummyTable=input, outputTable=out, keyColumn='id')
rse.append!(input)
assert 1, eqObj(out.price, NULL NULL NULL NULL NULL 5.788743 -7.291889 7.031123 -24.039933 -16.766359, 6)
```


### 7.3. 并行计算

到现在为止，我们讨论的都是因子的核心逻辑实现，尚未涉及通过并行计算或分布式计算来加快计算速度的问题。在因子计算的工程实践中，可以通过并行来加速的维度包括：证券（股票），因子和时间。在DolphinDB中，实现并行（或分布式）计算的技术路径有以下4个途径。
- 通过SQL语句来实现隐式的并行计算。当SQL语句作用于一个分布式表时，引擎会尽可能下推计算到各个分区执行。
- 创建多个数据源（data source），然后使用mr函数（map reduce）来实现并行计算。
- 用户通过 submitJob 或 submitJobEx 提交多个任务。
- 用peach或ploop实现并行。
我们不建议在因子计算中采用peach或ploop的方式来实现并行。DolphinDB中可用于计算的线程分为两类，分别称之为worker和executor。一般worker用于接受一个任务（job），并将任务分解成多个子任务（task）在本地的executor或远程的worker上执行。一般executor执行的都是本地的耗时比较短的子任务，也就是说在executor上执行的任务一般不会再分解出子任务。peach或ploop将所有的子任务都在本地的exeuctor执行。如果子任务本身再分解出子任务（譬如子任务是一个分布式SQL Query），将严重影响整个系统的吞吐量。
下面我们讨论前三种方法在因子的并行计算中的应用。

#### 7.3.1. 分布式SQL

分布式SQL的第一个应用是计算无状态的因子。对于无状态的因子，即计算本身可能只涉及单条记录内一个或者几个字段。这样的计算可以利用分布式表的机制，在各分区内并行计算。
以 权重偏度因子 为例，此因子计算只用了一个字段，且计算逻辑不涉及前后数据，所以在SQL中调用时，DolphinDB会自动在各分区内并行计算。如果目标数据是内存表，可以使其变为内存分区表，使之并行计算。内存分区表的创建，参考 createPartitionedTable 。

```
resWeight =  select TradeTime, SecurityID, `mathWghtSkew as factorname, mathWghtSkew(BidPrice, w)  as val from loadTable("dfs://LEVEL2_Snapshot_ArrayVector","Snap")  where date(TradeTime) = 2020.01.02
```

分布式SQL的第二个应用场景是计算按标的分组的时序相关因子。对于组内计算的因子，在SQL模式中，将组字段设为分区字段，可以用 context by 组字段并行。如若计算涉及到的数据不跨分区，则可以用 map 语句，加速结果输出。如若计算涉及到的数据跨分区，则SQL会在分区内并行计算，最后在结果部分检查再合并。
以 日内收益率偏度的因子 dayReturnSkew 计算为例， 这个计算本身是需要对标的分组，在组内每天分别做计算。涉及到的数据为分钟频数据，数据源是按月分区，标的 HASH 3 分区。因此，我们在做计算的时候除了可以用 context by 组字段并行以外，还可以用 map 语句加速输出结果。

```
minReturn = select `dayReturnSkew as factorname, dayReturnSkew(close) as val from loadTable("dfs://k_minute_level", "k_minute") where date(tradetime) between 2020.01.02 : 2020.01.31 group by date(tradetime) as tradetime, securityid map
```


#### 7.3.2. map reduce

当用户不想根据分区做并行计算时，可以通过mr函数自定义做并行计算。
以第三章中介绍的 factorDoubleEMA因子 为例。DoubleEMA因子的计算是对标的分组，在组内连续做窗口计算。此类计算由于将窗口的划分会跨时间分区，所以在SQL计算中会先在分区内做计算，然后最后合并再做一次计算，耗时会比较长。
更合理的做法是，如果分区只按照标的分区，那么计算就可以直接在分区内做完而不用合并检查最终结果了。此时可以用 repartitionDS 函数先将原本的数据重新分区再通过map reduce的方式做并行计算。

```
//将原数据按股票重新10个HASH分区
ds = repartitionDS(<select * from loadTable("dfs://k_minute_level", "k_minute") where date(tradetime) between 2020.01.02 : 2020.03.31>, `securityid, HASH,10)

def factorDoubleEMAMap(table){
	return select tradetime, securityid, `doubleEMA as factorname, factorDoubleEMA(close) as val from table context by securityid map
}

res = mr(ds,factorDoubleEMAMap,,unionAll)
```


#### 7.3.3. 通过submitJob提交任务

之前的两种并行计算都是在前台执行的，并行度是由参数 localExecutors 设置。而有些作业可能很大，或者用户不想影响前台使用，此时可以通过 submitJob 提交任务。submitJob的并行度由 maxBatchJobWorker 参数设置。由于后台作业之间是独立的，通常不需要返回到前端的任务都推荐用后台提交 submitJob 的形式。
仍旧以 dayReturnSkew 因子为例。通常我们是需要将因子写入因子库表的，此时可以将整一个过程提交几个后台作业去执行，而在客户端中，同时可以继续做其他计算。由于此例存入的因子库的分区是按月和因子名VALUE分区，故此时应按照月份去提交作业。这样既可以并行写入不会冲突，又可以将作业提交到后台，不影响前台提交其他任务。

```
def writeDayReturnSkew(dBegin,dEnd){
	dReturn = select `dayReturnSkew as factorname, dayReturnSkew(close) as val from loadTable("dfs://k_minute_level", "k_minute") where date(tradetime) between dBegin : dEnd group by date(tradetime) as tradetime, securityid
	//写入因子库
	loadTable("dfs://K_FACTOR_VERTICAL","factor_k").append!(dReturn)
	}

for (i in 0..11){
	dBegin = monthBegin(temporalAdd(2020.01.01,i,"M"))
	dEnd = monthEnd(temporalAdd(2020.01.01,i,"M"))
	submitJob("writeDayReturnSkew","writeDayReturnSkew_"+dBegin+"_"+dEnd, writeDayReturnSkew,dBegin,dEnd)
}
```


### 7.4. 内存管理

内存管理一直是运维人员和研究人员关注的重中之重，本节将从批和流两个角度简单介绍如何在DolphinDB中高效地使用内存。更多有关内存管理的详细内容，请参阅 DolphinDB内存管理教程 。
在配置 DolphinDB 环境时，计算和事务的内存占用可在单节点的 ”dolphindb.cfg” 或集群的 cluster.cfg 中，通过参数”maxMemSize“配置单节点最大可用内存。
- 批处理的内存管理 如 sql模式的例子 ，若对半年的快照数据做操作，批处理方式的中间变量占用内存达到21GB，如果设置的内存小于21GB，则会报Out of Memory错误。这种情况下可以将作业拆分后再提交。 在调试大任务量的计算完成后，可通过 undef 函数将变量赋值为 NULL，或者关闭 session 来及时释放变量的内存。
- 流计算的内存管理 如 数据回放的例子 ，代码中对中间流表调用了函数 enableTableShareAndPersistence 以持久化，指定缓存开始为80万行。当流表数据量超过80万行时，旧的数据会持久化到磁盘上，以空出内存里的空间供新数据写入，这样该流表就可以连续处理远远超过80万行的数据。

### 7.5. 权限管理

因子数据是非常重要的数据，一般来说，用户并不能随意访问所有因子，因此需要对因子数据做好权限管理。DolphinDB database 提供了强大、灵活、安全的权限控制系统，可以满足因子库表级，函数视图级的管理。更多有关权限管理的详细内容，请参考 权限管理教程 。
在实际的生产中通常使用以下三种管理方式：
- 研发人员是管理员，完全掌握数据库 这种情况可以授予研发组 DB_OWNER 的权限(创建数据库并管理其创建的数据库的权限)，使其可以自行创建数据库、表，并对自己创建的数据、表进行权限管理。 login("admin", "123456"); createUser("user1", "passwd1") grant("user1", DB_OWNER)
- 运维人员管理数据库，研发人员只有库表的读写权限 这种情况，数据库管理人员可以将数据表的权限授予给因子研发人员，或者创建一个group组，将权限授予这个组，再将需要权限的人员添加到这个组中统一进行管理。 //以用户的方式进行授权 createUser("user1", "passwd1") grant("user1", TABLE_READ, "dfs://db1/pt1") //以group的方式进行授权 createGroup("group1name", "user1") grant("group1name", TABLE_READ, "dfs://db1/pt1")

### 7.6. 任务管理

因子计算的任务通常分为全量计算所有因子任务、交互式单因子重算任务、所有因子增量计算任务这三种，本章会对每一种因子计算任务进行详细介绍。
因子任务可以通过以下三种方式执行:
- 通过交互的方式执行。
- 通过 submitJob 提交一个Job来执行。
- 通过 scheduleJob 提交一个定时任务来进行周期性的执行。

#### 7.6.1. 全量计算

因子的全量跑批任务，通常是系统初始化因子数据时的一次性任务，或者较长周期进行一次的任务，这类任务可以通过单次触发或者定时任务(scheduleJob)的方式进行管理。
- 单次触发的任务：这种任务可以通过 gui 直接执行，也可以通过 api 来调用命令，最好的方式是通过 submitJob 函数提交任务。通过 submitJob 提交的任务，会提交到服务器的Job 队列中执行，不再受客户端影响，并且可以通过 getRecentJobs 观察到任务是否完成。 //对于跑批的任务封装函数 def bacthExeCute(){} // 通过summitjob进行提交 submitJob("batchTask","batchTask", bacthExeCute)
- 周期性任务：如果计算的因子频率较低需要每天盘后或者其他周期定期全部重算一次，那我们可以使用定时任务(ScheduleJob)的方式进行管理。 //设置一段时间每天执行 scheduleJob(jobId=`daily, jobDesc="Daily Job 1", jobFunc=bacthExeCute, scheduleTime=17:23m, startDate=2018.01.01, endDate=2018.12.31, frequency='D')

#### 7.6.2. 因子运维管理

在因子研发过程中，当碰到因子算法、参数调整的情况，我们会需要对该因子进行重新计算，同时需要将计算的新的因子数据更新到数据库中，对于因子更新的频率通常我们有两种方式：
- 因子的数据频率较高，数据量很大 因子的数据频率较高，数据量很大时，我们推荐在因子数据分区时拉长时间维度，以因子名进行VALUE分区。这样可以使每个因子的数据独立的保存在一个分区中，控制分区大小在一个合适的范围。当我们碰到因子重算的情况，便可以用 dropPartition 函数先删除这个因子所对应的分区数据，然后直接重算这个因子并保存到数据表中。
- 因子的数据频率较低，因子的总数据量较小 当因子的数据频率较低，因子的总数据量较小时，如若将每个因子划分为独立的分区会使得每个分区特别小，而过小的分区可能会影响写入速度。这种情况下，我们可以按照因子 HASH 分区。使用 update! 来进行因子数据更新操作，或使用 upsert 来进行插入更新操作。此外，对于 TSDB 引擎，可以设置参数 keepDuplicates=LAST , 此时可以直接使用 append! 或者 tableInsert 插入数据，从而达到效率更高的更新数据的效果。
update! , upsert 以及 TSDB 引擎特殊设置下的直接 append! 覆盖数据，这三种更新操作都建议在数据量较小，且更新不频繁的情况下使用。对于需要大量因子重算的数据更新的场景，我们推荐使用 单因子独立分区 的方式。当因子重算时先用 dropPartition 函数删除因子所在分区，再重算写入新因子入库。

## 8. 实际案例


### 8.1. 日频因子

日频的数据，一般是由逐笔数据或者其他高频数据聚合而成。日频的数据量不大，在日频数据上经常会计算一些动量因子，或者一些复杂的需要观察长期数据的因子。因此在分区考虑上，建议按年分区即可。在因子计算上，日频因子通常会涉及时间和股票多个维度，因此建议用面板模式计算。当然也可以根据不同存储模式，选择不同的计算模式。
在 日频因子全流程代码汇总 中，模拟了 10 年 4000 只股票的数据，总数据量压缩前大约为 1 GB。代码中会展现上述教程中所涉及日频因子的最佳实践，因子包括 Alpha 1、Alpha 98 ，以及不同计算方式（面板或者SQL模式）写入单值模型、多值模型的最佳实践。

### 8.2. 分钟频因子

分钟频的数据，一般是从逐笔数据或快照数据合成而来。分钟频的数据相比日频的数据较大，在分区设计上建议按月VALUE分区，股票HASH的组合分区。在分钟频的数据上，一般会计算日内的收益率等因子。对于这类因子，建议使用SQL的方式以字段作为参数。很多时候，会将投研的因子，在每日收盘之后，增量做所有因子的计算，此时，也需要对于每日增量的因子做工程化管理。建议将所有此类因子用维度表做一个维护，用定时作业将这些因子批量做计算。
在 分钟频因子全流程代码汇总 中，模拟了一年4000只股票的数据，总数据量压缩前大约20GB。其中，会展现上述教程中所有涉及分钟频率的因子的最佳实践，因子包括日内收益偏度因子，factorDoubleEMA等因子，，后续将因子写入单值模型、多值模型的全过程，以及每日增量计算所有因子的工程化最佳实践。

### 8.3. 快照因子

快照数据，一般指3s一条的多档数据。在实际生产中，往往会根据这样的数据产生实时的因子，或根据多档报价、成交量计算，或根据重要字段做计算。这一类因子，推荐使用字段名作为自定义函数的参数。除此之外，由于快照数据的多档的特殊性，普通存储会占用很大的空间，故在存储模式上，我们也推荐将多档数据存为ArrayVector的形式。如此一来，既能节省磁盘空间，又能使代码简洁，省去选取多个重复字段的困扰。
在 快照因子全流程代码汇总 中，模拟数据生成了20天快照数据，并将其存储为了普通快照数据和ArrayVector快照数据两种。代码中也展示了对于有状态因子flow和无状态因子权重偏度的在流批一体中的最佳实践。

### 8.4. 逐笔因子

逐笔成交数据，是交易所提供的最详细的每一笔撮合成交数据。每3秒发布一次，每次提供这3秒内的所有撮合记录。涉及逐笔成交数据的因子都是高频因子，推荐调试建模阶段可以在小数据量上使用批处理计算。一旦模型定型，就可以用批处理中同样的计算代码，迁移到流计算中实时处理(这就是所谓的批流一体)，比批处理方式节省内存，同时实时性也更高，模型迭代也更快。
在 逐笔因子全流程代码汇总 中，会展现上述教程中所有涉及逐笔成交数据的因子计算、流计算。

## 9. 总结

用DolphinDB来进行因子的计算时，可选择面板和SQL两种方式来封装因子的核心逻辑。面板方式使用矩阵来计算因子，实现思路非常简练；而SQL方式要求投研人员使用向量化的思路进行因子开发。无论哪种方式，DolphinDB均支持批流一体的实现。DolphinDB内置了相关性和回归分析等计算工具，可分析因子的有效性，可对多因子建模。
在因子库的规划上，如果追求灵活性，建议采用单值纵表模型。如果追求效率和性能，推荐使用TSDB引擎，启用多值宽表模式，标的（股票代码）作为表的列。
最后，基于大部分团队的IT和投研相对独立的事实，给出了在代码管理上的工程化方案，投研团队通过模块和自定义函数封装核心因子业务逻辑，IT团队则维护框架代码。同时利用权限模块有效隔离各团队之间的数据访问权限。

## 10. 附录

Alpha #1流式计算
流计算doubleEma因子
python接口订阅流数据
通过ZMQ消息队列收取DolphinDB推送来的流数据
流计算因子结果推送到外部ZMQ消息队列
日频因子全流程代码汇总
分钟频因子全流程代码汇总
快照因子全流程代码汇总
逐笔因子全流程代码汇总


---

> 来源: `window_cal.html`


# 窗口计算

在时序处理中经常需要使用窗口计算。DolphinDB 提供了强大的窗口计算函数，既可以处理数据表(使用 SQL 语句)，又可处理矩阵，在流式计算中亦可使用。第2-4章分别对这三种情况进行详细解释。
DolphinDB 的窗口函数在使用上十分灵活，可嵌套多个内置或自定义函数。在此基础上，DolphinDB 还对窗口计算进行了精心优化，与其他系统相比拥有显著的性能优势。
本篇将系统的介绍 DolphinDB 的窗口计算，从概念划分、应用场景、指标计算等角度，帮助用户快速掌握和运用 DolphinDB 强大的窗口计算功能。
DolphinDB 1.30.15，2.00.3及以上版本支持本篇所有代码。此外,1.30.7，2.00.0以上版本支持绝大部分代码，细节部分会在小节内部详细说明。

## 1. 窗口的概念及分类

DolphinDB 内有五种窗口，分别是：滚动窗口、滑动窗口、累计窗口和会话窗口和 segment window。
在 DolphinDB 中，窗口的度量标准有两种：数据记录数和时间。

### 1.1. 滚动窗口

滚动窗口长度固定，且相邻两个窗口没有重复的元素。
滚动窗口根据度量标准的不同可以分为2种：
- 以记录数划分窗口
图1-1-1 指定窗口长度为3行记录，横坐标以时间为单位。从图上可以看出，按每三行记录划分为了一个窗口，窗口之间的元素没有重叠。
- 以时间划分窗口
下图指定窗口大小为3个时间单位，横坐标以时间为单位。从图上可以看出，按每三个时间单位划分一个窗口，窗口之间的元素没有重叠 ，且窗口内的记录数是不固定的。

### 1.2. 滑动窗口

滑动窗口，即指定长度的窗口根据步长进行滑动。与滚动窗口不同，滑动窗口相邻两个窗口可能包括重复的元素。滑动窗口的窗口长度与步长既可为记录数，亦可为时间长度。
滑动窗口根据度量标准的不同可以分为2种：
- 以记录数划分窗口
图1-2-1 指定窗口大小为6行记录，窗口每次向后滑动1行记录。
- 以时间划分窗口 步长为1行 图1-2-2 指定窗口大小为3个时间单位，窗口以右边界为基准进行前向计算，窗口每次向后滑动1行记录。 步长为指定时间长度 图1-2-3 指定窗口大小为4个时间单位，每次向后滑动2个时间单位。

### 1.3. 累计窗口

累计窗口，即窗口的起始边界固定，结束边界不断右移，因此窗口长度不断增加。 累计窗口根据度量标准的不同可以分为2种：
- 步长为1行
图1-3-1 窗口右边界每次右移1行，窗口大小累计增加。
- 步长为指定时间长度
图1-3-2 窗口右边界每次右移2个时间单位，窗口大小累计增加。

### 1.4. 会话窗口

会话窗口之间的切分，是依据某段时长的空白：若某条数据之后指定时间长度内无数据进入，则该条数据为一个窗口的终点，之后第一条新数据为下一个窗口的起点。会话窗口的窗口长度可变。

#### 1.4.1. segment 窗口

segment 窗口根据连续的相同元素切分窗口。其窗口长度可变。

## 2. 对数据表使用窗口计算（使用 SQL 语句）

本章将介绍 DolphinDB 在 SQL 中的窗口计算：滚动窗口、滑动窗口、累计窗口，segment 窗口，以及窗口连接。

### 2.1. 滚动窗口


#### 2.1.1. 时间维度的滚动窗口

在SQL中，可使用 interval , bar , dailyAlignedBar 等函数配合 group by 语句实现滚动窗口的聚合计算。
下面的例子根据10:00:00到10:05:59每秒更新的数据，使用 bar 函数每2分钟统计一次交易量之和：

```
t=table(2021.11.01T10:00:00..2021.11.01T10:05:59 as time, 1..360 as volume)
select sum(volume) from t group by bar(time, 2m)

// output

bar_time            sum_volume
------------------- ----------
2021.11.01T10:00:00 7260      
2021.11.01T10:02:00 21660     
2021.11.01T10:04:00 36060
```

bar 函数的分组规则是将每条记录最近的能整除 interval 参数的时间作为开始时间。对于给定窗口起始时刻（且该时刻不能被 interval 整除）的场景， bar 函数不适用。 在金融场景中，在交易时段之外也存在一些数据输入，但是在做数据分析的时候并不会用到这些数据；在期货市场，通常涉及到两个交易时间段，有些时段会隔天。 dailyAlignedBar 函数可以设置每天的起始时间和结束时间，很好地解决了这类场景的聚合计算问题。
以期货市场为例，数据模拟为国内期货市场的两个交易时段，分别为下午1:30-3:00和晚上9:00-凌晨2:30。使用 dailyAlignedBar 函数计算每个交易时段中的7分钟均价。

```
sessions = 13:30:00 21:00:00
ts = 2021.11.01T13:30:00..2021.11.01T15:00:00 join 2021.11.01T21:00:00..2021.11.02T02:30:00
ts = ts join (ts+60*60*24)
t = table(ts, rand(10.0, size(ts)) as price)

select avg(price) as price, count(*) as count from t group by dailyAlignedBar(ts, sessions, 7m) as k7

 // output
 
k7                  price             count
------------------- ----------------- -----
2021.11.01T13:30:00 4.815287529108381 420  
2021.11.01T13:37:00 5.265409774828835 420  
2021.11.01T13:44:00 4.984934388122167 420  
...
2021.11.01T14:47:00 5.031795592230213 420  
2021.11.01T14:54:00 5.201864532018313 361  
2021.11.01T21:00:00 4.945093814017518 420 


//如果使用bar函数会不达预期
select avg(price) as price, count(*) as count from t group by bar(ts, 7m) as k7

 // output

k7                  price             count
------------------- ----------------- -----
2021.11.01T13:26:00 5.220721067537347 180       //时间从13:26:00开始，不符合预期
2021.11.01T13:33:00 4.836406542137931 420  
2021.11.01T13:40:00 5.100716347573325 420  
2021.11.01T13:47:00 5.041169475132067 420  
2021.11.01T13:54:00 4.853431270784876 420  
2021.11.01T14:01:00 4.826169502311608 420
```

期货市场中有一些不活跃的期货，一段时间内可能都没有报价，但是在数据分析的时候需要每2秒输出该期货的数据，这个场景下就需要用到 interval 函数进行插值处理。
在以下示例中，缺失值使用前一个值进行填充。如果同一窗口内有重复值，则用最后一个作为输出值。

```
t=table(2021.01.01T01:00:00+(1..5 join 9..11) as time, take(`CLF1,8) as contract, 50..57 as price)

select last(contract) as contract, last(price) as price from t group by interval(time, 2s,"prev") 

 // output

interval_time       contract price
------------------- -------- -----
2021.01.01T01:00:00 CLF1     50   
2021.01.01T01:00:02 CLF1     52   
2021.01.01T01:00:04 CLF1     54   
2021.01.01T01:00:06 CLF1     54   
2021.01.01T01:00:08 CLF1     55   
2021.01.01T01:00:10 CLF1     57   

//如果使用bar函数会不达预期

select last(contract) as contract, last(price) as price from t group by bar(time, 2s)

bar_time            contract price
------------------- -------- -----
2021.01.01T01:00:00 CLF1     50   
2021.01.01T01:00:02 CLF1     52   
2021.01.01T01:00:04 CLF1     54   
2021.01.01T01:00:08 CLF1     55   
2021.01.01T01:00:10 CLF1     57
```


#### 2.1.2. 记录数维度的滚动窗口

除了时间维度可以做滚动窗口计算之外，记录数维度也可以做滚动窗口计算。在股票市场临近收盘的时候，往往一分钟之内的交易量、笔数是非常大的，做策略时如果单从时间维度去触发可能会导致偏差。因此分析师有时会想要从每100笔交易而非每一分钟的角度去做策略，这个时候就可以用 rolling 函数实现。
下面是某天股票市场最后一分钟内对每100笔交易做成交量之和的例子：

```
t=table(2021.01.05T02:59:00.000+(1..2000)*30 as time, take(`CL,2000) as sym, 10* rand(50, 2000) as vol)

select rolling(last,time,100,100) as last_time,rolling(last,t.sym,100,100) as sym, rolling(sum,vol,100,100) as vol_100_sum from t

 // output (每次结果会因为rand函数结果而不同)

last_time               sym vol_100_sum
----------------------- --- -----------
2021.01.05T02:59:03.000	CL	24,900
2021.01.05T02:59:06.000	CL	24,390
2021.01.05T02:59:09.000	CL	24,340
2021.01.05T02:59:12.000	CL	24,110
2021.01.05T02:59:15.000	CL	23,550
2021.01.05T02:59:18.000	CL	25,530
2021.01.05T02:59:21.000	CL	26,700
2021.01.05T02:59:24.000	CL	26,790
2021.01.05T02:59:27.000	CL	27,090
2021.01.05T02:59:30.000	CL	25,610
2021.01.05T02:59:33.000	CL	23,710
2021.01.05T02:59:36.000	CL	23,920
2021.01.05T02:59:39.000	CL	23,000
2021.01.05T02:59:42.000	CL	24,490
2021.01.05T02:59:45.000	CL	23,810
2021.01.05T02:59:48.000	CL	22,230
2021.01.05T02:59:51.000	CL	25,380
2021.01.05T02:59:54.000	CL	25,830
2021.01.05T02:59:57.000	CL	24,020
2021.01.05T03:00:00.000	CL	25,150
```


### 2.2. 滑动窗口

使用滑动窗口处理表数据有以下四种情况：

#### 2.2.1. 步长为1行，窗口长度为 n 行

此类情况可使用 m 系列函数， moving 函数，或者 rolling 函数。
从1.30.16/2.00.4版本开始，亦可使用 window 函数。 window 函数与 moving 函数类似，均为高阶函数，不同的是， window 函数更为灵活，不同于 moving 函数的窗口右边界是固定的， window 函数的左右边界均可自由设定。
下面以 msum 为例，滑动计算窗口长度为5行的vol值之和。

```
t=table(2021.11.01T10:00:00 + 0 1 2 5 6 9 10 17 18 30 as time, 1..10 as vol)

select time, vol, msum(vol,5,1) from t

 // output

time                vol msum_vol
------------------- --- --------
2021.11.01T10:00:00 1   1       
2021.11.01T10:00:01 2   3       
2021.11.01T10:00:02 3   6       
2021.11.01T10:00:05 4   10      
2021.11.01T10:00:06 5   15    
...
```

DolphinDB SQL可以通过 context by 对各个不同的 symbol 在组内进行窗口计算。 context by 是DolphinDB 独有的功能，是对标准 SQL 语句的拓展，具体其他用法参照： context by

```
t=table(2021.11.01T10:00:00 + 0 1 2 5 6 9 10 17 18 30 join 0 1 2 5 6 9 10 17 18 30 as time, 1..20 as vol, take(`A,10) join take(`B,10) as sym)

select time, sym, vol, msum(vol,5,1) from t context by sym

 // output

time                sym vol msum_vol
------------------- --- --- --------
2021.11.01T10:00:00 A   1   1       
2021.11.01T10:00:01 A   2   3       
2021.11.01T10:00:02 A   3   6       
...    
2021.11.01T10:00:30 A   10  40      
2021.11.01T10:00:00 B   11  11      
2021.11.01T10:00:01 B   12  23      
...    
2021.11.01T10:00:30 B   20  90
```

m 系列函数是经过优化的窗口函数，如果想要使用自定义函数做窗口计算，DolphinDB 支持在 moving 函数、 window 函数和 rolling 函数中使用自定义聚合函数。下面以 moving 嵌套自定义聚合函数为例：
以下的行情数据有四列(代码，日期，close 和 volume)，按照代码分组，组内按日期排序。设定窗口大小为20，在窗口期内按照 volume 排序，取 volume 最大的五条数据的平均 close 的计算。

```
//t是模拟的四列数据
t = table(take(`IBM, 100) as code, 2020.01.01 + 1..100 as date, rand(100,100) + 20 as volume, rand(10,100) + 100.0 as close)

//1.30.15及以上版本可以用一行代码实现
//moving 支持用户使用自定义匿名聚合函数
select code, date, moving(defg(vol, close){return close[isort(vol, false).subarray(0:min(5,close.size()))].avg()}, (volume, close), 20) from t context by code 

//其他版本可以用自定义命名聚合函数实现：
defg top_5_close(vol,close){
return close[isort(vol, false).subarray(0:min(5,close.size()))].avg()
}
select code, date, moving(top_5_close,(volume, close), 20) from t context by code
```

在做数据分析的时候，还会经常用到窗口嵌套窗口的操作。 举一个更复杂的例子：在做 101 Formulaic Alphas 中98号因子计算的时候，DolphinDB可以运用窗口嵌套窗口的方法，将原本在C#中需要几百行的代码，简化成几行代码，且计算性能也有接近三个数量级的提升。 trade 表有需要可以自行模拟数据，或用 sample 数据 CNTRADE 。

```
// 输入表trade的schema如下，如需要可自行模拟数据。

name       typeString typeInt 
---------- ---------- ------- 
ts_code    SYMBOL     17             
trade_date DATE       6              
open       DOUBLE     16             
vol        DOUBLE     16             
amount     DOUBLE     16    

// alpha 98 计算：

def normRank(x){
	return rank(x)\x.size()
}

def alpha98SQL(t){
	update t set adv5 = mavg(vol, 5), adv15 = mavg(vol, 15) context by ts_code
	update t set rank_open = normRank(open), rank_adv15 = normRank(adv15) context by trade_date
	update t set decay7 = mavg(mcorr(vwap, msum(adv5, 26), 5), 1..7), decay8 = mavg(mrank(9 - mimin(mcorr(rank_open, rank_adv15, 21), 9), true, 7), 1..8) context by ts_code
	return select ts_code, trade_date, normRank(decay7)-normRank(decay8) as a98 from t context by trade_date 
}

input = select trade_date,ts_code,amount*1000/(vol*100 + 1) as vwap,vol,open from trade
timer alpha98DDBSql = alpha98SQL(input)
```


#### 2.2.2. 步长为1行，窗口为指定时间长度

此类情况可使用 tm 系列或者 tmoving 系列函数。
从1.30.16/2.00.4版本开始，亦可使用 twindow 函数。 twindow 函数与 tmoving 函数类似，均为高阶函数，不同的是， twindow 函数更为灵活，不同于 tmoving 函数的窗口右边界是固定的， twindow 函数的左右边界均可自由设定。
下面以 tmsum 为例，计算滑动窗口长度为5秒的 vol 值之和。

```
//1.30.14，2.00.2以上版本支持 tmsum 函数。
t=table(2021.11.01T10:00:00 + 0 1 2 5 6 9 10 17 18 30 as time, 1..10 as vol)
select time, vol, tmsum(time,vol,5s) from t

 // output
time                vol tmsum_time
------------------- --- ----------
2021.11.01T10:00:00 1   1         
2021.11.01T10:00:01 2   3         
2021.11.01T10:00:02 3   6         
2021.11.01T10:00:05 4   9         
2021.11.01T10:00:06 5   12        
2021.11.01T10:00:09 6   15        
2021.11.01T10:00:10 7   18        
2021.11.01T10:00:17 8   8         
2021.11.01T10:00:18 9   17        
2021.11.01T10:00:30 10  10
```

实际场景中，计算历史分位的时候也会广泛运用到这类情况的窗口计算，具体在 步长为1行窗口为n行 这一小节介绍。

#### 2.2.3. 步长为 n 行，窗口为 m 行

此类情况可使用高阶函数 rolling 。
下面的例子计算步长为3行，窗口长度为6行的 vol 值之和。与 interval 函数不同的是， rolling 不会对缺失值进行插值，如果窗口内的元素个数不足窗口大小，该窗口不会被输出。 该例子中，数据一共是10条，在前两个窗口计算完之后，第三个窗口因为只有4条数据，所以不输出第三个窗口的结果。

```
t=table(2021.11.01T10:00:00+0 3 5 6 7 8 15 18 20 29 as time, 1..10 as vol)
select rolling(last,time,6,3) as last_time, rolling(sum,vol,6,3) as sum_vol from t

 // output

last_time           sum_vol
------------------- -------
2021.11.01T10:00:08 21     
2021.11.01T10:00:20 39
```


#### 2.2.4. 步长为指定时间长度，窗口为 n 个步长时间

此类情况可使用 interval 函数配合 group by 语句。下面的例子以5秒为窗口步长，10秒为窗口长度，计算 vol 值之和。
推荐使用1.30.14, 2.00.2及以上版本使用 interval 函数。
2.1.1.1中 interval 的场景可以看作是窗口长度与步长相等的特殊的滑动窗口，而本节则是窗口长度为 n 倍步长时间的滑动窗口。

### 2.3. 累计窗口

累计窗口有两种情况：一种是步长是1行，另一种是步长为指定时间长度。

#### 2.3.1. 步长为1行

步长为1行的累计窗口计算在 SQL 中通常直接用 cum 系列函数。下面是累计求和 cumsum 的例子：

```
t=table(2021.11.01T10:00:00..2021.11.01T10:00:04 join 2021.11.01T10:00:06..2021.11.01T10:00:10 as time,1..10 as vol)
select *, cumsum(vol) from t 

// output

time                vol cum_vol
------------------- --- -------
2021.11.01T10:00:00 1   1      
2021.11.01T10:00:01 2   3      
2021.11.01T10:00:02 3   6      
2021.11.01T10:00:03 4   10     
2021.11.01T10:00:04 5   15     
2021.11.01T10:00:06 6   21     
2021.11.01T10:00:07 7   28     
2021.11.01T10:00:08 8   36     
2021.11.01T10:00:09 9   45     
2021.11.01T10:00:10 10  55
```

在实际场景中经常会用 cum 系列函数与 context by 连用，做分组内累计计算。比如行情数据中，根据各个不同股票的代码，做各自的累计成交量。

#### 2.3.2. 步长为指定时间长度

要在SQL中实现步长为指定时间长度的累计窗口计算，可以使用 bar 函数搭配 cgroup by 来实现。

### 2.4. segment 窗口

以上所有例子中，窗口大小均固定。在 DolphinDB 中亦可将连续的相同元素作为一个窗口，用 segment 来实现。实际场景中， segment 经常用于逐笔数据中。
下面的例子是根据 order_type 中的数据进行窗口分割，连续相同的 order_type 做累计成交额计算。

```
vol = 0.1 0.2 0.1 0.2 0.1 0.2 0.1 0.2 0.1 0.2 0.1 0.2
order_type = 0 0 1 1 1 2 2 1 1 3 3 2;
t = table(vol,order_type);
select *, cumsum(vol) as cumsum_vol from t context by segment(order_type);

// output

vol order_type cumsum_vol
--- ---------- ----------
0.1 0          0.1       
0.2 0          0.3       
0.1 1          0.1       
0.2 1          0.3       
0.1 1          0.4       
0.2 2          0.2       
0.1 2          0.3       
0.2 1          0.2       
0.1 1          0.3       
0.2 3          0.2       
0.1 3          0.3       
0.2 2          0.2
```


### 2.5. 窗口连接计算

在 DolphinDB 中，除了常规的窗口计算之外，还支持窗口连接计算。即在表连接的同时，进行窗口计算。可以通过 wj 和 pwj 函数来实现 。
window join 基于左表每条记录的时间戳，确定一个时间窗口，并计算对应时间窗口内右表的数据。左表每滑动一条记录，都会与右表窗口计算的结果连接。因为窗口的左右边界均可以指定，也可以为负数，所以也可以看作非常灵活的滑动窗口。
详细用法参见用户手册 window join 。

```
//data
t1 = table(1 1 2 as sym, 09:56:06 09:56:07 09:56:06 as time, 10.6 10.7 20.6 as price)
t2 = table(take(1,10) join take(2,10) as sym, take(09:56:00+1..10,20) as time, (10+(1..10)\10-0.05) join (20+(1..10)\10-0.05) as bid, (10+(1..10)\10+0.05) join (20+(1..10)\10+0.05) as offer, take(100 300 800 200 600, 20) as volume);

//window join
wj(t1, t2, -5s:0s, <avg(bid)>, `sym`time);

// output

sym time     price  avg_bid           
--- -------- ----- -------
1   09:56:06 10.6 10.3
1   09:56:07 10.7 10.4
2   09:56:06 20.6 20.3
```

由于窗口可以灵活设置，所以不仅是多表连接的时候会用到，单表内部的窗口计算也可以用到 window join 。下面的例子可以看作是 t2 表中每一条数据做一个 (time-6s) 到 (time+1s) 的计算。

```
t2 = table(take(1,10) join take(2,10) as sym, take(09:56:00+1..10,20) as time, (10+(1..10)\10-0.05) join (20+(1..10)\10-0.05) as bid, (10+(1..10)\10+0.05) join (20+(1..10)\10+0.05) as offer, take(100 300 800 200 600, 20) as volume);

wj(t2, t2, -6s:1s, <avg(bid)>, `sym`time);

// output

sym time     bid   offer volume avg_bid           
--- -------- ---- ------ ------ --------
1   09:56:01 10.05 10.15 100    10.1
...  
1   09:56:08 10.75 10.85 800    10.5              
1   09:56:09 10.85 10.95 200    10.6
1   09:56:10 10.95 11.05 600    10.65             
2   09:56:01 20.05 20.15 100    20.1
2   09:56:02 20.15 20.25 300    20.15
...
2   09:56:08 20.75 20.85 800    20.5              
2   09:56:09 20.85 20.9  200    20.6
2   09:56:10 20.95 21.05 600    20.65
```

从1.30.16/2.00.4版本开始，亦可使用 window 函数以及 twindow 函数实现单表内部的灵活窗口计算。
以上 wj 的代码也可以用 twindow 或 window 实现：

## 3. 对矩阵使用窗口计算

表的窗口计算在前一章节已经描述，所以在这一章节中着重讨论矩阵的计算。

### 3.1. 矩阵的滑动窗口计算

滑动窗口 m 系列函数以及 window 函数可以用于处理矩阵，在矩阵每列内进行计算，返回一个与输入矩阵维度相同的矩阵。如果滑动维度为时间，则要先使用 setIndexedMatrix! 函数将矩阵的行与列标签设为索引。这里需要注意的是，行与列标签均须严格递增。
首先我们新建一个矩阵，并将其设为 IndexedMatrix：

```
m=matrix(1..4 join 6, 11..13 join 8..9)
m.rename!(2020.01.01..2020.01.04 join 2020.01.06,`A`B)
m.setIndexedMatrix!();
```


#### 3.1.1. 步长为1行，窗口为n行

m 系列函数的参数可以是一个正整数（记录数维度）或一个 duration（时间维度）。通过设定不同的参数，可以指定理想的滑动窗口类型。
以 msum 滑动求和为例。以下例子是对一个矩阵内部，对每一列进行窗口长度为3行的滑动求和计算。

```
msum(m,3,1)

// output

           A  B 
           -- --
2020.01.01|1  11
2020.01.02|3  23
2020.01.03|6  36
2020.01.04|9  33
2020.01.06|13 30
```

矩阵运算中，也可以做复杂的窗口嵌套。曾在2.2.1节中提到的98号因子也可以在矩阵中通过几行代码实现（trade 表有需要可以自行模拟数据，或用 sample 数据 CNTRADE ）：

```
// 输入表trade的schema如下，如需要可自行模拟数据：

name       typeString typeInt 
---------- ---------- ------- 
ts_code    SYMBOL     17             
trade_date DATE       6              
open       DOUBLE     16             
vol        DOUBLE     16             
amount     DOUBLE     16   

// alpha 98 的矩阵计算

def prepareDataForDDBPanel(){
	t = select trade_date,ts_code,amount*1000/(vol*100 + 1) as vwap,vol,open from trade 
	return dict(`vwap`open`vol, panel(t.trade_date, t.ts_code, [t.vwap, t.open, t.vol]))
}

def myrank(x) {
	return rowRank(x)\x.columns()
}

def alpha98Panel(vwap, open, vol){
	return myrank(mavg(mcorr(vwap, msum(mavg(vol, 5), 26), 5), 1..7)) - myrank(mavg(mrank(9 - mimin(mcorr(myrank(open), myrank(mavg(vol, 15)), 21), 9), true, 7), 1..8))
}

input = prepareDataForDDBPanel()
alpha98DDBPanel = alpha98Panel(input.vwap, input.open, input.vol)
```


#### 3.1.2. 步长为1行，窗口为指定时间

以 msum 滑动求和为例。以下例子是对一个矩阵内部，每一列根据左边的时间列进行窗口大小为3天的滑动求和计算。

```
msum(m,3d)

// output

           A  B 
           -- --
2020.01.01|1  11
2020.01.02|3  23
2020.01.03|6  36
2020.01.04|9  33
2020.01.06|10 17
```

在实际运用中，这类矩阵窗口运算非常常见。比如在做历史分位的计算中，将数据转化为 IndexedMatrix 之后，直接用一行代码就可以得到结果了。
下面例子对 m 矩阵做10年的历史分位计算：

```
//推荐使用1.30.14, 2.00.2及以上版本来使用 mrank 函数。
mrank(m, true, 10y, percent=true)

// output
           A B   
           - ----
2020.01.01|1 1   
2020.01.02|1 1   
2020.01.03|1 1   
2020.01.04|1 0.25
2020.01.06|1 0.4
```


### 3.2. 矩阵的累计窗口计算

在矩阵中，累计函数 cum 系列也可以直接使用。以 cumsum 为例：

```
cumsum(m)

 // output 

            A  B 
           -- --
2020.01.01|1  11
2020.01.02|3  23
2020.01.03|6  36
2020.01.04|10 44
2020.01.06|16 53
```

结果为在矩阵的每一列，计算累计和。

## 4. 流式数据的窗口计算

在 DolphindDB 中，设计了许多内置的流计算引擎。有些支持聚合计算，有些则支持滑动窗口或者累计窗口计算，也有针对于流数据的会话窗口引擎，可以满足不同的场景需求。下面根据不同窗口以及引擎分别介绍。

### 4.1. 滚动窗口在流计算中的应用

实际场景中，滚动窗口计算在流数据中的应用最为广泛，比如5分钟 k 线，1分钟累计交易量等。滚动窗口在流计算中的应用通过各种时间序列引擎实现。
createTimeSeriesEngine 时间序列引擎应用广泛，类似的引擎还有 createDailyTimeSeriesEngine 与 createSessionWindowEngine 。 createDailyTimeSeriesEngine 与 dailyAlignedBar 类似，可以指定时间段进行窗口计算，而非按照流入数据的时间窗口聚合计算。 createSessionWindowEngine 会在4.3中详细介绍。 本节以 createTimeSeriesEngine 为例。
下例中，时间序列引擎 timeSeries1 订阅流数据表 trades，实时计算表 trades 中过去1分钟内每只股票交易量之和。

```
share streamTable(1000:0, `time`sym`volume, [TIMESTAMP, SYMBOL, INT]) as trades
output1 = table(10000:0, `time`sym`sumVolume, [TIMESTAMP, SYMBOL, INT])
timeSeries1 = createTimeSeriesEngine(name="timeSeries1", windowSize=60000, step=60000, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=output1, timeColumn=`time, useSystemTime=false, keyColumn=`sym, garbageSize=50, useWindowStartTime=false)
subscribeTable(tableName="trades", actionName="timeSeries1", offset=0, handler=append!{timeSeries1}, msgAsTable=true);

insert into trades values(2018.10.08T01:01:01.785,`A,10)
insert into trades values(2018.10.08T01:01:02.125,`B,26)
insert into trades values(2018.10.08T01:01:10.263,`B,14)
insert into trades values(2018.10.08T01:01:12.457,`A,28)
insert into trades values(2018.10.08T01:02:10.789,`A,15)
insert into trades values(2018.10.08T01:02:12.005,`B,9)
insert into trades values(2018.10.08T01:02:30.021,`A,10)
insert into trades values(2018.10.08T01:04:02.236,`A,29)
insert into trades values(2018.10.08T01:04:04.412,`B,32)
insert into trades values(2018.10.08T01:04:05.152,`B,23)

sleep(10)

select * from output1;

 // output

time                    sym sumVolume
----------------------- --- ---------
2018.10.08T01:02:00.000 A   38       
2018.10.08T01:02:00.000 B   40       
2018.10.08T01:03:00.000 A   25       
2018.10.08T01:03:00.000 B   9       


//to drop the time series engine
dropStreamEngine(`timeSeries1)
unsubscribeTable(tableName="trades", actionName="timeSeries1")
undef("trades",SHARED)
```


### 4.2. 滑动、累计窗口在流计算中的应用

另一个常用的引擎是响应式状态引擎 createReactiveStateEngine 。在这个引擎中，我们可以使用经过优化的状态函数，其中包括累计窗口函数（cum 系列函数）和滑动窗口函数（m 系列函数以及 tm 系列函数）。
createReactiveStateEngine 响应式状态引擎的功能非常强大，可以让流数据像 SQL 一样处理，实现批流一体。下面的例子同时展示了 cum 系列函数，m 系列函数和 tm 系列函数在 createReactiveStateEngine 响应式状态引擎中的作用。

```
//1.30.14，2.00.2以上版本支持tmsum函数。
share streamTable(1000:0, `time`sym`volume, [TIMESTAMP, SYMBOL, INT]) as trades
output2 = table(10000:0, `sym`time`Volume`msumVolume`cumsumVolume`tmsumVolume, [ SYMBOL,TIMESTAMP,INT, INT,INT,INT])
reactiveState1= createReactiveStateEngine(name="reactiveState1", metrics=[<time>,<Volume>,<msum(volume,2,1)>,<cumsum(volume)>,<tmsum(time,volume,2m)>], dummyTable=trades, outputTable=output2, keyColumn="sym")
subscribeTable(tableName="trades", actionName="reactiveState1", offset=0, handler=append!{reactiveState1}, msgAsTable=true);

insert into trades values(2018.10.08T01:01:01.785,`A,10)
insert into trades values(2018.10.08T01:01:02.125,`B,26)
insert into trades values(2018.10.08T01:01:10.263,`B,14)
insert into trades values(2018.10.08T01:01:12.457,`A,28)
insert into trades values(2018.10.08T01:02:10.789,`A,15)
insert into trades values(2018.10.08T01:02:12.005,`B,9)
insert into trades values(2018.10.08T01:02:30.021,`A,10)
insert into trades values(2018.10.08T01:04:02.236,`A,29)
insert into trades values(2018.10.08T01:04:04.412,`B,32)
insert into trades values(2018.10.08T01:04:05.152,`B,23)

sleep(10)

select * from output2

 // output

sym time                    Volume msumVolume cumsumVolume tmsumVolume
--- ----------------------- ------ ---------- ------------ -----------
A   2018.10.08T01:01:01.785 10     10         10           10         
B   2018.10.08T01:01:02.125 26     26         26           26         
A   2018.10.08T01:01:12.457 28     38         38           38         
B   2018.10.08T01:01:10.263 14     40         40           40         
A   2018.10.08T01:02:10.789 15     43         53           53         
B   2018.10.08T01:02:12.005 9      23         49           49         
A   2018.10.08T01:02:30.021 10     25         63           63         
A   2018.10.08T01:04:02.236 29     39         92           54         
B   2018.10.08T01:04:04.412 32     41         81           41         
B   2018.10.08T01:04:05.152 23     55         104          64           

//to drop the reactive state engine

dropAggregator(`reactiveState1)
unsubscribeTable(tableName="trades", actionName="reactiveState1")
undef("trades",SHARED)
```


### 4.3. 会话窗口引擎

createSessionWindowEngine 可以根据间隔时间（session gap）切分不同的窗口，即当一个窗口在session gap 时间内没有接收到新数据时，窗口会关闭。所以这个引擎中的window size会根据流入数据的情况发生变化。

```
share streamTable(1000:0, `time`volume, [TIMESTAMP, INT]) as trades
output1 = keyedTable(`time,10000:0, `time`sumVolume, [TIMESTAMP, INT])
engine_sw = createSessionWindowEngine(name = "engine_sw", sessionGap = 5, metrics = <sum(volume)>, dummyTable = trades, outputTable = output1, timeColumn = `time)
subscribeTable(tableName="trades", actionName="append_engine_sw", offset=0, handler=append!{engine_sw}, msgAsTable=true)

n = 5
timev = 2018.10.12T10:01:00.000 + (1..n)
volumev = (1..n)%1000
insert into trades values(timev, volumev)

n = 5
timev = 2018.10.12T10:01:00.010 + (1..n)
volumev = (1..n)%1000
insert into trades values(timev, volumev)

n = 3
timev = 2018.10.12T10:01:00.020 + (1..n)
volumev = (1..n)%1000
timev.append!(2018.10.12T10:01:00.027 + (1..n))
volumev.append!((1..n)%1000)
insert into trades values(timev, volumev)

select * from trades;

//传入数据如下：

 time                    volume
----------------------- ------
2018.10.12T10:01:00.001 1     
2018.10.12T10:01:00.002 2     
2018.10.12T10:01:00.003 3     
2018.10.12T10:01:00.004 4     
2018.10.12T10:01:00.005 5     
2018.10.12T10:01:00.011 1     
2018.10.12T10:01:00.012 2     
2018.10.12T10:01:00.013 3     
2018.10.12T10:01:00.014 4     
2018.10.12T10:01:00.015 5     
2018.10.12T10:01:00.021 1     
2018.10.12T10:01:00.022 2     
2018.10.12T10:01:00.023 3     
2018.10.12T10:01:00.028 1     
2018.10.12T10:01:00.029 2     
2018.10.12T10:01:00.030 3    


//经过createSessionWindowEngine会话窗口引擎后，根据session gap=5(ms)聚合形成的窗口计算结果为：
select * from output1

time                    sumVolume
----------------------- ---------
2018.10.12T10:01:00.001 15       
2018.10.12T10:01:00.011 15       
2018.10.12T10:01:00.021 6    

// to drop SessionWindowEngine

unsubscribeTable(tableName="trades", actionName="append_engine_sw")
dropAggregator(`engine_sw)
undef("trades",SHARED)
```


## 5. 窗口计算的空值处理规则

在 DolphinDB 中，各个窗口函数的空值处理略有不同，本节将阐述各个系列函数空值处理的规则：

### 5.1. moving，m 系列函数，tm 系列函数以及 cum 系列函数的空值处理

在 mrank ， tmrank 以及 cumrank 函数中，可以指定 NULL 值是否参与计算。其他窗口函数与聚合函数保持一致，计算时忽略 NULL 值。
moving 以及大部分 m 系列函数参数里都有一个可选参数 minPeriods 。若没有指定 minPeriods ，结果的前 ( window - 1) 个元素为NULL；若指定了 minPeriods ，结果的前 ( minPeriods - 1) 个元素为 NULL。如果窗口中的值全为 NULL，该窗口的计算结果为 NULL。 minPeriods 的默认值为 window 之值。

```
m=matrix(1..5, 6 7 8 NULL 10)

//不指定 minPeriods 时，由于 minPeriods 默认值与 window 相等，所以结果的前二行均为 NULL。

msum(m,3)

 #0 #1
-- --
     
     
6  21
9  15
12 18

//若指定 minPeriods=1，结果的前二行不是 NULL 值。

 msum(m,3,1)

 #0 #1
-- --
1  6 
3  13
6  21
9  15
12 18
```


### 5.2. rolling 的空值处理

与 moving 函数不同的是， rolling 函数不输出前 ( window - 1) 个元素的 NULL 值结果。可以通过下面的例子来感受：
t 是一个包含 NULL 值的表，我们分别用 rolling 和 moving 对 vol 这一列做窗口为3行的求和计算。

```
vol=1 2 3 4 NULL NULL NULL 6 7 8
t= table(vol)

//rolling做窗口为3行的滑动求和计算
rolling(sum,t.vol,3)

 // output
[6,9,7,4,,6,13,21]

//moving做窗口为3行的滑动求和计算
moving(sum,t.vol,3)

 // output
[,,6,9,7,4,,6,13,21]

//rolling做窗口为3行，步长为2行的窗口计算
rolling(sum,t.vol,3,2)

 // output
[6,7,,13]     ///最后的窗口没有足够的元素时，不会输出
```


## 6. 常用指标的计算复杂度

假设共有 n 个元素，窗口大小为 m，那么常用的 m 系列，tm 系列函数都经过了优化，其时间复杂度为 O(n)，即每一次计算结果只会把位置0去掉，加入新的观察值。 而 mrank 与其他函数稍许不同，计算速度会比其他的慢，原因是其时间复杂度为O(mn)，与其窗口长度有关，窗口越大，复杂度越高。即每一次都会将结果重置。
moving ， tmoving ， rolling , window , twindow 这些高阶函数的复杂度与其参数内的 func 有关，是没有做过优化的。所以每一次滑动都是整个窗口对于 func 函数进行计算，而非 m 系列，tm 系列函数的增量计算。
故相比于 moving ， tmoving ， rolling , window , 和 twindow 这些高阶函数， m 系列和 tm 系列函数对于相同的计算功能会有更好的性能。

```
n=1000000
x=norm(0,1, n);

//moving
timer moving(avg, x, 10);
Time elapsed:  243.331 ms

//rolling
timer rolling(avg, x, 10);
Time elapsed: 599.389ms

//mavg
timer mavg(x, 10);
Time elapsed: 3.501ms
```


## 7. 涉及到窗口计算的函数

| 聚合函数 | m系列 | ReactiveStateEngine 是否支持 | tm系列 | ReactiveStateEngine 是否支持 | cum系列 | ReactiveStateEngine 是否支持 |
| --- | --- | --- | --- | --- | --- | --- |
|  | moving（高阶函数） | √ | tmoving（高阶函数） | √ |  |  |
|  | window（高阶函数） | 可用WndowJoinEngine | twindow（高阶函数） | 可用WndowJoinEngine |  |  |
| avg | mavg | √ | tmavg | √ | cumavg | √ |
| sum | msum | √ | tmsum | √ | cumsum | √ |
| beta | mbeta | √ | tmbeta | √ | cumbeta | √ |
| corr | mcorr | √ | tmcorr | √ | cumcorr | √ |
| count | mcount | √ | tmcount | √ | cumcount | √ |
| covar | mcovar | √ | tmcovar | √ | cumcovar | √ |
| imax | mimax | √ |  |  |  |  |
| imin | mimin | √ |  |  |  |  |
| max | mmax | √ | tmmax | √ | cummax | √ |
| min | mmin | √ | tmmin | √ | cummin | √ |
| first | mfirst | √ | tmfirst | √ |  |  |
| last | mlast | √ | tmlast | √ |  |  |
| med | mmed | √ | tmmed | √ | cummed |  |
| prod | mprod | √ | tmprod | √ | cumprod | √ |
| var | mvar | √ | tmvar | √ | cumvar | √ |
| varp | mvarp | √ | tmvarp | √ | cumvarp | √ |
| std | mstd | √ | tmstd | √ | cumstd | √ |
| stdp | mstdp | √ | tmstdp | √ | cumstdp | √ |
| skew | mskew | √ | tmskew | √ |  |  |
| kurtosis | mkurtosis | √ | tmkurtosis | √ |  |  |
| percentile | mpercentile | √ | tmpercentile | √ | cumpercentile |  |
| rank | mrank | √ | tmrank | √ | cumrank |  |
| wsum | mwsum | √ | tmwsum | √ | cumwsum | √ |
| wavg | mwavg | √ | tmwavg | √ | cumwavg | √ |
| ifirstNot | mifirstNot |  |  |  |  |  |
| ilastNot | milastNot |  |  |  |  |  |
| firstNot |  |  |  |  | cumfirstNot | √ |
| lastNot |  |  |  |  | cumlastNot | √ |
| mad | mmad | √ |  |  |  |  |
|  | move | √ | tmove | √ |  |  |
|  | mslr | √ |  |  |  |  |
|  | ema | √ |  |  |  |  |
|  | kama | √ |  |  |  |  |
|  | sma | √ |  |  |  |  |
|  | wma | √ |  |  |  |  |
|  | dema | √ |  |  |  |  |
|  | tema | √ |  |  |  |  |
|  | trima | √ |  |  |  |  |
|  | t3 | √ |  |  |  |  |
|  | ma | √ |  |  |  |  |
|  | wilder | √ |  |  |  |  |
|  | gema | √ |  |  |  |  |
|  | linearTimeTrend | √ |  |  |  |  |
| mse | mmse |  |  |  |  |  |
|  |  |  |  |  | cumPositiveStreak |  |
其他涉及窗口的函数： deltas, ratios, interval, bar, dailyAlignedBar, coevent, createReactiveStateEngine, createDailyTimeSeriesEngine, createReactiveStateEngine, createSessionWindowEngine

## 8. 总结

DolphinDB 中的窗口函数功能非常齐全。合理运用窗口，能够简便地实现各种复杂逻辑，使数据分析步骤更简洁，效率更高。


---

> 来源: `panel_data.html`


# 面板数据处理

时间序列数据、截面数据和面板数据是金融领域中常见的数据组织方式。面板数据包含了时间序列和横截面两个维度。在 Python 中，通常可以用 pandas 的 DataFrame 或 numpy 的二维数组来表示。在 DolphinDB 中面板数据也可以用表（table）或矩阵（matrix）来表示。
本教程主要介绍如何在 DolphinDB 中表示和分析面板数据。本文的所有例子都基于 DolphinDB 1.30.16/2.00.4。

## 1. 面板数据的表示方法和处理函数

DolphinDB 提供了两种方法处理面板数据：
- 通过 SQL 和向量化函数来处理用二维表表示的面板数据
- 通过向量化函数来处理用矩阵表示的面板数据
DolphinDB 中数据表和矩阵都采用了列式存储。以下是表和矩阵中的列常用的计算函数和二元运算符:
- 二元运算符：+, -, *, /, ratio, %, &&, ||, &, |, pow
- 序列函数：ratios, deltas, prev, next, move
- 滑动窗口函数：mcount，mavg, msum, mmax, mimax, mimin, mmin, mprod, mstd, mvar, mmed, mpercentile, mrank, mwavg, mwsum, mbeta, mcorr, mcovar
- 累计窗口函数：cumcount, cumavg, cumsum, cummax, cummin, cumprod, cumstd, cumvar, cummed, cumpercentile, cumPositiveStreak, cumrank, cumwavg, cumwsum, cumbeta, cumcorr, cumcovar
- 聚合函数：count, avg, sum, sum2, first, firstNot, last, lastNot, max, min, std, var, med, mode, percentile, atImax, atImin, wavg, wsum, beta, corr, covar
- row 系列函数（针对面板数据的每一行进行计算）：rowCount, rowAvg, rowSum, rowSum2, rowProd, rowMax, rowMin, rowStd, rowVar, rowBeta, rowCorr, rowAnd, rowOr, rowXor
下文通过举例的方式让读者更能了解这些函数是如何进行面板数据操作。

## 2. SQL 语句处理面板数据

当使用 DolphinDB 的二维数据表来表示 SQL 的面板数据时，通常一个列存储一个指标，譬如 open, high, low, close, volume 等，一行代表一个股票在一个时间点的数据。这样的好处是，多个指标进行处理时，不再需要对齐数据。缺点是分组计算（按股票分组的时间序列计算，或者按时间分组的横截面计算），需要先分组。SQL 语句的 group by/context by/pivot by 子句均可用于分组。分组有一定的开销，通常尽可能把所有的计算在一次分组内全部计算完成。
DolphinDB 的 SQL 不仅支持 SQL 的标准功能，还进行了扩展，包括面板数据处理，非同时连接，窗口连接，窗口函数等。本章节中会分别展示如何用 SQL 语句处理面板数据。
首先，模拟一份含有 3 种股票代码的数据。这份数据在之后的例子中都会用到：

```
sym = `C`C`C`C`MS`MS`MS`IBM`IBM
timestamp = [09:34:57,09:34:59,09:35:01,09:35:02,09:34:57,09:34:59,09:35:01,09:35:01,09:35:02]
price= 50.6 50.62 50.63 50.64 29.46 29.48 29.5 174.97 175.02
volume = 2200 1900 2100 3200 6800 5400 1300 2500 8800
t = table(sym, timestamp, price, volume);
t;

// output
sym timestamp price  volume
--- --------- ------ ------
C   09:34:57  50.6   2200
C   09:34:59  50.62  1900
C   09:35:01  50.63  2100
C   09:35:02  50.64  3200
MS  09:34:57  29.46  6800
MS  09:34:59  29.48  5400
MS  09:35:01  29.5   1300
IBM 09:35:01  174.97 2500
IBM 09:35:02  175.02 8800
```


### 2.1. context by

context by 是 DolphinDB 独有的功能，是对标准 SQL 语句的拓展，我们可以通过 context by 子句实现的分组计算功能来简化对数据面板的操作。
SQL 的 group by 子句将数据分成多组，每组产生一个值，也就是一行。因此使用 group by 子句后，行数一般会大大减少。
在对面板数据进行分组后，每一组数据通常是时间序列数据，譬如按股票分组，每一个组内的数据是一个股票的价格序列。处理面板数据时，有时候希望保持每个组的数据行数，也就是为组内的每一行数据生成一个值。例如，根据一个股票的价格序列生成回报序列，或者根据价格序列生成一个移动平均价格序列。其它数据库系统（例如 SQL Server, PostgreSQL），用窗口函数（window function）来解决这个问题。DolpinDB 引入了 context by 子句来处理面板数据。context by 与 group by, pivot by 一起组成了 DolphinDB 分组数据处理系统。它与窗口函数相比，除了语法更简洁以外，表达能力上也更强大，具体表现在：
- 不仅能与 select 配合查询数据，也可以与 update 配合更新数据。
- 绝大多数数据库系统在窗口函数中只能使用表中现有的字段分组。context by 子句可以使用任何现有字段和计算字段。
- 绝大多数数据库系统的窗口函数仅限于少数几个函数。context by 不仅不限制使用的函数，而且可以使用任意表达式，譬如多个函数的组合。
- context by 可以与 having 子句配合使用，以过滤每个组内部的行。
(1) 按股票代码进行分组，应用序列函数计算每一只股票的前后交易量比率，进行对比：

```
select timestamp, sym, price, ratios(volume) ,volume from t context by sym;

// output
timestamp sym price  ratios_volume volume
--------- --- ------ ------------- ------
09:34:57  C   50.6                  2200
09:34:59  C   50.62  0.86           1900
09:35:01  C   50.63  1.106          2100
09:35:02  C   50.64  1.52           3200
09:35:01  IBM 174.97                2500
09:35:02  IBM 175.02 3.52           8800
09:34:57  MS  29.46                 6800
09:34:59  MS  29.48  0.79           5400
09:35:01  MS  29.5   0.24           1300
```

(2) 结合滑动窗口函数，计算每只股票在 3 次数据更新中的平均价格：

```
select *, mavg(price,3) from t context by sym;

// output
sym timestamp price  volume mavg_price
--- --------- ------ ------ -----------
C   09:34:57  50.60  2200
C   09:34:59  50.62  1900
C   09:35:01  50.63  2100   50.62
C   09:35:02  50.64  3200   50.63
IBM 09:35:01  174.97 2500
IBM 09:35:02  175.02 8800
MS  09:34:57  29.46  6800
MS  09:34:59  29.48  5400
MS  09:35:01  29.50  1300   29.48
```

(3) 结合累计窗口函数，计算每只股票在每一次的数据更新中最大交易量：

```
select timestamp, sym, price,volume, cummax(volume) from t context by sym;

// output
timestamp sym price  volume cummax_volume
--------- --- ------ ------ -------------
09:34:57  C   50.6   2200   2200
09:34:59  C   50.62  1900   2200
09:35:01  C   50.63  2100   2200
09:35:02  C   50.64  3200   3200
09:35:01  IBM 174.97 2500   2500
09:35:02  IBM 175.02 8800   8800
09:34:57  MS  29.46  6800   6800
09:34:59  MS  29.48  5400   6800
09:35:01  MS  29.5   1300   6800
```

(4) 应用聚合函数，计算每只股票在每分钟中的最大交易量：

```
select *, max(volume) from t context by sym, timestamp.minute();

// output
sym timestamp price  volume max_volume
--- --------- ------ ------ ----------
C   09:34:57  50.61  2200   2200
C   09:34:59  50.62  1900   2200
C   09:35:01  50.63  2100   3200
C   09:35:02  50.64  3200   3200
IBM 09:35:01  174.97 2500   8800
IBM 09:35:02  175.02 8800   8800
MS  09:34:57  29.46  6800   6800
MS  09:34:59  29.48  5400   6800
MS  09:35:01  29.5   1300   1300
```


### 2.2. pivot by

pivot by 是 DolphinDB 的独有功能，是对标准 SQL 语句的拓展，可将数据表中某列的内容按照两个维度整理，产生数据表或矩阵。
通过应用 pivot by 子句，可以对于数据表 t 进行重新排列整理：每行为一秒钟，每列为一只股票，既能够了解单个股票每个时刻的变化，也可以了解各股票之间的差异。
如：对比同一时间段不同股票的交易价格：

```
select price from t pivot by timestamp, sym;

// output
timestamp C     IBM    MS
--------- ----- ------ -----
09:34:57  50.6         29.46
09:34:59  50.62        29.48
09:35:01  50.63 174.97 29.5
09:35:02  50.64 175.02
```

pivot by 还可以与聚合函数一起使用。比如，将数据中每分钟的平均收盘价转换为数据表：

```
select avg(price) from t where sym in `C`IBM pivot by minute(timestamp) as minute, sym;

// output
minute C      IBM
------ ------ -------
09:34m 50.61
09:35m 50.635 174.995
```

pivot by 与 select 子句一起使用时返回一个表，而和 exec 语句一起使用时返回一个矩阵：

```
resM = exec avg(price) from t where sym in `C`IBM pivot by minute(timestamp) as minute, sym;
resM

// output
       C      IBM
       ------ -------
09:34m|50.61
09:35m|50.635 174.995
```


```
typestr(resM)

// output
FAST DOUBLE MATRIX
```


## 3. 向量化函数处理面板数据

当使用 DolphinDB 的矩阵来表示面板数据时，数据按时间序列和横截面两个维度进行排列。
对矩阵表示的面板数据进行分析时，如：每行是按时间戳排序的时间点，每列是一只股票，我们既可以对某一只股票进行多个时间点的动态变化分析，也可以了解多个股票之间在某个时点的差异情况。
向量化函数 panel 可将一列或多列数据转换为矩阵。例如，将数据表 t 中的 price 列转换为一个矩阵：

```
price = panel(t.timestamp, t.sym, t.price);
price;

// output
         C     IBM    MS
         ----- ------ -----
09:34:57|50.60        29.46
09:34:59|50.62        29.48
09:35:01|50.63 174.97 29.5
09:35:02|50.64 175.02
```

以下脚本将 price 与 volume 列分别转换为矩阵。返回的结果是一个元组，每个元素对应一列转换而来的矩阵。

```
price, volume = panel(t.timestamp, t.sym, [t.price, t.volume]);
```

使用 panel 函数时，可以指定结果矩阵的行与列的标签。这里需要注意，行与列的标签均需严格升序。例如：

```
rowLabel = 09:34:59..09:35:02;
colLabel = ["C", "MS"];
volume = panel(t.timestamp, t.sym, t.volume, rowLabel, colLabel);
volume;

// output
         C    MS
         ---- ----
09:34:59|1900 5400
09:35:00|
09:35:01|2100 1300
09:35:02|3200
```

使用 rowNames 和 colNames 函数可以获取 panel 函数返回的矩阵的行和列标签：

```
volume.rowNames();
volume.colNames();
```

如果后续要对面板数据做一步的计算和处理，推荐使用矩阵来表示面板数据。这是因为矩阵天然支持向量化操作和二元操作，计算效率会更高，代码会更简洁。

### 3.1. 矩阵操作示例

下文例举了矩阵形式面板数据的常用操作。
(1) 通过序列函数，对每个股票的相邻价格进行比较。

```
price = panel(t.timestamp, t.sym, t.price);
deltas(price);

// output
         C    IBM  MS
         ---- ---- -----
09:34:57|
09:34:59|0.02      0.02
09:35:01|0.01      0.02
09:35:02|0.01 0.05
```

(2) 结合滑动窗口函数，计算每只股票在每 2 次数据更新中的平均价格。

```
mavg(price,2);

// output
         C      IBM      MS
         ------ ------ -----------
09:34:57|
09:34:59|50.61         29.47
09:35:01|50.63  174.97 29.49
09:35:02|50.63  175.00 29.50
```

(3) 结合累计窗口函数，计算每只股票中价格的排序。

```
cumrank(price);

// output
         C IBM MS
         - --- --
09:34:57|0     0
09:34:59|1     1
09:35:01|2 0   2
09:35:02|3 1
```

(4) 通过聚合函数, 得到每只股票中的最低价格。

```
min(price);

// output
[50.60,174.97,29.46]
```

(5) 通过聚合函数，得到每一个同时间段的最低股票价格。

```
rowMin(price);

 // output
[29.46,29.48,29.5,50.64]
```


### 3.2. 对齐矩阵的二次运算

普通矩阵进行二元运算时，按照对应元素分别进行计算，需要保持维度 (shape) 一致，DolphinDB 提供了矩阵对齐的方法，使得矩阵计算不再受维度的限制。
在 1.30.20/2.00.8 版本前，用户需要通过 indexedMatrix 和 indexedSeries 来支持矩阵的对齐运算，其标签必须是严格递增的。
- indexedMatrix：以行列标签为索引的矩阵。
- indexedSeries：带索引标签的向量。
indexedMatrix 和 indexedSeries 在进行二元运算时，系统会自动以 "outer join" 的方式对齐，然后进行运算。
1.30.20/2.00.8 版本后，DolphinDB 提供了用于矩阵对齐的函数 align ，拓展了标签矩阵的对齐功能，使矩阵对齐和运算更加灵活。
(1) indexedSeries 之间的对齐运算
两个 indexedSeries 进行二元操作，会根据 index 进行对齐再做计算。

```
index1 = 2020.11.01..2020.11.06;
value1 = 1..6;
s1 = indexedSeries(index1, value1);

index2 = 2020.11.04..2020.11.09;
value2 =4..9;
s2 = indexedSeries(index2, value2);

s1+s2;

 // output
           #0
           --
2020.11.01|
2020.11.02|
2020.11.03|
2020.11.04|8
2020.11.05|10
2020.11.06|12
2020.11.07|
2020.11.08|
2020.11.09|
```

(2) indexedMatrix 之间的对齐运算
两个 indexedMatrix 进行二元操作，其对齐的方法和 indexedSeries 一致。

```
id1 = 2020.11.01..2020.11.06;
m1 = matrix(1..6, 7..12, 13..18).rename!(id1, `a`b`d)
m1.setIndexedMatrix!()

id2 = 2020.11.04..2020.11.09;
m2 = matrix(4..9, 10..15, 16..21).rename!(id2, `a`b`c)
m2.setIndexedMatrix!()

m1+m2;

 // output
           a  b  c d
           -- -- - -
2020.11.01|         
2020.11.02|         
2020.11.03|         
2020.11.04|8  20    
2020.11.05|10 22    
2020.11.06|12 24    
2020.11.07|         
2020.11.08|         
2020.11.09|
```

(3) indexedSeries 和 indexedMatrix 之间的对齐运算
indexedSeries 与 indexedMatrix 进行二元操作，会根据行标签进行对齐，indexedSeries 与 indexedMatrix 的每列进行计算。

```
m1=matrix(1..6, 11..16);
m1.rename!(2020.11.04..2020.11.09, `A`B);
m1.setIndexedMatrix!();
m1;

 // output
           A B
           - --
2020.11.04|1 11
2020.11.05|2 12
2020.11.06|3 13
2020.11.07|4 14
2020.11.08|5 15
2020.11.09|6 16

s1;

 // output
           #0
           --
2020.11.01|1
2020.11.02|2
2020.11.03|3
2020.11.04|4
2020.11.05|5
2020.11.06|6

m1 + s1;

 // output
           A B
           - --
2020.11.01|
2020.11.02|
2020.11.03|
2020.11.04|5 15
2020.11.05|7 17
2020.11.06|9 19
2020.11.07|
2020.11.08|
2020.11.09|
```

(4) 使用 align 函数进行对齐

```
x1 = [09:00:00, 09:00:01, 09:00:03]
x2 = [09:00:00, 09:00:03, 09:00:03, 09:00:04]
y1 = `a`a`b
y2 = `a`b`b
m1 = matrix(1 2 3, 2 3 4, 3 4 5).rename!(y1,x1)
m2 = matrix(11 12 13, 12 13 14, 13 14 15, 14 15 16).rename!(y2,x2)
a, b = align(m1, m2, 'ej,aj', false);
a;

// output
  09:00:00 09:00:01 09:00:03
  -------- -------- --------
a|1        2        3       
a|2        3        4       
b|3        4        5      

b;

// output
  09:00:00 09:00:01 09:00:03
  -------- -------- --------
a|11       11       13      
b|12       12       14      
b|13       13       15
```


### 3.3. 重采样和频度转换

DolphinDB 提供了 resample 和 asfreq 函数，用于对有时间类型索引的 indexedSeries 或者 indexedMatrix 进行重采样和频度转换。
其实现目的是为用户提供一个对常规时间序列数据重新采样和频率转换的便捷的方法。

#### 3.3.1. resample（重采样）

重采样是指将时间序列的频度转换为另一个频度。重采样时必须指定一个聚合函数对数据进行计算。

```
index=2020.01.01..2020.06.30;
s=indexedSeries(index, take(1,size(index)));
s.resample("M",sum);

 // output
           #0
           --
2020.01.31|31
2020.02.29|29
2020.03.31|31
2020.04.30|30
2020.05.31|31
2020.06.30|30
```


#### 3.3.2. asfreq（频率转换）

asfreq 函数转换给定数据的时间频率。与 resample 函数不同，asfreq 不能使用聚合函数对数据进行处理。asfreq 通常应用于将低频转时间换为高频时间的场景，且与各类 fill 函数配合使用。

```
index=2020.01.01 2020.01.05 2020.01.10
s=indexedSeries(index, take(1,size(index)));
s.asfreq("D").ffill()

 // output
            #0
           --
2020.01.01|1
2020.01.02|1
2020.01.03|1
2020.01.04|1
2020.01.05|1
2020.01.06|1
2020.01.07|1
2020.01.08|1
2020.01.09|1
2020.01.10|1
```


#### 3.3.3. NULL 值的处理

在重采样和频率转换中，可能需要对结果的 NULL 值进行处理。具体的处理方法请参考： 矩阵运算教程 。

### 3.4. 矩阵聚合


#### 3.4.1. 列聚合

对矩阵应用内置的向量函数、聚合函数以及窗口函数，计算都是按列进行的。
以对某个矩阵应用求和 sum 为例：

```
m = rand(10, 20)$10:2
sum(m)

// output
[69, 38]
```

可以看出，矩阵每一列都被单独视为一个向量进行计算。
自定义函数，若要应用到矩阵每列单独计算，可以通过高阶函数 each 实现。

```
m = rand(10, 20)$10:2
m

// output
#0 #1
-- --
6  6 
9  2 
7  0 
5  5 
8  8 
8  1 
8  4 
7  8 
4  3 
7  0 

def mfunc(x, flag){if(flag==1) return sum(x); else return avg(x)}
each(mfunc, m, 0 1)

// output
[6.5, 38]
```


#### 3.4.2. 行聚合

DolphinDB 提供了按行进行运算的高阶函数 byRow，以及一系列内置的 row 函数（参见 row 系列函数）。
以对某个矩阵应用 row 函数 rowCount 为例：

```
m=matrix([4.5 NULL 1.5, 1.5 4.8 5.9, 4.9 2.0 NULL]);
rowCount(m);

// output
[3,2,2]
```

rowCount 统计每行非空的元素个数，返回一个长度和原矩阵行数相同的向量。
自定义函数，若要应用到矩阵每行单独计算，可以通过高阶函数 byRow 实现。复用自定义函数 mfunc 和矩阵 m：

```
byRow(mfunc{, 0}, m)

// output
[6,5.5,3.5,5,8,4.5,6,7.5,3.5,3.5]
```


#### 3.4.3. 分组聚合

数据表的分组聚合可以通过 SQL 的 group by 语句实现；而通过 regroup 函数，可以实现矩阵的分组聚合操作。
根据给出的时间标签将一个价格矩阵进行分组聚合：

```
timestamp = 09:00:00 + rand(10000, 1000).sort!()
id= rand(['st1', 'st2'], 1000)
price = (190 + rand(10.0, 2000))$1000:2
regroup(price, minute(timestamp), avg, true)
```

对于 pivot by 产生的面板矩阵，按照 label 进行聚合，可以通过 rowNames 或者 colNames 获取标签：

```
n=1000
timestamp = 09:00:00 + rand(10000, n).sort!()
id = take(`st1`st2`st3, n)
vol = 100 + rand(10.0, n)
t = table(timestamp, id, vol)

m = exec vol from t pivot by timestamp, id
regroup(m,minute(m.rowNames()), avg)
```


## 4. 面板数据处理方式的对比

下面以一个更为复杂的实际例子演示如何高效的解决面板数据问题。著名论文 101 Formulaic Alphas 给出了世界顶级量化对冲基金 WorldQuant 所使用的 101 个因子公式，其中里面 80% 的因子仍然行之有效并被运用在实盘项目中。
这里选取了 WorldQuant 公开的 Alpha98 因子的表达式。

```
alpha_098 = (rank(decay_linear(correlation(((high_0+low_0+open_0+close_0)*0.25), sum(mean(volume_0,5), 26.4719), 4.58418), 7.18088)) -rank(decay_linear(ts_rank(ts_argmin(correlation(rank(open_0), rank(mean(volume_0,15)), 20.8187), 8.62571),6.95668), 8.07206)))
```

为了更好的对比各个处理方式之间的差异，我们选择了一年的股票每日数据，涉及的原始数据量约为 100 万条。如需数据请参考 模拟数据脚本 。
以下是脚本测试所需要的数据, 输入数据为包含以下字段的 table：
- securityid：股票代码
- tradetime：时间日期
- vol：成交量
- vwap：成交量的加权平均价格
- open：开盘价格
- close：收盘价格

### 4.1. DolphinDB SQL 与向量化函数处理面板数据的对比

下例分别使用 DolphinDB SQL 语句和矩阵来实现计算 Alpha98 因子。全部 DolphinDB 脚本请参考 DolphinDB 实现 98 号因子脚本 。
- DolphinDB SQL 语句实现 Alpha98 因子计算的脚本如下：

```
def alpha98(stock){
	t = select securityid, tradetime, vwap, open, mavg(vol, 5) as adv5, mavg(vol,15) as adv15 from stock context by securityid
	update t set rank_open = rank(open), rank_adv15 = rank(adv15) context by tradetime
	update t set decay7 = mavg(mcorr(vwap, msum(adv5, 26), 5), 1..7), decay8 = mavg(mrank(9 - mimin(mcorr(rank_open, rank_adv15, 21), 9), true, 7), 1..8) context by securityid
	return select securityid, tradetime, rank(decay7)-rank(decay8) as A98 from t context by tradetime
}
```


```
t = loadTable("dfs://k_day_level","k_day")
timer alpha98(t)
```

- 以下是在 DolphinDB 中通过向量化函数来计算 98 号因子的脚本：

```
def myrank(x){
	return rowRank(x)\x.columns()
}

def alphaPanel98(vwap, open, vol){
	return myrank(mavg(mcorr(vwap, msum(mavg(vol, 5), 26), 5), 1..7)) - myrank(mavg(mrank(9 - mimin(mcorr(myrank(open), myrank(mavg(vol, 15)), 21), 9), true, 7), 1..8))
}
```


```
t = select * from loadTable("dfs://k_day_level","k_day")
timer vwap, open, vol = panel(t.tradetime, t.securityid, [t.vwap, t.open, t.vol])
timer res = alphaPanel98(vwap, open, vol)
```

通过两个 Alpha98 因子脚本的对比，可以发现用向量化函数来实现 Alpha98 因子的脚本会更加简洁一点。
因为 Alpha98 因子在计算过程中用到了截面数据，也用到了大量时间序列的计算结果。所以在计算某支股票某一天的因子中，既要用到该股票的历史数据，也要用到当天所有股票的信息，对信息量的要求很大。而矩阵形式的面板数据是截面数据和时间序列数据综合起来的一种数据类型，可以支持股票数据按两个维度进行排列，在实现 Alpha98 因子计算中，不需要多次对中间数据或输出数据进行维度转换，简化了计算逻辑。在实现 Alpha98 因子计算的过程中，进行函数嵌套的同时还需要多次进行分组计算来处理数据。对比使用 SQL 语句执行计算，用 panel 函数来处理面板数据，明显计算效率会更高，代码会更简洁。
在性能测试方面，使用单线程计算，SQL 语句计算 Alpha98 因子耗时 610ms。而 panel 函数生成面板数据耗时 70ms，计算 Alpha98 因子耗时 440ms。两者的耗时差异不大，矩阵方式可能略胜一筹。
但是向量化函数处理面板数据也有局限性, 矩阵的面板数据无法进行再次分组，单值模型格式不够直观，而 SQL 支持多列分组，可以联合查询多个字段的信息，适用于海量数据的并行计算。
在处理面板数据时，客户可根据自身对数据的分析需求，来选择不同的方法处理面板数据。

### 4.2. DolphinDB 与 pandas 处理面板数据的性能对比：

pandas 实现 alpha98 因子的部分脚本如下，完整脚本请参考 python 中实现 98 号因子脚本 ：

```
def myrank(x):
    return ((x.rank(axis=1,method='min'))-1)/x.shape[1]

def imin(x):
    return np.where(x==min(x))[0][0]


def rank(x):
    s = pd.Series(x)
    return (s.rank(ascending=True, method="min")[len(s)-1])-1


def alpha98(vwap, open, vol):
    return myrank(vwap.rolling(5).corr(vol.rolling(5).mean().rolling(26).sum()).rolling(7).apply(lambda x: np.sum(np.arange(1, 8)*x)/np.sum(np.arange(1, 8)))) - myrank((9 - myrank(open).rolling(21).corr(myrank(vol.rolling(15).mean())).rolling(9).apply(imin)).rolling(7).apply(rank).rolling(8).apply(lambda x: np.sum(np.arange(1, 9)*x)/np.sum(np.arange(1, 9))))
```


```
start_time = time.time()
re=alpha98(vwap, open, vol)
print("--- %s seconds ---" % (time.time() - start_time))
```

使用 pandas 计算 Alpha98 的耗时为 520s，而使用矩阵实现计算仅耗时 440ms，性能相差千倍。
DolphinDB 内置了许多与时序数据相关的函数，并进行了优化，性能优于其它系统 1~2 个数量级。例如上面例子中使用到的 mavg, mcorr, mrank, mimin, msum 等计算滑动窗口函数。 尤其在计算测试二元滑动窗口（mcorr）中，DolphinDB 的计算耗时 0.6 秒，pandas 耗时 142 秒，性能相差 200 倍以上。为了避免计算结果的偶然性，我们使用了十年的股市收盘价数据，涉及的原始数据量约为 530 万条，对比结果是连续运行十次的耗时。
整体而言，在 Alpha98 因子的计算中，DolphinDB 出现性能上的断层式优势是有迹可循的。

## 5. 附录

- 模拟数据脚本
- DolphinDB 实现 98 号因子脚本
- Python 实现 98 号因子脚本


---

> 来源: `ddb_sql_cases.html`


# SQL 编写案例

本教程重点介绍了一些常见场景下的 SQL 编写案例。介绍如何正确编写 SQL 语句来提升脚本运行性能，通过优化前后性能对比，来说明DolphinDB SQL脚本的编写技巧。包括以下内容：

## 1. 测试环境说明

处理器：Intel(R) Xeon(R) Silver 4216 CPU @ 2.10GHz
操作系统：CentOS Linux release 7.9
License：免费版License，CPU 2核，内存 8GB
DolphinDB Server 版本：DolphinDB_Linux64_V2.00.4，单节点模式部署
DolphinDB GUI 版本：DolphinDB_GUI_V1.30.15
以下章节案例中所用到的2020年06月测试数据为上交所 Level-1 快照数据，基于真实数据结构模拟2000只股票快照数据，基于 OLAP 与 TSDB 存储引擎的建库建表、数据模拟、数据插入脚本如下：

```
model = table(1:0, `SecurityID`DateTime`PreClosePx`OpenPx`HighPx`LowPx`LastPx`Volume`Amount`BidPrice1`BidPrice2`BidPrice3`BidPrice4`BidPrice5`BidOrderQty1`BidOrderQty2`BidOrderQty3`BidOrderQty4`BidOrderQty5`OfferPrice1`OfferPrice2`OfferPrice3`OfferPrice4`OfferPrice5`OfferQty1`OfferQty2`OfferQty3`OfferQty4`OfferQty5, [SYMBOL, DATETIME, DOUBLE, DOUBLE, DOUBLE, DOUBLE, DOUBLE, LONG, DOUBLE, DOUBLE, DOUBLE, DOUBLE, DOUBLE, DOUBLE, LONG, LONG, LONG, LONG, LONG, DOUBLE, DOUBLE, DOUBLE, DOUBLE, DOUBLE, LONG, LONG, LONG, LONG, LONG])

// OLAP 存储引擎建库建表
dbDate = database("", VALUE, 2020.06.01..2020.06.07)
dbSecurityID = database("", HASH, [SYMBOL, 10])
db = database("dfs://Level1", COMPO, [dbDate, dbSecurityID])
createPartitionedTable(db, model, `Snapshot, `DateTime`SecurityID)

// TSDB 存储引擎建库建表
dbDate = database("", VALUE, 2020.06.01..2020.06.07)
dbSymbol = database("", HASH, [SYMBOL, 10])
db = database("dfs://Level1_TSDB", COMPO, [dbDate, dbSymbol], engine="TSDB")
createPartitionedTable(db, model, `Snapshot, `DateTime`SecurityID, sortColumns=`SecurityID`DateTime)

def mockHalfDayData(Date, StartTime) {
	t_SecurityID = table(format(600001..602000, "000000") + ".SH" as SecurityID)
	t_DateTime = table(concatDateTime(Date, StartTime + 1..2400 * 3) as DateTime)
	t = cj(t_SecurityID, t_DateTime)
	size = t.size()
	return  table(t.SecurityID as SecurityID, t.DateTime as DateTime, rand(100.0, size) as PreClosePx, rand(100.0, size) as OpenPx, rand(100.0, size) as HighPx, rand(100.0, size) as LowPx, rand(100.0, size) as LastPx, rand(10000, size) as Volume, rand(100000.0, size) as Amount, rand(100.0, size) as BidPrice1, rand(100.0, size) as BidPrice2, rand(100.0, size) as BidPrice3, rand(100.0, size) as BidPrice4, rand(100.0, size) as BidPrice5, rand(100000, size) as BidOrderQty1, rand(100000, size) as BidOrderQty2, rand(100000, size) as BidOrderQty3, rand(100000, size) as BidOrderQty4, rand(100000, size) as BidOrderQty5, rand(100.0, size) as OfferPrice1, rand(100.0, size) as OfferPrice2, rand(100.0, size) as OfferPrice3, rand(100.0, size) as OfferPrice4, rand(100.0, size) as OfferPrice5, rand(100000, size) as OfferQty1, rand(100000, size) as OfferQty2, rand(100000, size) as OfferQty3, rand(100000, size) as OfferQty4, rand(100000, size) as OfferQty5)
}

def mockData(DateVector, StartTimeVector) {
	for(Date in DateVector) {
		for(StartTime in StartTimeVector) {
			data = mockHalfDayData(Date, StartTime)

			// OLAP 存储引擎分布式表插入模拟数据
			loadTable("dfs://Level1", "Snapshot").append!(data)

			// TSDB 存储引擎分布式表插入模拟数据
			loadTable("dfs://Level1_TSDB", "Snapshot").append!(data)
		}
	}
}

mockData(2020.06.01..2020.06.02, 09:30:00 13:00:00)
```


## 2. 条件过滤相关案例

where 条件子句包含一个或多个条件表达式，根据表达式指定的过滤条件，可以过滤出满足需求的记录。
条件表达式中可以使用 DolphinDB 内置函数，如聚合、序列、向量函数，也可以使用用户自定义函数。需要注意的是，DolphinDB 不支持在分布式查询的 where 子句中使用聚合函数，如 sum 、 count 。因为执行聚合函数之前，分布式查询需要通过 where 子句来筛选相关分区的数据，达到分区剪枝的效果，减少查询耗时。如果聚合函数出现在 where 子句中，则分布式查询不能缩窄相关分区范围。

### 2.1. where 条件子句使用 in 关键字

场景：数据表 t1 含有股票的某些信息，数据表 t2 含有股票的行业信息，需要根据股票的行业信息进行过滤。
首先，载入测试数据库中的表 “Snapshot” 赋给变量 t1，并模拟构建行业信息数据表 t2，示例如下：

```
t1 = loadTable("dfs://Level1", "Snapshot")
SecurityIDs = exec distinct SecurityID from t1 where date(DateTime) = 2020.06.01
t2 = table(SecurityIDs as SecurityID, 
           take(`Mul`IoT`Eco`Csm`Edu`Food, SecurityIDs.size()) as Industry)
```


#### 2.1.1. 优化前

将数据表 t1 与数据表 t2 根据 SecurityID 字段进行 left join，然后指定 where 条件进行过滤，示例如下：

```
timer res1 = select SecurityID, DateTime 
			 from lj(t1, t2, `SecurityID) 
			 where date(DateTime) = 2020.06.01, Industry=`Edu
```

查询耗时 336 ms。
需要注意的是，以上脚本中的 timer 函数通常用于计算一行或一段脚本的执行时间，该时间指的是脚本在 DolphinDB Server 端的运行耗时，而不包括脚本运行结果集返回到客户端的耗时。若结果集数据量过大，序列化/反序列化以及网络传输的耗时可能会远远超过脚本在服务器上的运行耗时。

#### 2.1.2. 优化后

从数据表 t2 获取行业为 “Edu” 的股票代码向量，并使用 in 关键字指定条件范围，示例如下：

```
SecurityIDs = exec SecurityID from t2 where Industry="Edu"
timer res2 = select SecurityID, DateTime 
			 from t1 
			 where date(DateTime) = 2020.06.01, SecurityID in SecurityIDs
```

查询耗时 72 ms。

```
each(eqObj, res1.values(), res2.values()) // true
```

each 函数对表的每列分别通过 eqObj 比较，返回均为 true，说明优化前后返回的结果相同。但与优化前写法相比，优化后写法查询性能提升约4倍。这是因为，在 SQL 语句中，表连接的耗时远高于 where 子句中的过滤条件的耗时，因此在能够使用字典或 in 关键字的情况下应避免使用 join。

### 2.2. 分组数据过滤

场景：截取单日全市场股票交易快照数据，筛选出每只股票交易量最大的前 25% 的记录。
首先，载入测试数据库表并将该表对象赋值给变量 snapshot，之后可以直接引用变量 snapshot，示例如下：

```
snapshot = loadTable("dfs://Level1", "Snapshot")
```

使用 context by 对于股票分组，并根据 Volume 字段计算 75% 分位点的线性插值作为最小值，示例如下：

```
timer res1 = select * from snapshot 
			 where date(DateTime) = 2020.06.01 
			 context by SecurityID having Volume >= percentile(Volume, 75, "linear")
```

context by 是 DolphinDB SQL 引入的一个关键词，用于分组计算。与 group by 用于聚合不同，context by 只是对数据分组而不做聚合操作，因此不改变数据的记录数。
having 子句总是跟在 group by 或者 context by 后，用来将结果进行过滤，只返回满足指定条件的聚合函数值的组结果。having 与 group by 搭配使用时，表示是否输出某个组的结果。having 与 context by 搭配使用时，既可以表示是否输出这个组的结果，也可以表示输出组中的哪些行。
场景：承接以上场景，选出每只股票交易量最大的 25% 的记录后，计算 LastPx 的标准差。

#### 2.2.1. 优化前

使用 context by 对股票分组，并根据 Volume 字段计算 75% 位置处的线性插值作为过滤条件的最小值，再根据 group by 对股票分组，并计算标准差，最后使用 order by 对于股票排序，示例如下：

```
timer select std(LastPx) as std from (
      select SecurityID, LastPx from snapshot 
      where date(DateTime) = 2020.06.01 
      context by SecurityID 
      having Volume >= percentile(Volume, 75, "linear")) 
      group by SecurityID 
      order by SecurityID
```


#### 2.2.2. 优化后

使用 group by 对股票分组，aggrTopN 高阶函数选择交易量最大的 25% 的记录，并计算标准差。示例如下：

```
timer select aggrTopN(std, LastPx, Volume, 0.25, false) as std from snapshot 
	  where date(DateTime) = 2020.06.01 
	  group by SecurityID 
	  order by SecurityID
```

优化前先把数据分组并进行过滤，合并数据后再分组计算聚合值。优化后，在数据分组后，直接进行过滤和聚合，减少了中间步骤，从而提升了性能。

### 2.3. where 条件子句使用逗号或 and

where 子句中多条件如果使用 “,” 进行连接时，在查询时会按照顺序对 “,” 前的条件层层进行过滤；若使用 and 进行连接时，会对所有条件在原表内分别进行筛选后再将结果取交集。
下面将通过几个示例，比较使用 and 和逗号再不同场景下进行条件过滤的异同。
首先，产生模拟数据，示例如下：

```
N = 10000000
t = table(take(2019.01.01..2019.01.03, N) as date, 			  
          take(`C`MS`MS`MS`IBM`IBM`IBM`C`C$SYMBOL, N) as sym, 
          take(49.6 29.46 29.52 30.02 174.97 175.23 50.76 50.32 51.29, N) as price, 
          take(2200 1900 2100 3200 6800 5400 1300 2500 8800, N) as qty)
```

根据过滤条件是否使用序列相关函数，如 deltas , ratios , ffill , move , prev , cumsum 等，可以分为以下两种情况。

#### 2.3.1. 过滤条件与序列无关


```
timer(10) t1 = select * from t where qty > 2000, date = 2019.01.02, sym = `C
timer(10) t2 = select * from t where qty > 2000 and date = 2019.01.02 and sym = `C

each(eqObj, t1.values(), t2.values()) // true
```

以上两个查询耗时分别为 902 ms、930 ms。 此时，使用逗号与 and 的查询性能相差不大。
测试不同条件先后顺序对于查询性能与查询结果的影响，示例代码如下：

```
timer(10) t3 = select * from t where date = 2019.01.02, sym = `C, qty > 2000
timer(10) t4 = select * from t where date = 2019.01.02 and sym = `C and qty > 2000

each(eqObj, t1.values(), t3.values()) // true
each(eqObj, t2.values(), t4.values()) // true
```

以上两个查询耗时分别为 669 ms、651 ms。 此时，使用逗号与 and 的查询性能相差不大。
说明过滤条件与序列无关时，条件先后顺序对于查询结果无影响。但性能方面 t3(t4) 较 t1(t2) 提升约30%，这是因为 date 字段比 qty 字段筛选性更强。

#### 2.3.2. 过滤条件与序列有关


```
timer(10) t1 = select * from t where ratios(qty) > 1, date = 2019.01.02, sym = `C
timer(10) t2 = select * from t where ratios(qty) > 1 and date = 2019.01.02 and sym = `C

each(eqObj, t1.values(), t2.values()) // true
```

以上两个查询耗时分别为 1503 ms、1465 ms。
此时，使用逗号与 and 的查询性能相差无几。序列条件作为第一个条件，使用逗号连接时，首先按照原表中数据的顺序进行计算，后面条件与序列无关，所以查询结果与 and 连接时保持一致。
测试不同条件先后顺序对于查询性能与查询结果的影响，示例代码如下：

```
timer(10) t3 = select * from t where date = 2019.01.02, sym = `C, ratios(qty) > 1
timer(10) t4 = select * from t where date = 2019.01.02 and sym = `C and ratios(qty) > 1

each(eqObj, t2.values(), t4.values()) // true
each(eqObj, t1.values(), t3.values()) // false
```

以上两个查询耗时分别为 507 ms、1433 ms。 第一个 each 函数返回均为 true，即 t2 与 t4 查询结果相同；第二个 each 函数返回均为 false，即 t1 与 t3 查询结果不同。
说明过滤条件与序列相关时，对于使用 and 连接的查询语句，条件先后顺序对于查询结果无影响，性能方面亦无差别；对于使用逗号的查询语句，序列条件在后，性能虽有提升，但查询结果不同。
综合上述测试结果分析可知：
- 过滤条件与序列无关时，使用逗号或 and 均可，这是因为系统内部对于 and 做了优化，即将 and 转换为逗号，逗号会按照条件先后顺序层层过滤，因此条件先后顺序不同，执行查询时会有所差别，建议尽可能将过滤能力较强的条件放在前面，以减少后面过滤条件需要查询的数据量；
- 过滤条件与序列相关时，必须使用 and，会对所有过滤条件在原表内分别筛选，再将过滤结果取交集，因此条件先后顺序不影响查询结果与性能。

## 3. 分布式表相关案例

分布式查询和普通查询的语法并无差异，理解分布式查询的工作原理有助于编写高效的 SQL 查询语句。系统首先根据 where 条件子句确定查询涉及的分区，然后分解查询语句为多个子查询，并把子查询发送到相关分区所在的位置(map)，最后在发起节点汇总所有分区的查询结果(merge)，并进行进一步的查询(reduce)。

### 3.1. 分区剪枝

场景：查询每只股票在某个时间范围内的记录数目。
首先，载入测试数据库下的表 “Snapshot” 并将该表对象赋值给变量 snapshot，示例如下：

#### 3.1.1. 优化前

where 条件子句根据日期过滤时，使用 temporalFormat 函数对于日期进行格式转换，如下：

```
timer t1 = select count(*) from snapshot 
		   where temporalFormat(DateTime, "yyyy.MM.dd") >= "2020.06.01" and temporalFormat(DateTime, "yyyy.MM.dd") <= "2020.06.02" 
		   group by SecurityID
```

查询耗时 4145 ms。

#### 3.1.2. 优化后

使用 date 函数将 DateTime 字段转换为 DATE 类型，如下：

```
timer t2 = select count(*) from snapshot 
		   where date(DateTime) between 2020.06.01 : 2020.06.02 group by SecurityID
```

查询耗时 92 ms。

```
each(eqObj, t1.values(), t2.values()) // true
```

与优化前写法相比，查询性能提升数十倍。DolphinDB 在解决海量数据的存取时，并不提供行级的索引，而是将分区作为数据库的物理索引。系统在执行分布式查询时，首先根据 where 条件确定需要的分区。大多数分布式查询只涉及分布式表的部分分区，系统无需全表扫描，从而节省大量时间。但若不能根据 where 条件确定分区，进行全表扫描，就会大大降低查询性能。
可以看到以上优化前的脚本，分区字段套用了 temporalFormat 函数先对所有日期进行转换，因此系统无法做分区剪枝。
下面例举了部分其它导致系统 无法做分区剪枝 的案例：
例1：对分区字段进行运算。

```
select count(*) from snapshot where date(DateTime) + 1 > 2020.06.01
```


```
select count(*) from snapshot where 2020.06.01 < date(DateTime) < 2020.06.03
```

例3：过滤条件未使用分区字段。

```
select count(*) from snapshot where Volume < 500
```

例4：与分区字段比较时使用其它列。AnnouncementDate 字段非 snapshot 表中字段，此处仅为举例说明。

```
select count(*) from snapshot where date(DateTime) < AnnouncementDate - 3
```


### 3.2. group by 并行查询

场景：对在某个时间范围内所有股票，标记多空势力方向，并计算第一档行情买卖双方报价之差、总交易量等指标。
首先，载入测试数据库中的表，示例如下：

#### 3.2.1. 优化前

首先，筛选2020年06月01日14:30:00以后的数据，卖方价格高于买方价格的记录，标志位设置为1；否则，标志位设置为0，将结果赋给一个内存表。然后，使用 group by 子句根据 Symbol, DateTime, Flag三个字段分组，并统计分组内 OfrPx 的记录数以及 OfrSize 的和，示例如下：

```
timer {
	tmp_t = select *, iif(LastPx > OpenPx, 1, 0) as Flag 
			from snapshot 
			where date(DateTime) = 2020.06.01, second(DateTime) >= 09:30:00
	t1 = select iif(max(OfferPrice1) - min(BidPrice1) == 0, 0, 1) as Price1Diff, count(OfferPrice1) as OfferPrice1Count, sum(Volume) as Volumes 
			from tmp_t 
			group by SecurityID, date(DateTime) as Date, Flag
}
```

查询耗时 6249 ms。

#### 3.2.2. 优化后

不再引入中间内存表，直接从分布式表进行查询计算。示例如下：

```
timer t2 = select iif(max(OfferPrice1) - min(BidPrice1) == 0, 0, 1) as Price1Diff, count(OfferPrice1) as OfferPrice1Count, sum(Volume) as Volumes 
			from snapshot 
			where date(DateTime) = 2020.06.01, second(DateTime) >= 09:30:00 
			group by SecurityID, date(DateTime) as Date, iif(LastPx > OpenPx, 1, 0) as Flag
```

查询耗时 1112 ms。

```
each(eqObj, t1.values(), (select * from t2 order by SecurityID, Date, Flag).values()) // true
```

与优化前写法相比，优化后写法查询性能提升约 6 倍。
性能的提升来自于两个方面：
（1）优化前的写法先把分区数据合并到一个内存表，然后再用 group by 分组计算，比优化后的写法多了合并与拆分的两个步骤。
（2）优化后的写法直接对分布式表进行分组计算，充分利用 CPU 多核并行计算。而优化前的写法合并成一个内存表后，只利用单核进行分组计算。
作为一个通用规则，对于分布式表的查询和计算，尽可能不要生成中间结果，直接在原始的分布式表上做计算，性能最优。

### 3.3. 分组查询使用 map 关键字

场景：查询每只股票每分钟的记录数目。
首先，载入测试数据库表：

#### 3.3.1. 优化前


```
timer result = select count(*) from snapshot group by SecurityID, bar(DateTime, 60)
```

查询耗时 996 ms。

#### 3.3.2. 优化后

使用 map 关键字。
查询耗时 864 ms。与优化前写法相比，查询性能提升约 10%~20%。
优化前分组查询或计算时分为两个步骤：
- 每个分区内部计算；
- 所有分区的结果进行进一步计算，以确保最终结果的正确。
如果分区的粒度大于分组的粒度，那么第一步骤完全可以保证结果的正确。此场景中，一级分区为粒度为“天”，大于分组的粒度“分钟”，可以使用 map 关键字，避免第二步骤的计算开销，从而提升查询性能。

## 4. 分组计算相关案例


### 4.1. 查询最新的 N 条记录

场景：获取每只股票最新的10条记录。
仅对2020年06月01日的数据进行分组求 TOP 10。context by 子句对数据进行分组，返回结果中每一组的行数和组内元素数量相同，再结合 csort 和 top 关键字，可以获取每组数据的最新记录。以行数为960万行的数据为例：

```
timer t1 = select * from loadTable("dfs://Level1", "Snapshot") where date(DateTime) = 2020.06.01 context by SecurityID csort DateTime limit -10
```

查询耗时 4289 ms。

```
timer t2 = select * from loadTable("dfs://Level1_TSDB", "Snapshot") where date(DateTime) = 2020.06.01 context by SecurityID csort DateTime limit -10
```

查询耗时 1122 ms。

```
each(eqObj, t1.values(), t2.values()) //true
```

TSDB 是 DolphinDB 2.0 版本推出的存储引擎，引入了排序列，相当于对分区内部建立了一个索引。因此对于时间相关、单点查询场景，性能较 OLAP 存储引擎会有进一步提升。
此例中，TSDB 存储引擎的查询性能较 OLAP 存储引擎提升约 4 倍。
context by 是 DolphinDB SQL 独有的创新，是对标准 SQL 语句的拓展。在关系型数据库管理系统中，一张表由行的集合组成，行之间没有顺序。可以使用如 min , max , avg 等聚合函数来对行进行分组，但是不能对分组内的行使用序列相关的聚合函数，比如 first , last 等，或者使用顺序敏感的滑动窗口函数和累积计算函数，如 cumsum , cummax , ratios , deltas 等。
DolphinDB 使用列式存储引擎，因此能更好地支持对时间序列的数据进行处理，而其特有的 context by 子句使组内处理时间序列数据更加方便。

### 4.2. 计算滑动 VWAP

场景：一个内存表包含3000只股票，每只股票10000条记录，使用循环与 context by 两种方法分别计算 mwavg (移动加权平均，Moving Weighted Average)，比较二者性能差异。
首先，产生模拟数据，示例如下：

```
syms = format(1..3000, "SH000000")
N = 10000
t = cj(table(syms as symbol), table(rand(100.0, N) as price, rand(10000, N) as volume))
```

使用循环，每一次取出某只股票相应的10000条记录的价格、交易量字段，计算 mwavg ，共执行3000次，然后合并每一次的计算结果。

```
arr = array(ANY, syms.size())

timer {
	for(i in 0 : syms.size()) {
		price_vec = exec price from t where symbol = syms[i]
		volume_vec = exec volume from t where symbol = syms[i]
		arr[i] = mwavg(price_vec, volume_vec, 4)
	}
	res1 = reduce(join, arr)
}
```

查询耗时 25 min。
使用 context by，根据股票分组，每个分组内部分别计算 mwavg。

```
timer res2 = select mwavg(price, volume, 4) from t 
			   context by symbol
```

查询耗时 3176 ms。

```
each(eqObj, res1, res2[`mwavg_price]) // true
```

两种方法的性能相差约 400 多倍。
原因是，context by 仅对全表数据扫描一次，并对所有股票分组，再对每组分别进行计算；而 for 循环每一次循环都要扫描全表以获取某只股票相应的10000记录，所以耗时较长。

### 4.3. 计算累积 VWAP

场景：每分钟计算每只股票自开盘到现在的所有交易的 vwap (交易量加权平均价格，Volume Weighted Average Price)。
首先，载入测试数据库表：
使用 group by 对股票分组，再对时间做分钟聚合并使用 cgroup by 分组，计算 vwap；然后使用 order by 子句对分组计算结果排序，最后对每只股票分别计算累计值。

```
timer result = select wavg(LastPx, Volume) as vwap 
			   from snapshot 
			   group by SecurityID 
			   cgroup by minute(DateTime) as Minute 
			   order by SecurityID, Minute
```

查询耗时 1499 ms。
cgroup by (cumulative group) 为 DolphinDB SQL 独有的功能，是对标准 SQL 语句的拓展，可以进行累计分组计算，第一次计算使用第一组记录，第二次计算使用前两组记录，第三次计算使用前三组记录，以此类推。
使用 cgroup by 时，必须同时使用 order by 对分组计算结果进行排序。cgroup by 的 SQL 语句仅支持以下聚合函数： sum , sum2 , sum3 , sum4 , prod , max , min , first , last , count , size , avg , std , var , skew , kurtosis , wsum , wavg , corr , covar , contextCount , contextSum , contextSum2 。

### 4.4. 计算 N 股 VWAP

场景：计算每只股票最近 1000 shares 相关的所有 trades 的 vwap。
筛选1000 shares 时可能出现以下情形，如 shares 为100、300、600的3个 trades 之和恰好为1000，或者shares 为900、300两个 trades 之和超过1000。首先需要找到参与计算的 trades，使得 shares 之和恰好超过1000，且保证减掉时间点最新的一个 trade 后，shares 之和小于1000，然后计算一下它们的 vwap。
首先，产生模拟数据，示例如下：

```
n = 500000
t = table(rand(string(1..4000), n) as sym, rand(10.0, n) as price, rand(500, n) as vol)
```

使用 group by 对于股票进行分组，针对每只股票分别调用自定义聚合函数 lastVolPx1，针对所有 trades 采用循环计算，并判断 shares 是否恰好超过 bound，最后计算 vwag。如下：

```
defg lastVolPx1(price, vol, bound) {
	size = price.size()
	cumSum = 0
	for(i in 0:size) {
		cumSum = cumSum + vol[size - 1 - i]
		if(cumSum >= bound) {
			price_tmp = price.subarray(size - 1 - i :)
			vol_tmp = vol.subarray(size - 1 - i :)
			return wavg(price_tmp, vol_tmp)
		}
		if(i == size - 1 && cumSum < bound) {
			return wavg(price, vol)
		}
	}
}

timer lastVolPx_t1 = select lastVolPx1(price, vol, 1000) as lastVolPx from t group by sym
```

查询耗时 187 ms。
使用 group by 对股票进行分组，针对每支股票分别调用自定义聚合函数 lastVolPx2，计算累积交易量向量，以及恰好满足 shares 大于 bound 的起始位置，最后计算 vwag。如下：

```
defg lastVolPx2(price, vol, bound) {
	cumVol = vol.cumsum()
	if(cumVol.tail() <= bound)
		return wavg(price, vol)
	else {
		start = (cumVol <= cumVol.tail() - bound).sum()
		return wavg(price.subarray(start:), vol.subarray(start:))
	}
}

timer lastVolPx_t2 = select lastVolPx2(price, vol, 1000) as lastVolPx from t group by sym
```

查询耗时 73 ms。

```
each(eqObj, lastVolPx_t1.values(), lastVolPx_t2.values()) // true
```

与优化前写法相比，lastVolPx2 使用了向量化编程方法，性能提升一倍多。因此，编写 DolphinDB SQL 时，应当尽可能地使用向量化函数，避免使用循环。

### 4.5. 分段统计股票价格变化率

场景：已知股票市场快照数据，根据其中某个字段，分段统计并计算每只股票价格变化率。
仅对2020年06月01日的数据举例说明。首先，使用 group by 对股票以及 OfferPrice1 字段连续相同的数据分组，然后计算每只股票第一档价格的变化率，示例如下：

```
timer t = select last(OfferPrice1) \ first(OfferPrice1) - 1 
		  from loadTable("dfs://Level1", "Snapshot") 
		  where date(DateTime) = 2020.06.01 
		  group by SecurityID, segment(OfferPrice1, false)
```

查询耗时 511 ms。
segment 函数用于向量分组，将连续相同的元素分为一组，返回与输入向量等长的向量。下一个案例中也使用了 segment 函数分组，以展示该函数在连续区间分组计算时的易用性。

### 4.6. 计算不同连续区间的最值

场景：期望根据某个字段的值，获取大于或等于目标值的连续区间窗口，并在每个窗口内取该字段最大值的第一条记录。
首先，产生模拟数据，示例如下：

```
t = table(2021.09.29 + 0..15 as date, 
          0 0 0.3 0.3 0 0.5 0.3 0.5 0 0 0.3 0 0.4 0.6 0.6 0 as value)
targetVal = 0.3
```

自定义一个函数 generateGrp，如果当前值大于或等于目标值，记录下当前记录对应的分组 ID；如果下一条记录的值小于目标值，分组 ID 加 １，以保证不同的连续数据划分到不同的分组。

```
def generateGrp(targetVal, val) {
	arr = array(INT, val.size())
	n = 1
	for(i in 0 : val.size()) {
		if(val[i] >= targetVal) {
			arr[i] = n
			if(val[i + 1] < targetVal) n = n + 1
		}
	}
	return arr
}
```

使用 context by 根据分组 ID 分组，并结合 having 语句过滤最大值，limit 语句限制返回第一条记录。

```
timer(1000) {
	tmp = select date, value, generateGrp(targetVal, value) as grp from t
	res1 = select date, value from tmp where grp != 0 
		   context by grp 
		   having value = max(value) limit 1
}
```

查询耗时 142 ms。
使用 segment 函数结合 context by 语句对大于或等于目标值的连续数据分组，并使用 having 语句过滤。

```
timer(1000) res2 = select * from t 
				   context by segment(value >= targetVal) 
				   having value >= targetVal and value = max(value) limit 1
```

查询耗时 123 ms。
与优化前写法相比，优化后写法查询性能提升约 10%。
segment 函数一般用于序列相关的分组，与循环相比，性能略有提升，可以化繁为简，使代码更为优雅。

### 4.7. 不同聚合方式计算指标

场景：期望根据不同的标签对于某个字段采用不同的聚合方式。
例如，标签为 code1 时，每10分钟取 max；标签为 code2 时，每10分钟取 min；标签为 code3 时，每10分钟取 avg。最后获得一个行转列宽表。
首先，产生模拟数据，示例如下：

```
N = 1000000
t = table("code" + string(take(1..3, N)) as tag, 
          sort(take([2021.06.28T00:00:00, 2021.06.28T00:10:00, 2021.06.28T00:20:00], N)) as time, 
          take([1.0, 2.0, 9.1, 2.0, 3.0, 9.1, 9.1, 2.0, 3.0], N) as value)
```

构建一个字典，标签为键，函数名称为值。使用 group by 对时间、标签分组，并调用自定义聚合函数，实现对不同标签的 value 进行不同的运算。

```
codes = dict(`code1`code2`code3, [max, min, avg])

defg func(tag, value, codes) : codes[tag.first()](value)
 
timer {
	t_tmp = select func(tag, value, codes) as value from t 
			group by tag, interval(time, 10m, "null") as time
	t_result = select value from t_tmp pivot by time, tag
}
```

查询耗时 76 ms。
上例中使用的 interval 函数只能在 group by 子句中使用，不能单独使用，缺失值的填充方式可以为："prev", "post", "linear", "null", 具体数值和 "none"。

### 4.8. 计算股票收益波动率

场景：已知某只股票过去十年的日收益率，期望按月计算该股票的波动率。
首先，产生模拟数据，示例如下：

```
N = 3653
t = table(2011.11.01..2021.10.31 as date, 
          take(`AAPL, N) as code, 
          rand([0.0573, -0.0231, 0.0765, 0.0174, -0.0025, 0.0267, 0.0304, -0.0143, -0.0256, 0.0412, 0.0810, -0.0159, 0.0058, -0.0107, -0.0090, 0.0209, -0.0053, 0.0317, -0.0117, 0.0123], N) as rate)
```

使用 interval 函数按日期分组，并计算标准差。其中， fill 类型为 "prev"，表示使用前一个值填充缺失值。

```
timer res = select std(rate) from t group by code, interval(date(date), 1, "prev")
```

查询耗时 4.029 ms。

### 4.9. 计算股票组合的价值

场景：进行指数套利交易回测时，计算给定股票组合的价值。
当数据量极大时，一般数据分析系统进行回测时，对系统内存及速度的要求极高。以下案例，展现了使用 DolphinDB SQL 语言可极为简洁地进行此类计算。
为了简化起见，假定某个指数仅由两只股票组成：AAPL 与 FB。模拟数据如下：

```
syms = take(`AAPL, 6) join take(`FB, 5)
time = 2019.02.27T09:45:01.000000000 + [146, 278, 412, 445, 496, 789, 212, 556, 598, 712, 989]
prices = 173.27 173.26 173.24 173.25 173.26 173.27 161.51 161.50 161.49 161.50 161.51
quotes = table(take(syms, 100000) as Symbol, 
               take(time, 100000) as Time, 
               take(prices, 100000) as Price)
weights = dict(`AAPL`FB, 0.6 0.4)
ETF = select Symbol, Time, Price*weights[Symbol] as weightedPrice from quotes
```

首先，需要将原始数据表的3列（时间，股票代码，价格）转换为同等长度但是宽度为指数成分股数量加1的数据表，然后向前补充空值（forward fill NULLs），进而计算每行的指数成分股对指数价格的贡献之和。示例如下：

```
timer {
	colAAPL = array(DOUBLE, ETF.Time.size())
	colFB = array(DOUBLE, ETF.Time.size())
	
	for(i in 0:ETF.Time.size()) {
		if(ETF.Symbol[i] == `AAPL) {
			colAAPL[i] = ETF.weightedPrice[i]
			colFB[i] = NULL
		}
		if(ETF.Symbol[i] == `FB) {
			colAAPL[i] = NULL
			colFB[i] = ETF.weightedPrice[i]
		}
	}
	
	ETF_TMP1 = table(ETF.Time, ETF.Symbol, colAAPL, colFB)
	ETF_TMP2 = select last(colAAPL) as colAAPL, last(colFB) as colFB from ETF_TMP1 group by time, Symbol
	ETF_TMP3 = ETF_TMP2.ffill()
	
	t1 = select Time, rowSum(colAAPL, colFB) as rowSum from ETF_TMP3
}
```

以上代码块耗时 713 ms。
使用 pivot by 子句根据时间、股票代码对于数据表重新排序，将时间作为行，股票代码作为列，然后使用 ffill 函数填充 NULL 元素，使用 avg 函数计算均值，最后 rowSum 函数计算每个时间点的股票价值之和，仅需以下一行代码，即可实现上述所有步骤。示例如下：

```
timer t2 = select rowSum(ffill(last(weightedPrice))) from ETF pivot by Time, Symbol
```

查询耗时 23 ms。
与优化前写法相比，优化后写法查询性能提升约 30 倍。
此例中，仅以两只股票举例说明，当股票数量更多时，使用循环遍历的方式更为繁琐，而且性能极低。
pivot by 是 DolphinDB SQL 独有的功能，是对标准 SQL 语句的拓展，可以将表中两列或多列的内容按照两个维度重新排列，亦可配合数据转换函数使用。不仅编程简洁，而且无需产生中间过程数据表，有效避免了内存不足的问题，极大地提升了计算速度。
以下是与此场景类似的另外一个案例，属于物联网典型场景。
场景：假设一个物联网场景中存在三个测点进行实时数据采集，期望针对每个测点分别计算一分钟均值，再对同一分钟的三个测点均值求和。
首先，产生模拟数据，示例如下：

```
N = 10000
t = table(take(`id1`id2`id3, N) as id, 
          rand(2021.01.01T00:00:00.000 +  100000 * (1..10000), N) as time, 
          rand(10.0, N) as value)
```

使用 bar 函数对时间做一分钟聚合，并使用 pivot by 子句根据分钟、测点对数据表重新排序，将分钟作为行，测点作为列，然后使用 ffill 函数填充 NULL 元素，使用 avg 函数计算均值，然后再使用 rowSum 函数计算每个时间点的测点值之和。最后使用 group by 子句结合 interval 函数对于缺失值进行填充。

```
timePeriod = 2021.01.01T00:00:00.000 : 2021.01.01T01:00:00.000
timer result = select sum(rowSum) as v from (
    		   select rowSum(ffill(avg(value))) from t 
    		   where id in `id1`id2`id3, time between timePeriod 
    		   pivot by bar(time, 60000) as minute, id) 
    		   group by interval(minute, 1m, "prev") as minute
```

查询耗时 12 ms。

### 4.10. 根据成交量切分时间窗口

场景：已知股票市场分钟线数据，期望根据成交量对股票在时间上进行切分，最终得到时间窗口不等的若干条数据，包含累计成交量，以及每个窗口的起止时间。
具体切分规则为：假如期望对某只股票成交量约150万股便进行一次时间切分。切分时，如果当前组加上下一条数据的成交量与150万更接近，则下一条数据加入当前组；否则，从下一条数据开始一个新的组。
首先，产生模拟数据，示例如下：

```
N = 28
t = table(take(`600000.SH, N) as wind_code, 
          take(2015.02.11, N) as date, 
          take(13:03:00..13:30:00, N) as time, 
          take([288656, 234804, 182714, 371986, 265882, 174778, 153657, 201388, 175937, 138388, 169086, 203013, 261230, 398971, 692212, 494300, 581400, 348160, 250354, 220064, 218116, 458865, 673619, 477386, 454563, 622870, 458177, 880992], N) as volume)
```

根据切分规则，自定义一个累计函数 caclCumVol，如果当前组需要包含下一条数据的成交量，返回新的累计成交量；否则，返回下一条数据的成交量，即开始一个新的组。

```
def caclCumVol(target, cumVol, nextVol) {
	newVal = cumVol + nextVol
	if(newVal < target) return newVal
	else if(newVal - target > target - cumVol) return nextVol
	else return newVal
}
```

使用高阶函数 accumulate ，迭代地应用 caclCumVol 函数到前一个累计成交量和下一个成交量上。如果累计成交量等于当前一条数据的成交量，则表示开始一个新的组，此时记录下当前这条数据的时间，作为一个窗口的起始时间，否则为空，通过 ffill 填充，使得同一组数据拥有相同的起始时间，最后根据起始时间分组并做聚合计算。

```
timer result = select first(wind_code) as wind_code, first(date) as date, sum(volume) as sum_volume, last(time) as endTime 
			   from t 
			   group by iif(accumulate(caclCumVol{1500000}, volume) == volume, time, NULL).ffill() as startTime
```

查询耗时 0.9 ms。

### 4.11. 股票因子归整

场景：已知沪深两市某个10分钟因子，分别存储为一张分布式表，另有一张股票清单维度表存储股票代码相关信息。期望从沪市、深市分别取出部分股票代码相应因子，根据股票、日期对于因子做分组归整，并做行列转换。
首先，自定义一个函数 createDBAndTable，用于创建分布式库表，如下：

```
def createDBAndTable(dbName, tableName) {
    if(existsTable(dbName, tableName)) return loadTable(dbName, tableName)
    dbDate = database(, VALUE, 2021.07.01..2021.07.31)
    dbSecurityID = database(, HASH, [SYMBOL, 10])
    db = database(dbName, COMPO, [dbDate, dbSecurityID])
    model = table(1:0, `SecurityID`Date`Time`FactorID`FactorValue, [SYMBOL, DATE, TIME, SYMBOL, DOUBLE])
    return createPartitionedTable(db, model, tableName, `Date`SecurityID)
}
```

执行以下代码，创建两个分布式表、一个维度表，并写入模拟数据，如下：

```
dates = 2020.01.01..2021.10.31
time = join(09:30:00 + 1..12 * 60 * 10, 13:00:00 + 1..12 * 60 * 10)

syms = format(1..2000, "000000") + ".SH"
tmp = cj(cj(table(dates), table(time)), table(syms))
t = table(tmp.syms as SecurityID, tmp.dates as Date, tmp.time as Time, take(["Factor01"], tmp.size()) as FactorID, rand(100.0, tmp.size()) as FactorValue)

createDBAndTable("dfs://Factor10MinSH", "Factor10MinSH").append!(t)

syms = format(2001..4000, "000000") + ".SZ"
tmp = cj(cj(table(dates), table(time)), table(syms))
t = table(tmp.syms as SecurityID, tmp.dates as Date, tmp.time as Time, take(["Factor01"], tmp.size()) as FactorID, rand(100.0, tmp.size()) as FactorValue)
createDBAndTable("dfs://Factor10MinSZ", "Factor10MinSZ").append!(t)

db = database("dfs://infodb", VALUE, 1 2 3)
model = table(1:0, `SecurityID`Info, [SYMBOL, STRING])
if(!existsTable("dfs://infodb", "MdSecurity")) createTable(db, model, "MdSecurity")
loadTable("dfs://infodb", "MdSecurity").append!(
    table(join(format(1..2000, "000000") + ".SH", format(2001..4000, "000000") + ".SZ") as SecurityID, 
          take(string(NULL), 4000) as Info))

setMaxMemSize(32)
```

首先，分别从沪市、深市取出因子 Factor01 在某个时间范围的数据，合并后，再从股票代码维度表中取出需要归整的股票，通过表连接方式对合并结果进行过滤，最后使用 pivot by 子句根据时间、股票代码两个维度重新排列。

```
timer {
    nt1 = select concatDateTime(Date, Time) as TradeTime, SecurityID, FactorValue from loadTable("dfs://Factor10MinSH", "Factor10MinSH") where Date between 2019.01.01 : 2020.10.31, FactorID = "Factor01"
    nt2 = select concatDateTime(Date, Time) as TradeTime, SecurityID, FactorValue from loadTable("dfs://Factor10MinSZ", "Factor10MinSZ") where Date between 2019.01.01 : 2020.10.31, FactorID = "Factor01"
    unt = unionAll(nt1, nt2)
    
    sec = select SecurityID from loadTable("dfs://infodb", "MdSecurity") where substr(SecurityID, 0, 3) in ["001", "003", "005", "007"]
    res = select * from lj(sec, unt, `SecurityID)

    res1 = select FactorValue from res pivot by TradeTime, SecurityID
}
```

查询耗时 6922 ms。
首先，从股票代码维度表中取出需要归整的股票列表，然后从沪深两市取出因子 Factor01。使用 in 关键字进行过滤，再使用 pivot by 根据时间、股票代码两个维度进行重新排列，最后合并结果。

```
timer {
    sec = exec SecurityID from loadTable("dfs://infodb", "MdSecurity") where substr(SecurityID, 0, 3) in ["001", "003", "005", "007"]
    schema(loadTable("dfs://Factor10MinSH", "Factor10MinSH"))
    nt1 = exec FactorValue from loadTable("dfs://Factor10MinSH", "Factor10MinSH") where Date between 2019.01.01 : 2020.10.31, SecurityID in sec, FactorID = "Factor01" pivot by concatDateTime(Date, Time), SecurityID
    nt2 = exec FactorValue from loadTable("dfs://Factor10MinSZ", "Factor10MinSZ") where Date between 2019.01.01 : 2020.10.31, SecurityID in sec, FactorID = "Factor01" pivot by concatDateTime(Date, Time), SecurityID

    res3 = merge(nt1.setIndexedMatrix!(), nt2.setIndexedMatrix!(), 'left')
}
```

查询耗时 4210 ms。
与优化前相比，优化后查询性能提升约 40%。
综合对比上述写法，概括出几个 SQL 编写技巧：
（1）尽量避免不必要的表连接；
（2）尽可能早地使用分区过滤；
（3）推迟数据的合并。

### 4.12. 根据交易额统计单子类型

场景：对不同日期、不同股票、买单卖单，分别统计某个时间范围内的特大单、大单、中单、小单的累计交易量、交易额。
具体规则为：交易额小于4万是小单，大于等于4万且小于20万是中单，大于等于20万且小于100万是大单，大于100万是特大单。
首先，产生模拟数据，示例如下：

```
N = 1000000
t = table(take(2021.11.01..2021.11.15, N) as date, 
          take([09:30:00, 09:35:00, 09:40:00, 09:45:00, 09:47:00, 09:49:00, 09:50:00, 09:55:00, 09:56:00, 10:00:00], N) as time, 
          take(`AAPL`FB`MSFT$SYMBOL, N) as symbol, 
          take([10000, 30000, 50000, 80000, 100000], N) as volume, 
          rand(100.0, N) as price, 
          take(`BUY`SELL$SYMBOL, N) as side)
```

使用 group by 根据日期、股票、买卖方向分组，使用四个查询语句分别计算小单、中单、大单、特大单的累计交易量、交易额，再将结果合并。

```
timer {
	// 小单
	resS = select sum(volume) as volume_sum, sum(volume * price) as amount_sum, 0 as type 
			from t 
			where time <= 10:00:00, volume * price < 40000 
			group by date, symbol, side
	// 中单
	resM = select sum(volume) as volume_sum, sum(volume * price) as amount_sum, 1 as type 
			from t 
			where time <= 10:00:00, 40000 <= volume * price < 200000 
			group by date, symbol, side
	// 大单
	resB = select sum(volume) as volume_sum, sum(volume * price) as amount_sum, 2 as type 
			from t 
			where time <= 10:00:00, 200000 <= volume * price < 1000000 
			group by date, symbol, side
	// 特大单
	resX = select sum(volume) as volume_sum, sum(volume * price) as amount_sum, 3 as type 
			from t 
			where time <= 10:00:00, volume * price >= 1000000 
			group by date, symbol, side
	
	res1 = table(N:0, `date`symbol`side`volume_sum`amount_sum`type, [DATE, SYMBOL, SYMBOL, LONG, DOUBLE, INT])
	res1.append!(resS).append!(resM).append!(resB).append!(resX)
}
```

查询耗时 135 ms。
自定义一个函数 getType，使用 iif 函数嵌套方式得到当前成交单子类型，然后使用 group by 对日期、股票、买卖方向、单子类型分组，并计算累计交易量、交易额。

```
def getType(amount) {
	return iif(amount < 40000, 0, iif(amount >= 40000 && amount < 200000, 1, iif(amount >= 200000 && amount < 1000000, 2, 3)))
}

timer res2 = select sum(volume) as volume_sum, sum(volume*price) as amount_sum 
				from t 
				where time <= 10:00:00
				group by date, symbol, side, getType(volume * price) as type
```

查询耗时 114 ms。
与优化前写法相比，优化后写法查询性能提升约 20%。 虽然性能略有提升，但大大简化了代码编写。
需要注意的是，此处有个优化技巧，group by 后面字段类型为 INT、LONG、SHORT、SYMBOL 时，系统内部进行了优化，查询性能会有一定提升，所以本例中 getType 函数返回类型为 INT。

```
range = [0.0, 40000.0, 200000.0, 1000000.0, 100000000.0]

timer res3 = select sum(volume) as volume_sum, sum(volume*price) as amount_sum 
				from t 
				where time <= 10:00:00 
				group by date, symbol, side, asof(range, volume*price) as type
```

查询耗时 95 ms。
与第一种优化写法区别在于，使用 asof 函数而非自定义函数，判断交易额落在哪个区间，然后以此分组并计算累计交易量、交易额。

```
each(eqObj, (select date, symbol, side, type, volume_sum, amount_sum 
             from res1 order by date, symbol, side, type).values(), res2.values()) // true
each(eqObj, res2.values(), res3.values()) // true
```

以下是 asof 函数在另外一个场景下的应用。
场景：针对某个日期、某只股票，统计某一数值列落在不同的区间范围的记录数目。
首先，产生模拟数据，示例如下：

```
N = 100000
t = table(take(2021.11.01, N) as date, 
          take(`AAPL, N) as code, 
          rand([-5, 5, 10, 15, 20, 25, 100], N) as value)
range = [-9999, 0, 10, 30, 9999]
```

自定义一个函数 generateGrp，遍历不同的区间范围，判断数值列是否包含在当前的区间范围内，区间范围遵循左闭右开原则，并返回一个布尔型向量。如果为 true，表示数值包含在当前的区间范围内，则以下划线连接区间范围的左右边界作为分组 ID；如果为 false，表示数值不包含在当前的区间范围内，则将其置为空字符串。
然后使用高阶函数 reduce 将遍历结果合并，最后使用 group by 根据日期、股票、不同的区间范围分组，并聚合计算记录数目。

```
def generateGrp(range, val) {
	res = array(ANY, range.size()-1)
	for(i in 0 : (range.size()-1)) {
		cond = val >= range[i] && val < range[i+1]
		res[i] = iif(cond, strReplace(range[i] + "_" + range[i+1], "-", ""), string(NULL))
	}
	return reduce(add, res)
}

timer res1 = select count(*) from t group by date, code, generateGrp(range, value) as grp
```

查询耗时 38 ms。
使用 asof 函数结合 group by 语句对于日期、股票、不同的区间范围分组，并聚合计算记录数目。

```
timer res2 = select count(*) from t 
			group by date, code, asof(range, value) as grp
```

查询耗时 14 ms。
与优化前写法相比，优化后写法查询性能提升约 2 倍多。
asof 函数一般用于分段统计，与循环相比，不仅性能大大提升，而且代码更为简洁。下一个案例也是使用了 asof 函数用于统计。

## 5. 元编程相关案例


### 5.1. 动态生成 SQL 语句案例 1

场景：已知股票市场分钟线数据，使用元编程方式，根据股票、日期分组，每隔 10 分钟做一次聚合计算。
首先，产生模拟数据，示例如下：

```
N = 10000000
t = table(take(format(1..4000, "000000") + ".SH", N) as SecurityID, 
          take(2021.10.01..2021.10.31, N) as TradeDate, 
          take(join(09:30:00 + 1..120 * 60, 13:00:00 + 1..120 * 60), N) as TradeTime, 
          rand(100.0, N) as Px)

barMinutes = 10
```

查询语句拼接为一个字符串，使用 parseExpr 函数将字符串解析为元代码，再使用 eval 函数执行生成的元代码。

```
res = parseExpr("select " + avg + "(Px) as avgPx from t group by bar(TradeTime, " + barMinutes + "m) as minuteTradeTime, SecurityID, TradeDate").eval()
```

查询耗时 219 ms。
DolphinDB 内置了 sql 函数用于动态生成 SQL 语句，然后使用 eval 函数执行生成的 SQL 语句。其中， sqlCol 函数将列名转化为表达式， makeCall 函数指定参数调用 bar 函数并生成脚本， sqlColAlias 函数使用元代码和别名定义一个列。

```
groupingCols = [sqlColAlias(makeCall(bar, sqlCol("TradeTime"), duration(barMinutes.string() + "m")), "minuteTradeTime"), sqlCol("SecurityID"), sqlCol("TradeDate")]
res = sql(select = sqlCol("Px", funcByName("avg"), "avgPx"), 
          from = t, groupBy = groupingCols, groupFlag = GROUPBY).eval()
```

查询耗时 200 ms。
类似地， sqlUpdate 函数用于动态生成 SQL update 语句的元代码， sqlDelete 函数用于动态生成 SQL delete 语句的元代码。

### 5.2. 动态生成 SQL 语句案例 2

场景：每天需要执行一组查询，合并查询结果。
首先，产生模拟数据，示例如下：

```
N = 100000
t = table(take(50982208 51180116 41774759, N) as vn, 
          rand(25 1180 50, N) as bc, 
          take(814 333 666, N) as cc, 
          take(11 12 3, N) as stt, 
          take(2 116 14, N) as vt, 
          take(2020.02.05..2020.02.05, N) as dsl, 
          take(52354079..52354979, N) as mt)
```

例如，每天需要执行一组查询，如下：

```
t1 = select * from t where vn=50982208, bc=25, cc=814, stt=11, vt=2, dsl=2020.02.05, mt < 52355979 order by mt desc limit 1
t2 = select * from t where vn=50982208, bc=25, cc=814, stt=12, vt=2, dsl=2020.02.05, mt < 52355979 order by mt desc limit 1
t3 = select * from t where vn=51180116, bc=25, cc=814, stt=12, vt=2, dsl=2020.02.05, mt < 52354979 order by mt desc limit 1
t4 = select * from t where vn=41774759, bc=1180, cc=333, stt=3, vt=116, dsl=2020.02.05, mt < 52355979 order by mt desc limit 1

reduce(unionAll, [t1, t2, t3, t4])
```

以下案例通过元编程动态生成 SQL 语句实现。过滤条件包含的列和排序的列相同，可编写如下自定义函数 bundleQuery 实现相关操作：

```
def bundleQuery(tbl, dt, dtColName, mt, mtColName, filterColValues, filterColNames){
	cnt = filterColValues[0].size()
	filterColCnt = filterColValues.size()
	orderByCol = sqlCol(mtColName)
	selCol = sqlCol("*")
	filters = array(ANY, filterColCnt + 2)
	filters[filterColCnt] = expr(sqlCol(dtColName), ==, dt)
	filters[filterColCnt+1] = expr(sqlCol(mtColName), <, mt)
	
	queries = array(ANY, cnt)
	for(i in 0:cnt)	{
		for(j in 0:filterColCnt){
			filters[j] = expr(sqlCol(filterColNames[j]), ==, filterColValues[j][i])
		}
		queries.append!(sql(select=selCol, from=tbl, where=filters, orderBy=orderByCol, ascOrder=false, limit=1))
	}
	return loop(eval, queries).unionAll(false)
}
```

bundleQuery 中各个参数的含义如下：
- tbl 是数据表。
- dt 是过滤条件中日期的值。
- dtColName 是过滤条件中日期列的名称。
- mt 是过滤条件中 mt 的值。
- mtColName 是过滤条件中 mt 列的名称，以及排序列的名称。
- filterColValues 是其他过滤条件中的值，用元组表示，其中的每个向量表示一个过滤条件，每个向量中的元素表示该过滤条件的值。
- filterColNames 是其他过滤条件中的列名，用向量表示。
上面一组 SQL 语句，相当于执行以下代码：

```
dt = 2020.02.05
dtColName = "dsl"
mt = 52355979
mtColName = "mt"
colNames = `vn`bc`cc`stt`vt
colValues = [50982208 50982208 51180116 41774759, 25 25 25 1180, 814 814 814 333, 11 12 12 3, 2 2 2 116]

bundleQuery(t, dt, dtColName, mt, mtColName, colValues, colNames)
```

登录 admin 管理员用户后，执行以下脚本将 bundleQuery 函数定义为函数视图，以确保在集群的任何节点重启系统之后，都可直接调用该函数。

```
addFunctionView(bundleQuery)
```

如果每次都手动编写全部 SQL 语句，工作量大，并且扩展性差，通过元编程动态生成 SQL 语句可以解决这个问题。


---

> 来源: `func_progr_cases.html`


# 函数化编程案例

DolphinDB 支持函数化编程：函数对象可以作为高阶函数的参数。这提高了代码表达能力，简化了代码，复杂的任务可以通过一行或几行代码完成。
本教程介绍了一些常见场景下的函数化编程案例，重点介绍 DolphinDB 的高阶函数及其使用场景。

## 1. 数据导入


### 1.1. 整型时间转化为 TIME 格式并导入

CSV 数据文件中常用整数表示时间，如“93100000”表示“9:31:00.000”。为了便于查询分析，建议将这类数据转换为时间类型，再存储到 DolphinDB 数据库中。
针对这种场景，可通过 loadTextEx 函数的 transform 参数将文本文件中待转化的时间列指定为相应的数据类型。
本例中会用到 CSV 文件 candle_201801.csv ，数据样本如下：

```
symbol,exchange,cycle,tradingDay,date,time,open,high,low,close,volume,turnover,unixTime
000001,SZSE,1,20180102,20180102,93100000,13.35,13.39,13.35,13.38,2003635,26785576.72,1514856660000
000001,SZSE,1,20180102,20180102,93200000,13.37,13.38,13.33,13.33,867181
......
```

- 建库 用脚本创建如下分布式数据库（按天进行值分区）： login(`admin,`123456) dataFilePath="/home/data/candle_201801.csv" dbPath="dfs://DolphinDBdatabase" db=database(dbPath,VALUE,2018.01.02..2018.01.30)
- 建表 下面先通过 extractTextSchema 函数获取数据文件的表结构。csv 文件中的 time 字段被识别为整型。若要将其存为 TIME 类型，可以通过 update 语句更新表结构将其转换为 TIME 类型，然后用更新后的表结构来创建分布式表。该分布式表的分区列是 date 列。 schemaTB=extractTextSchema(dataFilePath) update schemaTB set type="TIME" where name="time" tb=table(1:0, schemaTB.name, schemaTB.type) tb=db.createPartitionedTable(tb, `tb1, `date); 这里通过 extractTextSchema 获取表结构。用户也可以自定义表结构。
- 导入数据 可以通过自定义函数 i2t 对时间列 time 进行预处理，将其转换为 TIME 类型，并返回处理后的数据表。 def i2t(mutable t){ return t.replaceColumn!(`time, t.time.format("000000000").temporalParse("HHmmssSSS")) } 请注意：在自定义函数体内对数据进行处理时，请尽量使用本地的修改（以 ! 结尾的函数）来提升性能。 调用 loadTextEx 函数导入 csv 文件的数据到分布式表，这里指定 transform 参数为 i2t 函数，导入时会自动应用 i2t 函数处理数据。 tmpTB=loadTextEx(dbHandle=db, tableName=`tb1, partitionColumns=`date, filename=dataFilePath, transform=i2t);
- 查询数据 查看表内前 2 行数据，可以看到结果符合预期。 select top 2 * from loadTable(dbPath,`tb1); symbol exchange cycle tradingDay date time open high low close volume turnover unixTime ------ -------- ----- ---------- ---------- -------------- ----- ----- ----- ----- ------- ---------- ------------- 000001 SZSE 1 2018.01.02 2018.01.02 09:31:00.000 13.35 13.39 13.35 13.38 2003635 2.678558E7 1514856660000 000001 SZSE 1 2018.01.02 2018.01.02 09:32:00.000 13.37 13.38 13.33 13.33 867181 1.158757E7 1514856720000

```
login(`admin,`123456)
dataFilePath="/home/data/candle_201801.csv"
dbPath="dfs://DolphinDBdatabase"
db=database(dbPath,VALUE,2018.01.02..2018.01.30)
schemaTB=extractTextSchema(dataFilePath)
update schemaTB set type="TIME" where name="time"
tb=table(1:0,schemaTB.name,schemaTB.type)
tb=db.createPartitionedTable(tb,`tb1,`date);

def i2t(mutable t){
    return t.replaceColumn!(`time,t.time.format("000000000").temporalParse("HHmmssSSS"))
}

tmpTB=loadTextEx(dbHandle=db,tableName=`tb1,partitionColumns=`date,filename=dataFilePath,transform=i2t);
```

注 ：关于文本导入的相关函数和案例，可以参考 数据导入教程

### 1.2. 有纳秒时间戳的文本导入

本例将以整数类型存储的纳秒级数据导入为 NANOTIMESTAMP 类型。本例使用文本文件 nx.txt ，数据样本如下：

```
SendingTimeInNano#securityID#origSendingTimeInNano#bidSize
1579510735948574000#27522#1575277200049000000#1
1579510735948606000#27522#1575277200049000000#2
...
```

每一行记录通过字符'#'来分隔列，SendingTimeInNano 和 origSendingTimeInNano 用于存储纳秒时间戳。
- 导入数据 导入数据时，使用函数 nanotimestamp ，将文本中的整型转化为 NANOTIMESTAMP 类型： def dataTransform(mutable t){ return t.replaceColumn!(`SendingTimeInNano, nanotimestamp(t.SendingTimeInNano)).replaceColumn!(`origSendingTimeInNano, nanotimestamp(t.origSendingTimeInNano)) } 最终通过 loadTextEx 导入数据。

```
dbSendingTimeInNano = database(, VALUE, 2020.01.20..2020.02.22);
dbSecurityIDRange = database(, RANGE,  0..10001);
db = database("dfs://testdb", COMPO, [dbSendingTimeInNano, dbSecurityIDRange]);

nameCol = `SendingTimeInNano`securityID`origSendingTimeInNano`bidSize;
typeCol = [`NANOTIMESTAMP,`INT,`NANOTIMESTAMP,`INT];
schemaTb = table(1:0,nameCol,typeCol);

db = database("dfs://testdb");
nx = db.createPartitionedTable(schemaTb, `nx, `SendingTimeInNano`securityID);

def dataTransform(mutable t){
  return t.replaceColumn!(`SendingTimeInNano, nanotimestamp(t.SendingTimeInNano)).replaceColumn!(`origSendingTimeInNano, nanotimestamp(t.origSendingTimeInNano))
}

pt=loadTextEx(dbHandle=db,tableName=`nx , partitionColumns=`SendingTimeInNano`securityID,filename="nx.txt",delimiter='#',transform=dataTransform);
```


## 2. Lambda 表达式

在 DolphinDB 中可以使用命名函数或匿名函数（通常为 lambda 表达式）来创建自定义函数。例如：

```
x = 1..10
each(x -> x + 1, [1, 2, 3])
```

在这个例子中使用了一个 lambda 表达式（ x -> x + 1, [1, 2, 3] ）作为高阶函数 each 的参数，其中，该 lambda 表达式接受一个输入 x 并返回 x + 1。它与 each 函数一起使用的结果是，将 1 添加到数组 [1, 2, 3] 中的每个元素。
后续章节将介绍其他 lambda 函数案例。

## 3. 高阶函数使用案例


### 3.1. cross 使用案例


#### 3.1.1. 将两个向量或矩阵，两两组合作为参数来调用函数

cross 函数的伪代码如下：

```
for(i:0~(size(X)-1)){
   for(j:0~(size(Y)-1)){
       result[i,j]=<function>(X[i], Y[j]);
   }
}
return result;
```

以计算协方差矩阵为例，一般需要使用两个 for 循环计算。代码如下：

```
def matlab_cov(mutable matt){
 nullFill!(matt,0.0)
 rowss,colss=matt.shape()
 msize = min(rowss, colss)
 df=matrix(float,msize,msize)
 for (r in 0..(msize-1)){
  for (c in 0..(msize-1)){
   df[r,c]=covar(matt[:,r],matt[:,c])
  }
 }
 return df
}
```

以上代码虽然逻辑简单，但是冗长，表达能力较差，且易出错。
在 DolphinDB 中可以使用高阶函数 cross 或 pcross 计算协方差矩阵：

```
cross(covar, matt)
```


#### 3.1.2. 计算股票两两之间的相关性

本例中，我们使用金融大数据开放社区 Tushare 的沪深股票 日线行情 数据，来计算股票间的两两相关性。
首先我们定义一个数据库和表，来存储沪深股票日线行情数据。相关语句如下：

```
login("admin","123456")
dbPath="dfs://tushare"
yearRange=date(2008.01M + 12*0..22)
if(existsDatabase(dbPath)){
 dropDatabase(dbPath)
}
columns1=`ts_code`trade_date`open`high`low`close`pre_close`change`pct_chg`vol`amount
type1=`SYMBOL`NANOTIMESTAMP`DOUBLE`DOUBLE`DOUBLE`DOUBLE`DOUBLE`DOUBLE`DOUBLE`DOUBLE`DOUBLE
db=database(dbPath,RANGE,yearRange)
hushen_daily_line=db.createPartitionedTable(table(100000000:0,columns1,type1),`hushen_daily_line,`trade_date)
```

注：上面的表是按照 日线行情 里的结构说明定义的。
定义好表结构后，如需获取对应的数据，可前往 Tushare 平台注册账户，获取 TOKEN，并参考 案例脚本 进行数据导入操作。本案例使用 DolphinDB 的 Python API 获取数据，用户也可参考 Tushare 的说明文档使用其它语言或库。本例使用 2008 年到 2017 年的日线行情进行说明。
在计算两两相关性时，首先使用 exec + pivot by 生成股票回报率矩阵：

```
retMatrix=exec pct_chg/100 as ret from daily_line pivot by trade_date, ts_code
```

exec 和 pivot by 是 DolphinDB 编程语言的特点之一。 exec 与 select 的用法相同，但 select 语句仅可生成表，, exec 语句可以生成向量。 pivot by 用于重整维度，与 exec 一起使用时会生成一个矩阵。
调用高阶函数 cross 生成股票两两相关性矩阵：

```
corrMatrix=cross(corr,retMatrix)
```

查询和每只股票相关性最高的 10 只股票：

```
syms=(exec count(*) from daily_line group by ts_code).ts_code
syms="C"+strReplace(syms, ".", "_")
mostCorrelated=select * from table(corrMatrix.columnNames() as ts_code, corrMatrix).rename!([`ts_code].append!(syms)).unpivot(`ts_code, syms).rename!(`ts_code`corr_ts_code`corr) context by ts_code having rank(corr,false) between 1:10
```

上面代码中，corrMatrix 是一个矩阵，需要转化为表做进一步处理，同时新增一列表示股票代码。使用 table 函数转化成表后，通过 rename! 函数去修改表的列名。由于表的列名不能以数字开头，故此例中，在 syms 前拼接了字符 "C"，并将 syms 中的字符'.'转化成'_'。
之后，对表做 unpivot 操作，把多列的数据转化成一列。
为了说明中间过程，我们将以上代码拆解出一个中间步骤：

```
select * from table(corrMatrix.columnNames() as ts_code, corrMatrix).rename!([`ts_code].append!(syms)).unpivot(`ts_code, syms)
```


```
ts_code   valueType  value
--------- ---------- -----------------
000001.SZ C600539_SH 1
000002.SZ C600539_SH 0.581235290880416
000004.SZ C600539_SH 0.277978963095669
000005.SZ C600539_SH 0.352580116619933
000006.SZ C600539_SH 0.5056164472398
......
```

这样就得到了每只股票与其它股票的相关系数。之后又使用 rename! 来修改列名，然后通过 context by 来按照 ts_code （股票代码）分组计算。每组中，查询相关性最高的 10 只股票。

```
login("admin","123456")
daily_line= loadTable("dfs://tushare","hushen_daily_line")

retMatrix=exec pct_chg/100 as ret from daily_line pivot by trade_date,ts_code
corrMatrix=cross(corr,retMatrix)

syms=(exec count(*) from daily_line group by ts_code).ts_code
syms="C"+strReplace(syms, ".", "_")
mostCorrelated=select * from table(corrMatrix.columnNames() as ts_code, corrMatrix).rename!([`ts_code].append!(syms)).unpivot(`ts_code, syms).rename!(`ts_code`corr_ts_code`corr) context by ts_code having rank(corr,false) between 1:10
```


### 3.2. each 使用案例

某些场景需要把函数应用到指定参数中的每个元素。若不使用函数化编程，需要使用 for 循环。DolphinDB 提供的高阶函数，例如 each , peach , loop , ploop 等，可以简化代码。

#### 3.2.1. 获取数据表各个列的 NULL 值个数

计算表 t 各列的 NULL 值个数，可以使用高阶函数 each 。

```
each(x->x.size() - x.count(), t.values())
```

注：在 DolphinDB 中，对于向量或矩阵，size 返回所有元素的个数，而 count 返回的是非 NULL 元素的个数。因此可以通过 size 和 count 的差值获得 NULL 元素的个数。
其中，t.values() 返回一个 tuple，每个元素为表 t 其中的一列。

#### 3.2.2. 去除表中存在 NULL 值的行

先通过如下代码生成表 t：

```
sym = take(`a`b`c, 110)
id = 1..100 join take(int(),10)
id2 =  take(int(),10) join 1..100
t = table(sym, id,id2)
```

可用以下两种方法实现。
第一种是直接按行处理，检查每一行是否存在 NULL 值，若存在就去除该行。解决方案如下：

```
t[each(x -> !(x.id == NULL || x.id2 == NULL), t)]
```

需要注意的是，按行处理表时，表的每一行是一个字典对象。这里定义了一个 lambda 表达式来检查空值。
若列数较多，不便枚举时，可以采用以下写法：

```
t[each(x -> all(isValid(x.values())), t)]
```

上面代码中， x.values 获取了该字典所有的值，然后通过 isValid 检查 NULL 值，最后通过 all 将结果汇总，判断该行是否包含 NULL 值。
当数据量较大时，上述脚本运行效率较低。
DolphinDB 采用列式存储，列操作较行操作具有更佳的性能。我们可以调用高阶函数 each 对表的每一列分别应用 isValid 函数，返回一个结果矩阵。通过 rowAnd 判断矩阵的每一行是否存在 0 值。

```
t[each(isValid, t.values()).rowAnd()]
```

当数据量很大时，可能会产生如下报错：

```
The number of cells in a matrix can't exceed 2 billions.
```

这是因为 each(isValid, t.values()) 生成的矩阵过大。为解决该问题，可以调用 reduce 进行迭代计算，遍历检查每一列是否存在 NULL 值。

```
t[reduce(def(x,y) -> x and isValid(y), t.values(), true)]
```


#### 3.2.3. 按行处理与按列处理性能比较案例

下例对表的某个字段进行如下处理："aaaa_bbbb" 替换为 "bbbb_aaaa"。

```
t=table(take("aaaa_bbbb", 1000000) as str);
```

有两种处理思路，可以按行处理或按列处理。
可以调用高阶函数 each 遍历每一行数据，切分后拼接。

```
each(x -> split(x, '_').reverse().concat('_'), t[`str])
```


```
pos = strpos(t[`str], "_")
substr(t[`str], pos+1)+"_"+t[`str].left(pos)
```

对比两种方式的性能，可以看到使用高阶函数 each 按行遍历的时间在 2s300ms 左右，而按列处理的时间在 100ms 左右。因此按列处理性能更高。
完整代码和测试结果如下：

```
t=table(take("aaaa_bbbb", 1000000) as str);

timer r = each(x -> split(x, '_').reverse().concat('_'), t[`str])

timer {
 pos = strpos(t[`str], "_")
 r = substr(t[`str], pos+1)+"_"+t[`str].left(pos)
}
```


#### 3.2.4. 判断两张表内容是否相同

判断两张表 t1 和 t2 的数据是否完全相同，可以使用 each 高阶函数，对表的每列进行比较。

```
all(each(eqObj, t1.values(), t2.values()))
```


### 3.3. loop 使用案例


#### 3.3.1. loop 与 each 的区别

高阶函数 loop 与 each 相似，区别在于函数返回值的格式和类型。。
each 高阶函数根据每个子任务计算结果的数据类型和形式，决定返回值的数据形式：
- 若单个任务返回一个 scalar，则 each 返回一个 vector；
- 若单个任务返回 vector，那么 each 返回一个 matrix；
- 若单个任务返回字典，那么 each 返回一个 table。
若所有子任务的数据类型和形式都相同，则返回 Vector 或 Matrix，否则返回 Tuple。例如：

```
m=1..12$4:3;
m;
each(add{1 2 3 4}, m);
```

| col1 | col2 | col3 |
| --- | --- | --- |
| 2 | 6 | 10 |
| 4 | 8 | 12 |
| 6 | 10 | 14 |
| 8 | 12 | 16 |
而 loop 总是返回 Tuple。例如，使用 loop 计算每一列的最大值：

```
t = table(1 2 3 as id, 4 5 6 as value, `IBM`MSFT`GOOG as name);
t;
loop(max, t.values());
```

| offset | 0 | 1 | 2 |
| --- | --- | --- | --- |
| 0 | 3 | 6 | MSFT |

#### 3.3.2. 导入多个文件

假设在一个目录下，有多个结构相同的 csv 文件，需将其导入到同一个 DolphinDB 内存表中。可以调用高阶函数 loop 来实现：

```
loop(loadText, fileDir + "/" + files(fileDir).filename).unionAll(false)
```


### 3.4. moving/rolling 使用案例


#### 3.4.1. moving 案例

以当前记录的 UpAvgPrice 和 DownAvgPrice 字段值确定一个区间，取 close 字段的前 20 个数计算其是否在区间 [DownAvgPrice, UpAvgPrice] 范围内，并统计范围内数据的百分比。
以 trade_date 为 2019.06.17 的记录的 UpAvgPrice 和 DownAvgPrice 字段确定一个区间 [11.5886533, 12.8061868]，检查该记录的前 20 行 close 数据（即图中标 1 这列）是否在对应区间中，若其中有 75% 的数据落在区间内，则 signal（图中标 4）的值设为 true，否则为 false。
使用高阶函数 moving 。下例编写自定义函数 rangeTest 对每个窗口的数据进行上述区间判断，返回 true 或 false。

```
defg rangeTest(close, downlimit, uplimit){
  size = close.size() - 1
  return between(close.subarray(0:size), downlimit.last():uplimit.last()).sum() >= size*0.75
}

update t set signal = moving(rangeTest, [close, downAvgPrice, upAvgPrice], 21)
```

注：本例中，因为是计算前 20 行作为当期行的列数值，因而窗口需要包含前 20 条记录和本条记录，故窗口大小为 21 行。
注：上例调用 between 函数，来检查每个元素是否在 a 和 b 之间（边界包含在内）。
下例模拟行情数据，创建一个测试表 t：

```
t=table(rand("d"+string(1..n),n) as ts_code, nanotimestamp(2008.01.10+1..n) as trade_date, rand(n,n) as open, rand(n,n) as high, rand(n,n) as low, rand(n,n) as close, rand(n,n) as pre_close, rand(n,n) as change, rand(n,n) as pct_chg, rand(n,n) as vol, rand(n,n) as amount, rand(n,n) as downAvgPrice, rand(n,n) as upAvgPrice, rand(1 0,n) as singna)
```

rolling 和 moving 类似，都将函数运算符应用到滑动窗口，进行窗口计算。两者也有细微区别： rolling 可以指定步长 step，moving 的步长为 1；且两者对空值的处理也不相同。详情可参考 rolling 的空值处理 。

#### 3.4.2. moving(sum) 和 msum 性能差距

虽然 DolphinDB 提供了高阶函数 moving，但是如果所要进行的计算可以用 m 系列函数（例如 msum , mcount , mavg 等）实现，请避免使用 moving 实现，这是因为 m 系列函数进行了优化，性能远超 moving。下面以 moving(sum) 和 msum 为例：

```
x=1..1000000
timer moving(sum, x, 10)
timer msum(x, 10)
```

根据数据量的不同，msum 比 moving(sum) 计算耗时缩短 50 至 200 倍。
性能差距的主要原因如下：
- 取数方式不同：msum 是一次性将数据读入内存，无需为每次计算任务单独分配内存；moving(sum) 每次计算都会生成一个子对象，每次计算都需要为子对象申请内存，计算完成后还需要进行内存回收。
- msum 为增量计算，每次窗口计算都使用上一个窗口计算的结果。即直接加上当前窗口新合入的数据，并减去上一个窗口的第一条数据；而 moving(sum) 为全量计算，即每次计算都会累加窗口内的所有数据。

### 3.5. eachPre 使用案例

创建一个表 t，包含 sym 和 BidPrice 两列：

```
t = table(take(`a`b`c`d`e ,100) as sym, rand(100.0,100) as bidPrice)
```

- 生成新的一列 ln 用于存储以下因子的计算结果：先计算当前的 bidPrice 值除以前 3 行 bidPrice 均值的结果（不包括当前行），然后取自然对数。
- 基于列 ln，生成新列 clean 用于存储以下因子的计算结果：计算 ln 的绝对值，若该值大于波动范围阈值 F，则取上一条记录的 ln 值，反之则认为当前报价正常，并保留当前的 ln 值。
根据 ln 列的因子计算规则，可以分析出该问题涉及到滑动窗口计算，窗口的大小为 3。参考 3.4.1 的 moving 案例，具体脚本如下：

```
t2 = select *, log(bidPrice / prev(moving(avg, bidPrice,3))) as ln from t
```

由于内置函数 msum , mcount 和 mavg 比 moving 高阶函数有更好的性能，可以将上述脚本改写如下：

```
//method 1
t2 = select *, log(bidPrice / prev(mavg(bidPrice,3))) as ln from t

//method 2
t22 = select *, log(bidPrice / mavg(prev(bidPrice),3)) as ln from t
```

此处调用 prev 函数获取前一行的数据。
“先计算均值再移动结果”和“先移动列再计算均值”效果等价的。唯一的区别是：表 t22 第三行会产生一个结果。
对于第二个数据处理要求，我们假设波动返回 F 为 0.02, 然后实现一个自定义函数 cleanFun 来实现其取值逻辑，如下：:

```
F = 0.02
def cleanFun(F, x, y): iif(abs(x) > F, y, x)
```

这里的参数 x 表示当前值，y 表示前一个值。然后调用高阶函数 eachPre 来对相邻元素两两计算，该函数等价于实现：F(X[0], pre), F(X[1], X[0]), ..., F(X[n], X[n-1])。对应脚本如下：

```
t2[`clean] = eachPre(cleanFun{F}, t2[`ln])
```


```
F = 0.02
t = table(take(`a`b`c`d`e ,100) as sym, rand(100.0,100) as bidPrice)
t2 = select *, log(bidPrice / prev(mavg(bidPrice,3))) as ln from t
def cleanFun(F,x,y) : iif(abs(x) > F, y,x)
t2[`clean] = eachPre(cleanFun{F}, t2[`ln])
```


### 3.6. byRow 使用案例

计算矩阵每行最大值的下标。下例生成一个矩阵 m：

```
a1=2 3 4
a2=1 2 3
a3=1 4 5
a4=5 3 2
m = matrix(a1,a2,a3,a4)
```

一种思路是，对每行分别计算最大值的下标，可以直接调用 imax 函数实现。 imax 在矩阵每列单独计算，返回一个向量。
为求每行的计算结果，可以先对矩阵进行转置操作，然后调用 imax 函数进行计算。

```
imax(m.transpose())
```

此外，DolphinDB 还提供了一个高阶函数 byRow ，对矩阵的每一行应用指定函数进行计算。使用该函数可以避免转置操作。

```
byRow(imax, m)
```

以上操作亦可用行计算函数 rowImax 来实现：

```
print rowImax(m)
```


### 3.7. segmentby 使用案例

高阶函数 segmentby 。其语法如下：

```
segmentby(func, funcArgs, segment)
```

根据 segment 参数取值确定分组方案，连续的相同值分为一组，进行分组计算。返回的结果与 segment 参数的长度相同。

```
x=1 2 3 0 3 2 1 4 5
y=1 1 1 -1 -1 -1 1 1 1
segmentby(cumsum,x,y);
```

上例中，根据 y 确定了 3 个分组：1 1 1, -1 -1 -1 和 1 1 1，由此把 x 也分为 3 组：1 2 3, 0 3 2 和 1 4 5，并将 cumsum 函数应用到 x 的每个分组，计算每个分组的累计和。
DolphinDB 还提供了内置函数 segment 用于在 SQL 语句中进行分组。与 segmentby 不同，它只返回分组信息，而不对分组进行计算。。
下例中，将表的某列数据按照给定阈值进行分组，连续小于或大于该阈值的数据被划分为一组。连续大于该阈值的分组将保留组内最大值对应的记录并输出（若有重复值则输出第一条）。
表内容如下图所示，当阈值为 0.3 时，希望结果保留箭头所指记录：

```
dated = 2021.09.01..2021.09.12
v = 0 0 0.3 0.3 0 0.5 0.3 0.7 0 0 0.3 0
t = table(dated as date, v)
```

将数据按照是否连续大于 minV 来分组时，可以使用函数 segment 。

```
segment(v>= minV)
```

在 SQL 中配合 context by 语句进行分组计算，通过 having 子句过滤分组的最大值。过滤结果可能存在多行，根据需求只保留第一行满足结果的数据，此时可以通过指定 limit 子句限定输出的记录数。
完整的 SQL 查询语句如下：

```
select * from t context by segment(v>= minV) having (v=max(v) and v>=minV) limit 1
```


### 3.8. pivot 使用案例

高阶函数 pivot 可以在指定的二维维度上重组数据，结果为一个矩阵。
现有包含 4 列数据的表 t1:

```
syms=`600300`600400`600500$SYMBOL
sym=syms[0 0 0 0 0 0 0 1 1 1 1 1 1 1 2 2 2 2 2 2 2]
time=09:40:00+1 30 65 90 130 185 195 10 40 90 140 160 190 200 5 45 80 140 170 190 210
price=172.12 170.32 172.25 172.55 175.1 174.85 174.5 36.45 36.15 36.3 35.9 36.5 37.15 36.9 40.1 40.2 40.25 40.15 40.1 40.05 39.95
volume=100 * 10 3 7 8 25 6 10 4 5 1 2 8 6 10 2 2 5 5 4 4 3
t1=table(sym, time, price, volume);
t1;
```

将 t1 的数据依据 time 和 sym 维度进行数据重组，并且计算每分钟股价的加权平均值，以交易量为权重。

```
stockprice=pivot(wavg, [t1.price, t1.volume], minute(t1.time), t1.sym)
stockprice.round(2)
```


### 3.9. contextby 使用案例

高阶函数 contextby 可以将数据根据列字段分组，并在组内调用指定函数进行计算。

```
sym=`IBM`IBM`IBM`MS`MS`MS
price=172.12 170.32 175.25 26.46 31.45 29.43
qty=5800 700 9000 6300 2100 5300
trade_date=2013.05.08 2013.05.06 2013.05.07 2013.05.08 2013.05.06 2013.05.07;
contextby(avg, price, sym);
```

contextby 亦可搭配 SQL 语句使用。下例调用 contextby 筛选出价格高于组内平均价的交易记录：

```
t1=table(trade_date,sym,qty,price);
select trade_date, sym, qty, price from t1 where price > contextby(avg, price,sym);
```


### 3.10. call/unifiedCall 使用案例

对需要批量调用不同函数进行计算的场景，可以通过高阶函数 call 或者 unifiedCall 配合高阶函数 each / loop 实现。
注： call 和 unifiedCall 功能相同，但参数形式不同，详情可参考用户手册。
下例中在部分应用中调用了函数 call 函数，该部分应用将向量 [1, 2, 3] 作为固定参数，在高阶函数 each 中调用函数 sin 与 log 。

```
each(call{, 1..3},(sin,log));
```

此外，还可通过元编程方式调用函数。这里会用到 funcByName 。上述例子可改写为：

```
each(call{, 1..3},(funcByName('sin'),funcByName('log')));
```

或者，使用 makeCall / makeUnifiedCall 生成元代码，后续通过 eval 来执行：

```
each(eval, each(makeCall{,1..3},(sin,log)))
```


### 3.11. accumulate 使用案例

已知分钟线数据如下，将某只股票每成交约 150 万股进行一次时间切分，最后得到时间窗口长度不等的若干条数据。具体的切分规则为：若某点的数据合入分组，可以缩小数据量和阈值（150 万）间的差值，则加入该点，否则当前分组不合入该点的数据。示意图如下：

```
timex = 13:03:00+(0..27)*60
volume = 288658 234804 182714 371986 265882 174778 153657 201388 175937 138388 169086 203013 261230 398871 692212 494300 581400 348160 250354 220064 218116 458865 673619 477386 454563 622870 458177 880992
t = table(timex as time, volume)
```

这里自定义一个分组计算函数，将 volume 的累加，按上述切分规则，以 150 万为阈值进行分组。
先定义一个分组函数，如下：

```
def caclCumVol(target, preResult, x){
 result = preResult + x
 if(result - target> target - preResult) return x
 else return result
}
accumulate(caclCumVol{1500000}, volume)
```

上述脚本通过自定义函数 caclCumVol 计算 volume 的累加值，结果最接近 150 万时划分分组。新的分组将从下一个 volume 值开始重新累加。对应脚本如下：

```
iif(accumulate(caclCumVol{1500000}, volume) ==volume, timex, NULL).ffill()
```

通过和 volume 比较，筛选出了每组的起始记录。若中间结果存在空值，则调用 ffill 函数进行前值填充。将获得的结果配合 group by 语句进行分组计算，查询时，注意替换以上脚本的 timex 为表的 time 字段。

```
output = select sum(volume) as sum_volume, last(time) as endTime from t group by iif(accumulate(caclCumVol{1500000}, volume) ==volume, time, NULL).ffill() as startTime
```


### 3.12. window 使用案例

对表中的某列数据进行以下计算，如果当前数值是前 5 个数据的最低值 (包括当前值)，也是后 5 个最低值 (包括当前值)，那么标记是 1，否则是 0。

```
t = table(rand(1..100,20) as id)
```

可以通过应用窗口函数 window ，指定一个前后都为 5 的数据窗口，在该窗口内通过调用 min 函数计算最小值。注意：函数 window 的窗口边界包含在窗口中。

```
select *, iif(id==window(min, id, -4:4), 1, 0) as mid from t
```


### 3.13. reduce 使用案例

上面的一些案例中，也有用到高阶函数 reduce 。伪代码如下：

```
result=<function>(init,X[0]);
for(i:1~size(X)){
  result=<function>(result, X[i]);
}
return result;
```

与 accumulate 返回中间结果不同， reduce 只返回最后一个结果。
例如下面的计算阶乘的例子：

```
r1 = reduce(mul, 1..10);
r2 = accumulate(mul, 1..10)[9];
```

最终 r1 和 r2 的结果是一样的。

## 4. 部分应用案例

部分应用是指固定一个函数的部分参数，产生一个参数较少的函数。部分应用通常应用在对参数个数有特定要求的高阶函数中。

### 4.1. 提交带有参数的作业

假设需要一个 定时任务 ，每日 0 点执行，用于计算某设备前一日温度指标的最大值。
假设设备的温度信息存储在分布式库 dfs://dolphindb 下的表 sensor 中，其时间字段为 ts ，类型为 DATETIME。下例定义一个 getMaxTemperature 函数来实现计算过程，脚本如下：

```
def getMaxTemperature(deviceID){
    maxTemp=exec max(temperature) from loadTable("dfs://dolphindb","sensor")
            where ID=deviceID ,date(ts) = today()-1
    return  maxTemp
}
```

定义计算函数后，可通过函数 scheduleJob 提交定时任务。由于函数 scheduleJob 不提供接口供任务函数进行传参，而自定义函数 getMaxTemperature 以设备 deviceID 作为参数，这里可以通过部分应用来固定参数，从而产生一个没有参数的函数。脚本如下：

```
scheduleJob(`testJob, "getMaxTemperature", getMaxTemperature{1}, 00:00m, today(), today()+30, 'D');
```

上例只查询了设备号为 1 的设备。

### 4.2. 获取集群其它节点作业信息

在 DolphinDB 中提交定时作业后，可通过函数 getRecentJobs 来取得本地节点上最近几个批处理作业的状态。如查看本地节点最近 3 个批处理作业状态，可以用如下所示脚本实现：

```
getRecentJobs(3);
```

若想获取集群上其它节点的作业信息，需通过函数 rpc 来在指定的远程节点上调用内置函数 getRecentJobs 。如获取节点别名为 P1-node1 的作业信息，可以如下实现：

```
rpc("P1-node1",getRecentJobs)
```

如需获取节点 P1-node1 上最近 3 个作业的信息，通过如下脚本实现会报错：

```
rpc("P1-node1",getRecentJobs(3))
```

因为 rpc 函数第二个参数需要为函数（内置函数或用户自定义函数）。这里可以通过 DolphinDB 的部分应用，固定函数参数，来生成一个新的函数给 rpc 使用，如下：

```
rpc("P1-node1",getRecentJobs{3})
```


### 4.3. 带“状态”的流计算消息处理函数

在流计算中，用户通常需要给定一个消息处理函数，接受到消息后进行处理。这个处理函数是一元函数或数据表。若为函数，用于处理订阅数据，其唯一的参数是订阅的数据，即不能包含状态信息。
下例通过部分应用定义消息处理函数 cumulativeAverage ，用于计算数据的累计均值。
定义流数据表 trades，对于其 price 字段，每接受一条消息，计算一次 price 的均值，并输出到结果表 avgTable 中。脚本如下：

```
share streamTable(10000:0,`time`symbol`price, [TIMESTAMP,SYMBOL,DOUBLE]) as trades
avgT=table(10000:0,[`avg_price],[DOUBLE])

def cumulativeAverage(mutable avgTable, mutable stat, trade){
   newVals = exec price from trade;

   for(val in newVals) {
      stat[0] = (stat[0] * stat[1] + val )/(stat[1] + 1)
      stat[1] += 1
      insert into avgTable values(stat[0])
   }
}

subscribeTable(tableName="trades", actionName="action30", handler=cumulativeAverage{avgT,0.0 0.0}, msgAsTable=true)
```

自定义函数 cumulativeAverage 的参数 avgTable 为计算结果的存储表。stat 是一个向量，包含了两个值：其中，stat[0] 用来表示当前的所有数据的平均值，stat[1] 表示数据个数。函数体的计算实现为：遍历数据更新 stat 的值，并将新的计算结果插入表。
订阅流数据表时，通过在 handler 中固定前两个参数，实现带“状态”的消息处理函数。

## 5. 金融场景相关案例


### 5.1. 使用 map reduce，对 tick 数据降精度

下例中，使用 mr 函数（map reduce）将 tick 数据转化为分钟级数据。
在 DolphinDB 中，可以使用 SQL 语句基于 tick 数据计算分钟级数据：

```
minuteQuotes=select avg(bid) as bid, avg(ofr) as ofr from t group by symbol,date,minute(time) as minute
```

此例也可以通过 DolphinDB 的分布式计算框架实现。Map-Reduce 函数 mr 是 DolphinDB 通用分布式计算框架的核心功能。

```
login(`admin, `123456)
db = database("dfs://TAQ")
quotes = db.loadTable("quotes")

//create a new table quotes_minute
model=select  top 1 symbol,date, minute(time) as minute,bid,ofr from quotes where date=2007.08.01,symbol=`EBAY
if(existsTable("dfs://TAQ", "quotes_minute"))
db.dropTable("quotes_minute")
db.createPartitionedTable(model, "quotes_minute", `date`symbol)

//populate data for table quotes_minute
def saveMinuteQuote(t){
minuteQuotes=select avg(bid) as bid, avg(ofr) as ofr from t group by symbol,date,minute(time) as minute
loadTable("dfs://TAQ", "quotes_minute").append!(minuteQuotes)
return minuteQuotes.size()
}

ds = sqlDS(<select symbol,date,time,bid,ofr from quotes where date between 2007.08.01 : 2007.08.31>)
timer mr(ds, saveMinuteQuote, +)
```


### 5.2. 数据回放和高频因子计算

有状态的因子，即因子的计算不仅用到当前数据，还会用到历史数据。实现状态因子的计算，一般包括这几个步骤：
- 保存本批次的消息数据到历史记录；
- 根据更新后的历史记录，计算因子
- 将因子计算结果写入输出表中。如有必要，删除未来不再需要的的历史记录。
DolphinDB 的消息处理函数必须是单目函数，其唯一的参数就是当前的消息。要保存历史状态并在消息处理函数中计算历史数据，可以通过部分应用实现：对于多参数的消息处理函数，保留一个参数用于接收消息，固化其它所有的参数，用于保存历史状态。这些固化参数只对消息处理函数可见，不受其他应用的影响。
历史状态可保存在内存表，字典或分区内存表中。本例将使用 DolphinDB 流计算引擎 来处理 报价数据 通过字典保存历史状态并计算因子。如需通过内存表或分布式内存表保存历史状态，可以参考 实时计算高频因子 。
定义状态因子：计算当前第一档卖价 (askPrice1) 与 30 个报价前的第一档卖价的比值。
对应的因子计算函数 factorAskPriceRatio 实现如下：:

```
defg factorAskPriceRatio(x){
 cnt = x.size()
 if(cnt < 31) return double()
 else return x[cnt - 1]/x[cnt - 31]
}
```

导入数据创建对应的流数据表后，可以通过 replay 函数回放数据，模拟实时流计算的场景。

```
quotesData = loadText("/data/ddb/data/sampleQuotes.csv")

x=quotesData.schema().colDefs
share streamTable(100:0, x.name, x.typeString) as quotes1
```

由于这里使用字典保存历史状态，可以定义如下字典：

```
history = dict(STRING, ANY)
```

该字典的键值为 STRING 类型，存储股票字段，值为元组（tuple）类型，存储卖价的历史数据。
下例调用 dictUpdate! 函数更新字典，然后循环计算每只股票的因子，并通过表存储因子的计算结果。然后订阅流表，通过数据回放向流表注入数据，每到来一条新数据都将触发因子的计算。
消息处理函数定义如下：

```
def factorHandler(mutable historyDict, mutable factors, msg){
 historyDict.dictUpdate!(function=append!, keys=msg.symbol, parameters=msg.askPrice1, initFunc=x->array(x.type(), 0, 512).append!(x))
 syms = msg.symbol.distinct()
 cnt = syms.size()
 v = array(DOUBLE, cnt)
 for(i in 0:cnt){
     v[i] = factorAskPriceRatio(historyDict[syms[i]])
 }
 factors.tableInsert([take(now(), cnt), syms, v])
}
```

参数 historyDict 为保存历史状态的字典，factors 是存储计算结果的表。

```
quotesData = loadText("/data/ddb/data/sampleQuotes.csv")

defg factorAskPriceRatio(x){
 cnt = x.size()
 if(cnt < 31) return double()
 else return x[cnt - 1]/x[cnt - 31]
}
def factorHandler(mutable historyDict, mutable factors, msg){
 historyDict.dictUpdate!(function=append!, keys=msg.symbol, parameters=msg.askPrice1, initFunc=x->array(x.type(), 0, 512).append!(x))
 syms = msg.symbol.distinct()
 cnt = syms.size()
 v = array(DOUBLE, cnt)
 for(i in 0:cnt){
     v[i] = factorAskPriceRatio(historyDict[syms[i]])
 }
 factors.tableInsert([take(now(), cnt), syms, v])
}

x=quotesData.schema().colDefs
share streamTable(100:0, x.name, x.typeString) as quotes1
history = dict(STRING, ANY)
share streamTable(100000:0, `timestamp`symbol`factor, [TIMESTAMP,SYMBOL,DOUBLE]) as factors
subscribeTable(tableName = "quotes1", offset=0, handler=factorHandler{history, factors}, msgAsTable=true, batchSize=3000, throttle=0.005)

replay(inputTables=quotesData, outputTables=quotes1, dateColumn=`date, timeColumn=`time)
```


```
select top 10 * from factors where isValid(factor)
```


### 5.3. 基于字典的计算

下例创建表 orders，该表包含了一些简单的股票信息：

```
orders = table(`IBM`IBM`IBM`GOOG as SecID, 1 2 3 4 as Value, 4 5 6 7 as Vol)
```

创建一个字典。键为股票代码，值为从 orders 表中筛选出来的只包含该股票信息的子表。

```
historyDict = dict(STRING, ANY)
```

然后通过函数 dictUpdate! ，来更新每个键的值，实现如下：

```
historyDict.dictUpdate!(function=def(x,y){tableInsert(x,y);return x}, keys=orders.SecID, parameters=orders, initFunc=def(x){t = table(100:0, x.keys(), each(type, x.values())); tableInsert(t, x); return t})
```

可以把 dictUpdate! 的执行过程理解成，针对参数 parameters 遍历，每个 parameters 作为参数，通过 function 去更新字典 (字典的 key 由 keys 指定的)。当字典中不存在对应的 key 时，会调用 initFunc 去初始化 key 对应的值。
这个例子中，字典的 key 是股票代码，value 是 orders 的子表。
这里，我们使用 orders.SecID 作为 keys，在更新的函数参数中，我们定义了一个 lamda 函数将当前记录插入到表中，如下：

```
def(x,y){tableInsert(x,y);return x}
```

注意此处使用 lamda 函数封装了 tableInsert ，而非指定 function=tableInsert。这是因为 tableInsert 的返回值不是一个 table，而是插入的条数，如果直接调用 tableInsert ，在写入第二条 IBM 对应的记录时，会将字典中的值更新成插入的条数；写入第三条 IBM 对应的记录时，系统会抛出异常。
初始条件下，historyDict 未赋值，可以通过指定 initFunc 参数对字典进行初始化赋值：

```
def(x){
  t = table(100:0, x.keys(), each(type, x.values()));
  tableInsert(t, x);
  return t
}
```


```
orders = table(`IBM`IBM`IBM`GOOG as SecID, 1 2 3 4 as Value, 4 5 6 7 as Vol)
historyDict = dict(STRING, ANY)
historyDict.dictUpdate!(function=def(x,y){tableInsert(x,y);return x}, keys=orders.SecID, parameters=orders,
            initFunc=def(x){t = table(100:0, x.keys(), each(type, x.values())); tableInsert(t, x); return t})
```

执行后 historyDict 结果如下：

```
GOOG->
Vol Value SecID
--- ----- -----
7   4     GOOG

IBM->
Vol Value SecID
--- ----- -----
4   1     IBM
5   2     IBM
6   3     IBM
```


## 6. 机器学习相关案例


### 6.1. ols 残差

创建样本表 t 如下：

```
t=table(2020.11.01 2020.11.02 as date, `IBM`MSFT as ticker, 1.0 2 as past1, 2.0 2.5 as past3, 3.5 7 as past5, 4.2 2.4 as past10, 5.0 3.7 as past20, 5.5 6.2 as past30, 7.0 8.0 as past60)
```

计算每行数据和一个向量 benchX 的回归残差，并将结果保存到新列中。
向量 benchX 如下：

```
benchX = 10 15 7 8 9 1 2.0
```

DolphinDB 提供了最小二乘回归函数 ols 。
先将表中参与计算的以下列转化成矩阵：

```
mt = matrix(t[`past1`past3`past5`past10`past20`past30`past60]).transpose()
```

然后定义残差计算函数如下：

```
def(y, x) {
    return ols(y, x, true, 2).ANOVA.SS[1]
}
```

最后使用高阶函数 each 与部分应用，对每行数据应用残差计算函数：

```
t[`residual] = each(def(y, x){ return ols(y, x, true, 2).ANOVA.SS[1]}{,benchX}, mt)
```


## 7. 总结

除了上面提到的一些函数与高阶函数。DolphinDB 还提供了丰富的函数库，包括数学函数、统计函数、分布相关函数、假设检验函数、机器学习函数、逻辑函数、字符串函数、时间函数、数据操作函数、窗口函数、高阶函数、元编程、分布式计算函数、流计算函数、定时任务函数、性能监控函数、用户权限管理函数等。
