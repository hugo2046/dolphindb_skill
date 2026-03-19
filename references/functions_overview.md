# DolphinDB 参考：函数参考

> 来源：`funcs/funcs_intro.html`, `funcs/funcs_by_topics.html`

## 函数调用语法

### 一参函数（3 种等价写法）

```dolphindb
log(10)       // 标准调用
log 10        // 前置调用（不支持嵌套函数中使用）
(10).log()    // 面向对象调用
```

### 二参函数（3 种等价写法）

```dolphindb
add(X, Y)     // 标准调用
X add Y       // 中缀调用（不支持嵌套）
X.add(Y)      // 面向对象调用
```

---

## 常用函数分类速查

### 数学/统计函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `abs(x)` | 绝对值 | `abs(-3)` → 3 |
| `sqrt(x)` | 平方根 | `sqrt(4)` → 2 |
| `pow(x,y)` | x 的 y 次方 | `pow(2,10)` → 1024 |
| `log(x)` | 自然对数 | `log(10)` → 2.302585 |
| `exp(x)` | e 的 x 次方 | `exp(1)` → 2.718282 |
| `sum(x)` | 求和 | `sum(1..100)` → 5050 |
| `avg(x)` | 平均值 | `avg(1 2 3)` → 2 |
| `max(x)` | 最大值 | |
| `min(x)` | 最小值 | |
| `count(x)` | 计数（含 NULL） | |
| `std(x)` | 标准差 | |
| `var(x)` | 方差 | |
| `median(x)` | 中位数 | |
| `percentile(x,p)` | 百分位数 | |
| `corr(x,y)` | 相关系数 | |
| `covar(x,y)` | 协方差 | |

### 字符串函数

| 函数 | 说明 |
|------|------|
| `strlen(s)` | 字符串长度 |
| `upper(s)` / `lower(s)` | 转大/小写 |
| `substr(s,start,len)` | 截取子串 |
| `concat(s1,s2)` | 连接字符串 |
| `split(s,delim)` | 拆分字符串 |
| `trim(s)` | 去首尾空格 |
| `regexFind(s,pattern)` | 正则查找 |
| `format(x,fmt)` | 格式化 |

### 时间日期函数

| 函数 | 说明 |
|------|------|
| `now()` | 当前时间（TIMESTAMP） |
| `today()` | 当前日期（DATE） |
| `date(x)` | 提取日期部分 |
| `time(x)` | 提取时间部分 |
| `year(x)` | 提取年份 |
| `month(x)` | 提取月份 |
| `dayOfWeek(x)` | 星期几（0=星期日） |
| `temporalAdd(x,n,unit)` | 时间加法 |
| `businessDay(x)` | 工作日判断 |
| `weekBegin(x)` | 本周起始日 |

### 向量/数组操作

| 函数 | 说明 |
|------|------|
| `size(v)` | 元素个数 |
| `shape(v)` | 维度（矩阵） |
| `sort(v)` | 排序 |
| `rank(v)` | 排名 |
| `reverse(v)` | 反转 |
| `distinct(v)` | 去重 |
| `find(v,x)` | 查找元素位置 |
| `in(v,target)` | 成员测试 |
| `flatten(v)` | 展平 |
| `reshape(v,m,n)` | 重塑 |
| `append!(v,x)` | 追加元素 |
| `delete!(v,idx)` | 删除元素 |

### 表操作函数

| 函数 | 说明 |
|------|------|
| `table(col1,col2,...)` | 创建内存表 |
| `loadTable(db,tbl)` | 加载分布式表 |
| `select` | 查询 |
| `update` | 更新 |
| `insert into` | 插入 |
| `delete from` | 删除记录 |
| `addColumn!(t,name,vals)` | 添加列 |
| `dropColumns!(t,cols)` | 删除列 |
| `rename!(t,old,new)` | 重命名列 |
| `schema(t)` | 表结构信息 |
| `colNames(t)` | 列名 |
| `colDefs(t)` | 列定义（含类型） |

### 数据库操作函数

| 函数 | 说明 |
|------|------|
| `database(path,...)` | 创建/访问数据库 |
| `createTable(db,...)` | 创建表 |
| `createPartitionedTable(...)` | 创建分区表 |
| `loadTable(db,tbl)` | 加载已有表 |
| `dropTable(db,tbl)` | 删除表 |
| `dropDatabase(db)` | 删除数据库 |
| `existsDatabase(path)` | 是否存在库 |
| `existsTable(db,name)` | 是否存在表 |

### 流数据函数

| 函数 | 说明 |
|------|------|
| `enableTableShareAndPersist(t,name,...)` | 启用持久化流表 |
| `subscribeTable(...)` | 订阅流表 |
| `unsubscribeTable(...)` | 取消订阅 |
| `createReactiveStateEngine(...)` | 创建响应式状态引擎 |
| `createTimeSeriesEngine(...)` | 创建时序聚合引擎 |
| `createSessionWindowEngine(...)` | 创建会话窗口引擎 |
| `createCEPEngine(...)` | 创建 CEP 引擎 |
| `getStreamingStat()` | 查看流处理状态 |

### 系统函数

| 函数 | 说明 |
|------|------|
| `getNodeAlias()` | 当前节点别名 |
| `getSessionMemoryStat()` | 会话内存使用 |
| `objs(remote=false)` | 列出变量 |
| `undef(name)` | 删除变量 |
| `clearAllCache()` | 清空缓存 |
| `getConfig(key)` | 读取配置项 |
| `getClusterDFSNodes()` | 集群节点信息 |
| `pnodeRun(f,nodes)` | 在多节点运行 |

---

## 高阶函数

```dolphindb
// each - 对每个元素应用函数
each(sqrt, 1 4 9 16)     // [1, 2, 3, 4]

// reduce - 聚合
reduce(+, 1..10)          // 55

// accumulate - 累积计算
accumulate(+, 1..5)       // [1, 3, 6, 10, 15]

// moving - 移动窗口
moving(avg, data, 5)      // 5 期移动平均

// rolling - 滚动窗口
rolling(corr, x, y, 20)

// byColumn - 按列操作矩阵
byColumn(sum, matrix)
```

---

## 窗口函数

```dolphindb
// 移动平均（mAvg）
mAvg(x, 5)      // 5 窗口移动平均

// 移动求和（mSum）
mSum(x, 20)     // 20 窗口移动求和

// 移动标准差（mStd）
mStd(x, 10)

// 移动最大值（mMax）
mMax(x, 5)

// 超前/滞后
move(x, 1)      // 滞后 1 期（等同于 lag）
move(x, -1)     // 超前 1 期

// 百分比变化
ratios(x) - 1   // 环比变化

// 差分
deltas(x)       // x[i] - x[i-1]
```

---

## TA、金融相关函数

```dolphindb
// 需 include <ta> 模块或 use modules::ta
// MA（简单移动平均）
ta::ma(data, 5)

// EMA（指数移动平均）
ta::ema(data, 5)

// MACD
ta::macd(data, 12, 26, 9)

// RSI
ta::rsi(data, 14)

// Bollinger Bands
ta::bBands(data, 5)
```
