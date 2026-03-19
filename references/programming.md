# DolphinDB 参考：编程语言深度指南

> 来源：`progr/lang_intro.html`, `progr/data_types.html`, `progr/data_forms.html`,
> `progr/data_types_forms/Dictionary.html`, `progr/data_types_forms/vector.html`,
> `progr/operators/operators.html`, `progr/statements/`

---

## ⚠️ 与 Python/JS 差异速查（最容易踩坑）

> **核心提示：DolphinDB 是小众专业语言，绝大多数 Python/JavaScript 习惯在此不适用！**

### 1. 字典（Dictionary）

DolphinDB 字典**没有** `.get()` / `.items()` / `.keys()` / `.values()` 等 Python 方法，访问方式完全不同：

```dolphindb
// ✅ 正确方式
d = dict(`a`b`c, 1 2 3)
d[`a]             // 读取键 "a" → 1
d[`a`b]           // 批量读取 → [1, 2]
keys d            // 获取所有键（等价 d.keys()）
values d          // 获取所有值（等价 d.values()）
size d            // 获取长度（等价 len(d)）
1 in d            // 键存在检测 → false（等价 1 in d.keys()）
`a in d           // → true

// ✅ 查找键值（带默认值的方式：用 in 先判断）
def getWithDefault(dic, key, default) {
    if (key in dic) { return dic[key] }
    return default
}
getWithDefault(d, `a, 0)     // 相当于 d.get("a", 0)
getWithDefault(d, `z, 0)     // 键不存在时返回 0

// ✅ 也可用 find 函数（键不存在返回 NULL，不报错）
find(d, `a `z)    // [`a`z 的结果向量，z 位置为 NULL]
d find `a         // 中缀调用，返回 1

// ❌ 以下写法全部错误！
// d.get("a", 0)   ← 不存在
// d.items()       ← 不存在
// d.update(d2)    ← 不存在（用 d2 的键值覆盖 d1 请用 dictUpdate!）
// for k, v in d:  ← 语法不支持，用 for k in keys(d) 遍历键
```

### 2. 字典创建方式

```dolphindb
// ✅ 方式一：dict(keys, values)
d = dict(`a`b`c, 1 2 3)

// ✅ 方式二：JSON 字面量（3.x 支持）
d = {"a": 1, "b": 2, "c": 3}

// ✅ 方式三：先声明类型，再逐个插入
d = dict(STRING, INT)
d["x"] = 10
d["y"] = 20

// ✅ 有序字典（保留插入顺序）
d = dict(`a`b`c, 1 2 3, true)   // 第三个参数 true = ordered

// ❌ 错误写法
// d = {"a": 1}   ← 仅 3.x 支持，旧版本不行
// d.a = 1        ← 不能用点赋值（d.a 是只读访问）
```

### 3. 修改字典

```dolphindb
d = dict(STRING, INT)
d["key1"] = 100           // 新增或修改
d["key1"] = 200           // 更新值

// 批量更新多个键
d["key1" "key2"] = 100 200

// dictUpdate!（带运算的更新）
dictUpdate!(d, add, "key1", 50)  // d["key1"] += 50

// 删除键
d.erase!("key1")

// 清空字典
d.clear!()

// ❌ 错误写法
// del d["key1"]      ← Python del 不存在
// d.pop("key1")      ← 不存在
// d.key1 = 1         ← 点赋值不可用于修改
```

### 4. 遍历字典

```dolphindb
d = dict(`a`b`c, 1 2 3)

// 遍历键
for (k in keys(d)) {
    print(k + " -> " + string(d[k]))
}

// 获取键值对（没有 items()，需手动操作）
ks = keys(d)
vs = values(d)
for (i in 0..(size(d)-1)) {
    print(ks[i] + " -> " + string(vs[i]))
}
```

---

### 5. 向量（Vector）与 Python 列表的差异

```dolphindb
// 创建向量
v = 1 2 3 4 5              // 空格分隔（最常用）
v = [1, 2, 3, 4, 5]        // 方括号语法
v = 1..10                  // 范围（1到10，步长1）
v = array(INT, 5, 0)       // 长度5，填0

// ❌ 常见误区
// v = [1, 2, 3]  ← 是元组(any vector)，不是同类型向量
// v.append(1)    ← 不存在！用 append!(v, 1)
// len(v)         ← 不存在！用 size(v) 或 count(v)

// 正确的追加和修改
append!(v, 6)      // 原地追加
v.append!(6)       // OOP 风格
v[0] = 99          // 修改第一个元素

// 向量切片（注意：索引从0开始，与 Python 相同）
v[0]               // 第一个元素
v[0 2 4]           // 第1、3、5个元素（向量化索引）
v[0:3]             // ❌ 错误！DolphinDB 不支持切片语法
v[0..2]            // ✅ 正确！相当于 v[0], v[1], v[2]
subarray(v, 0, 3)  // 从位置0取3个元素 = v[0:3]

// 向量操作（全部向量化，无需循环）
v * 2              // 每个元素乘2
v + 1              // 每个元素加1
v > 3              // 布尔向量
v[v > 3]           // 条件筛选（类似 numpy 布尔索引！）
sum(v)             // 求和
avg(v)             // 平均
```

### 6. 字符串与 Symbol

```dolphindb
// 三种字符串字面量（等价）
s1 = "Hello"
s2 = 'Hello'
s3 = `Hello          // 反引号：创建 SYMBOL 类型（不是 STRING！）

// ⚠️ 重要：SYMBOL 类型特殊性
// - SYMBOL是整数编码的字符串（枚举化），不能用字符串拼接
// - 适合高频重复的分类值（如股票代码、业务类型）
// - 分区内不同 SYMBOL 值上限 2^21 = 2097152

// 字符串拼接
s = "Hello" + ", " + "World"  // ❌ 不支持！
s = concat("Hello", ", World") // ✅ 正确
s = format("{} → {}", `price, 3.14)  // 格式化

// Symbol vs String
sym = `AAPL           // SYMBOL 类型
str = "AAPL"          // STRING 类型
// 查询时两者一般可互换，但写分布式表时要注意列定义类型

// 字符串比较（大小写敏感）
"abc" == "ABC"   // false
"abc" < "abd"    // true

// 字符串函数
strlen("hello")     // 5（不是 len()）
upper("hello")      // "HELLO"
lower("HELLO")      // "hello"
trim("  hi  ")      // "hi"
split("a,b,c", ",") // ["a","b","c"]
substr("hello", 1, 3)  // "ell"（从1号索引取3个字符）
regexFind("abc123", "[0-9]+")  // "123"
```

### 7. 比较运算符

```dolphindb
// ⚠️ DolphinDB 比较结果是整数（1/0），不是 Python 的 True/False
3 > 2    // 1（不是 true）
3 < 2    // 0（不是 false）
3 == 3   // 1
3 != 2   // 1（也可写 3 <> 2）

// 条件中可用 1/0，也可用 true/false
if (x > 0) { ... }    // 两种方式均可

// ⚠️ 非 SQL 表达式中，= 是赋值，不是比较！
// 比较必须用 ==
x = 3       // 赋值
x == 3      // 比较

// SQL 中，= 是比较（与标准 SQL 一致）
select * from t where price = 100   // ✅ SQL 中用 =
select * from t where price == 100  // ✅ 也可用 ==
```

### 8. NULL 与空值

```dolphindb
// NULL 不是 Python 的 None，而是针对各类型有专用 NULL
nullInt = int()            // INT NULL（最小 INT 值）
nullDouble = double()      // DOUBLE NULL（NaN）
nullStr = string(NULL)     // STRING NULL

// 判断 NULL
isNull(x)
isValid(x)    // 等价 !isNull(x)

// ⚠️ NULL 参与运算
NULL + 1   // → NULL（NULL 会传播）
sum(1 2 NULL 4)  // → 7（聚合函数会自动忽略 NULL！）
avg(1 2 NULL 4)  // → 7/3 = 2.333...（分母只算非 NULL 数量）

// 填充 NULL
nullFill(v, 0)      // 把 NULL 替换为 0
ffill(v)            // 前向填充
bfill(v)            // 后向填充
```

### 9. 赋值与作用域

```dolphindb
// = 赋值（智能：如果变量已存在则在其作用域更新，否则创建局部变量）
x = 10

// := 强制创建局部变量（即使外部已有同名变量）
x := 20

// 全局变量（在函数内访问外部变量）
globalVar = 100
def myFunc() {
    // 读取全局变量：需要显式声明！
    // x = globalVar  ❌ 这只会创建局部变量 x
    // 正确方式：
    share globalVar as localAlias  // 或者用 rfc/remoteRun 等机制
}

// ⚠️ 函数内部的变量修改不影响外部（值传递，不是引用传递）
def modifyVector(v) {
    v[0] = 99    // 只修改了局部副本
}
arr = 1 2 3
modifyVector(arr)
arr     // 仍然是 [1, 2, 3]，没有被修改！
```

### 10. 函数调用的三种等价写法

```dolphindb
// 以 log(x) 为例
log(10)     // 标准调用
log 10      // 前缀调用（仅单参数，不能嵌套）
10.log()    // 面向对象调用

// 以 add(x, y) / pow(x, y) 为例
add(3, 4)    // 标准调用
3 add 4      // 中缀调用（仅双参数，不能嵌套）
3.add(4)     // 面向对象调用
pow(2, 10)   // → 1024

// ⚠️ 前缀/中缀调用不能嵌套
// log sqrt 10  ← 错误！
// 应写为：
log(sqrt(10))  // ← 正确：嵌套时必须用括号
```

### 11. 分号与换行

```dolphindb
// DolphinDB 中分号是语句分隔符（可选，换行也可以）
x = 1; y = 2; z = 3       // 同一行用分号分隔

// 换行即语句结束
x = 1
y = 2

// ⚠️ 特殊：向量字面量中不能换行（除非用括号）
v = 1 2 3     // ✅ 向量
// v = 1
//     2 3    // ❌ 错误：1 是标量，2 3 是另一个向量
v = (1
     2 3)     // ✅ 括号内可换行
```

### 12. 布尔逻辑

```dolphindb
// AND / OR 运算符
x = 5
x > 3 && x < 10   // → true（逻辑 AND，短路）
x > 3 and x < 10  // → true（same）
x < 0 || x > 3    // → true（逻辑 OR）
x < 0 or x > 3    // → true（same）
!( x > 3)         // → false（逻辑 NOT）
not (x > 3)       // → false（same）

// 向量化的位运算（比较两个向量）
[1,2,3] & [1,0,1]    // 按位 AND
[1,2,3] | [0,1,0]    // 按位 OR
```

---

## 数据类型

### 完整数据类型表

| 分类 | 名称 | 示例 | 符号 | 字节数 |
|------|------|------|------|--------|
| VOID | VOID | NULL | | 1 |
| LOGICAL | BOOL | `1b, 0b, true, false` | b | 1 |
| INTEGRAL | CHAR | `'a', 97c` | c | 1 |
| INTEGRAL | SHORT | `122h` | h | 2 |
| INTEGRAL | INT | `21` | i | 4 |
| INTEGRAL | LONG | `22l` | l | 8 |
| TEMPORAL | DATE | `2013.06.13` | d | 4 |
| TEMPORAL | MONTH | `2012.06M` | M | 4 |
| TEMPORAL | TIME | `13:30:10.008` | t | 4 |
| TEMPORAL | MINUTE | `13:30m` | m | 4 |
| TEMPORAL | SECOND | `13:30:10` | s | 4 |
| TEMPORAL | DATETIME | `2012.06.13T13:30:10` | D | 4 |
| TEMPORAL | TIMESTAMP | `2012.06.13T13:30:10.008` | T | 8 |
| TEMPORAL | NANOTIME | `13:30:10.008007006` | n | 8 |
| TEMPORAL | NANOTIMESTAMP | `2012.06.13T13:30:10.008007006` | N | 8 |
| TEMPORAL | DATEHOUR | `datehour(2012.06.13T13:30:10)` | | 4 |
| FLOATING | FLOAT | `2.1f` | f | 4 |
| FLOATING | DOUBLE | `2.1` | F | 8 |
| LITERAL | SYMBOL | `` `AAPL `` | S | 4 |
| LITERAL | STRING | `"Hello"` | W | ≤64KB |
| LITERAL | BLOB | | | ≤64MB |
| BINARY | UUID | `5d212a78-cc48-e3b1-4235-b4d91473ee87` | | 16 |
| BINARY | IPADDR | `192.168.1.13` | | 16 |
| BINARY | INT128 | `e1671797c52e15f763380b45e841ec32` | | 16 |
| DECIMAL | DECIMAL32(S) | `3.14$DECIMAL32(3)` | | 4 |
| DECIMAL | DECIMAL64(S) | `3.14P` | P | 8 |
| DECIMAL | DECIMAL128(S) | `3.14$DECIMAL128(3)` | | 16 |

**重要规则：**
- 整数溢出：采用二进制补码，**最小值 - 1 表示 NULL**（如 INT NULL = -2147483648）
- SYMBOL 每分区最多 2^21 个不同值；超出会报错
- DOUBLE 默认；FLOAT 精度更低，写法为 `2.1f`

### 类型转换

```dolphindb
// 类型转换函数
int(3.14)           // 3（截断，不是四舍五入）
long(3.14)          // 3L
double(3)           // 3.0
string(123)         // "123"
symbol("AAPL")      // `AAPL
date("2024.01.01")  // 日期

// 类型断言转换（$ 运算符）
3.14 $ INT          // 3
"2024.01.01" $ DATE // 日期标量

// 检查类型
typestr(3l)    // "LONG"
type(3l)       // 5（类型 ID）
form(1 2 3)   // 1（VECTOR 的 form ID）
```

---

## 数据形式（Data Forms）

| 名称 | ID | 创建方式 |
|------|-----|---------|
| Scalar（标量） | 0 | `5`, `` `a ``, `2024.01.01` |
| Vector（向量） | 1 | `1 2 3`, `[1,2,3]`, `1..10` |
| Pair（数据对） | 2 | `3:5`, `'a':'c'` |
| Matrix（矩阵） | 3 | `1..6$2:3`, `matrix(...)` |
| Set（集合） | 4 | `set(1 2 3)` |
| Dictionary（字典） | 5 | `dict(keys, values)` |
| Table（表） | 6 | `table(...)`, `loadTable(...)` |
| Tensor（张量） | 10 | `tensor(1..10$5:2)` |

---

## 字典（Dictionary）完整参考

### 创建

```dolphindb
// 1. 从键向量 + 值向量创建
z = dict(1 2 3, 2.3 4.6 5.3)
// 注意：若键有重复，以最后出现的值为准

// 2. 有序字典（ordered=true，保留插入顺序）
z = dict(1 2 3, 2.3 4.6 5.3, true)

// 3. 先声明类型，再插入
z = dict(INT, DOUBLE)     // 空字典，键 INT、值 DOUBLE
z[1] = 7.9
z[2] = 8.5

// 4. 字符串键 + 任意类型值（heterogeneous）
z = dict(STRING, ANY)
z[`IBM] = 172.91 173.45 171.6   // 值是向量
z[`COUNT] = 10                  // 值是标量

// 5. JSON 字面量（3.x 支持）
d = {"a": 1, "b": "hello", "c": [1,2,3]}

// 6. 并发安全的同步字典
sd = syncDict(STRING, INT)   // 允许多线程并发读写
```

### 访问

```dolphindb
z = dict(`IBM`GOOG`MSFT, 150.0 2800.0 380.0)

// 单个键
z[`IBM]         // 150.0

// 多个键（向量）
z[`IBM`MSFT]    // [150.0, 380.0]

// .键名（数值键时用 .数字，但此方式只读）
z.IBM           // 150.0（等价 z[`IBM]）

// find 函数（键不存在返回 NULL，不报错）
find(z, `IBM `XXX)   // [150.0, NULL]
z find `IBM          // 150.0

// 获取所有键、所有值
keys z           // [`IBM, `GOOG, `MSFT]（向量）
values z         // [150.0, 2800.0, 380.0]

// 长度
size z           // 3

// 键是否存在
`IBM in z        // 1（true）
`AAPL in z       // 0（false）
```

### 修改

```dolphindb
z = dict(STRING, DOUBLE)

// 新增/修改单个
z["IBM"] = 155.0

// 批量新增/修改
z["IBM" "GOOG"] = 155.0 2850.0

// dictUpdate!（带运算的更新，类似 Python 的 d[k] += val）
dictUpdate!(z, add, "IBM", 5.0)   // IBM 的值 += 5.0
dictUpdate!(z, mul, "IBM", 1.01)  // IBM 的值 *= 1.01

// 删除
z.erase!("IBM")    // 删除键 "IBM"

// 清空
z.clear!()
```

### 字典到表的转换

```dolphindb
// 将字典转为表（每个 key 是列名，每个 value 是列数据）
d = dict(`sym`val, [`a`b`c, 1 2 3])
transpose(d)
// 输出：
// sym  val
// a    1
// b    2
// c    3

// 多个字典合并为表（asis + each）
d1 = {'key1':10, 'key2':"A1", 'key3':0.15}
d2 = {'key2':"B1", 'key3':0.16, 'key4':[1,2]}
asis:E([d1, d2])
```

---

## 控制语句

```dolphindb
// if-else
if (x > 0) {
    print("正数")
} else if (x < 0) {
    print("负数")
} else {
    print("零")
}

// for（遍历向量/范围）
for (i in 1..10) {        // 1 到 10
    print(i)
}
for (k in keys(d)) {      // 遍历字典键
    print(d[k])
}

// while
while (x < 100) {
    x = x * 2
}

// do-while
do {
    x = x + 1
} while (x < 10)

// break / continue
for (i in 1..10) {
    if (i == 5) break
    if (i % 2 == 0) continue
    print(i)
}

// try-catch
try {
    x = 1 / 0
} catch (e) {
    print("错误：" + e.message)
}
```

---

## 函数定义

```dolphindb
// 普通函数（非 SQL 环境下变量作用域是局部的）
def add(a, b) {
    return a + b
}
add(3, 4)   // 7

// 向量化操作（自动多线程）
def calcFactor(prices) {
    return prices / mavg(prices, 5) - 1
}

// 默认参数
def greet(name, prefix = "Hello") {
    return prefix + ", " + name + "!"
}
greet("Alice")             // "Hello, Alice!"
greet("Alice", "Hi")       // "Hi, Alice!"

// 匿名函数（Lambda）
double = def(x) { x * 2 }
double(5)   // 10

// 部分应用（关键字 partial，类似 Python 的 functools.partial）
addOne = partial(add, 1)
addOne(5)    // 6
```

---

## 函数化编程

```dolphindb
// each（对每个元素应用函数）
each(sqrt, [1, 4, 9, 16])        // [1, 2, 3, 4]
each(def(x){x*2}, 1..5)          // [2, 4, 6, 8, 10]

// 等价中缀写法
sqrt each [1, 4, 9, 16]

// reduce（聚合，类似 Python reduce）
reduce(+, 1..10)      // 55
reduce(max, 1..10)    // 10

// accumulate（累积计算，类似 cumsum）
accumulate(+, 1..5)   // [1, 3, 6, 10, 15]

// moving（滑动窗口）
moving(avg, 1..10, 3)  // 3 期滑动平均

// 注意：map / filter 行为
each(def(x){x*2}, v)    // = map(lambda x: x*2, v)
v[v > 3]                // = filter(lambda x: x > 3, v)（但用布尔索引更高效）
```
