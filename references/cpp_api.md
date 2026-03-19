# DolphinDB 参考：C++ API

> 来源：`api/cpp/Cpp_intro.html`，官方 C++ API 文档

## 头文件与编译

```bash
# 获取 C++ API 库（从官网下载对应版本）
# 头文件：include/DolphinDB.h, include/Util.h 等
# 库文件：libDolphinDBAPI.so (Linux) / DolphinDB.dll (Windows)

# 编译示例（Linux）
g++ -std=c++11 -o myApp myApp.cpp \
    -I./include \
    -L./lib \
    -lDolphinDBAPI \
    -Wl,-rpath,./lib

# CMakeLists.txt 片段
# target_link_libraries(myApp DolphinDBAPI)
# target_include_directories(myApp PRIVATE ${DDB_INCLUDE_DIR})
```

---

## 1. 连接与脚本执行

```cpp
#include "DolphinDB.h"
using namespace dolphindb;

// 创建连接
DBConnection conn;
bool ok = conn.connect("localhost", 8848, "admin", "123456");
if (!ok) {
    std::cerr << "连接失败" << std::endl;
    return -1;
}

// 执行脚本
ConstantSP result = conn.run("1 + 1");
std::cout << result->getInt() << std::endl;  // 2

// 执行 SQL 查询，返回表
TableSP table = (TableSP)conn.run(
    "select * from loadTable('dfs://trade_db','trades') limit 100"
);
std::cout << "行数: " << table->rows() << std::endl;
std::cout << "列数: " << table->columns() << std::endl;

// 关闭
conn.close();
```

---

## 2. 数据类型

```cpp
// 标量
ConstantSP intVal = Util::createInt(42);
ConstantSP dblVal = Util::createDouble(3.14);
ConstantSP strVal = Util::createString("hello");
ConstantSP tsVal  = Util::createTimestamp(/* nanoseconds since epoch */);

// 向量
int size = 5;
VectorSP intVec = Util::createVector(DT_INT, size);
for (int i = 0; i < size; i++) {
    intVec->setInt(i, i + 1);
}

VectorSP dblVec = Util::createVector(DT_DOUBLE, size);
for (int i = 0; i < size; i++) {
    dblVec->setDouble(i, (double)(i + 1) * 1.5);
}

// 字典
DictionarySP dict = Util::createDictionary(DT_STRING, nullptr, DT_ANY, nullptr);
dict->set(Util::createString("key1"), Util::createInt(100));
dict->set(Util::createString("key2"), Util::createDouble(3.14));

// 读字典
ConstantSP val = dict->getMember(Util::createString("key1"));
std::cout << val->getInt() << std::endl;  // 100
```

---

## 3. 创建表并写入

```cpp
// 创建列
std::vector<std::string> colNames = {"TradeTime", "SecurityID", "Price", "Volume"};
std::vector<DATA_TYPE> colTypes = {DT_TIMESTAMP, DT_SYMBOL, DT_DOUBLE, DT_LONG};

int rows = 1000;
VectorSP tsCol  = Util::createVector(DT_TIMESTAMP, rows);
VectorSP symCol = Util::createVector(DT_SYMBOL, rows);
VectorSP priceCol = Util::createVector(DT_DOUBLE, rows);
VectorSP volCol = Util::createVector(DT_LONG, rows);

long long baseTs = 1704067200000LL; // 2024-01-01 00:00:00 UTC (ms)
for (int i = 0; i < rows; i++) {
    tsCol->setLong(i, (baseTs + i * 1000) * 1000000LL); // ns
    symCol->setString(i, "AAPL");
    priceCol->setDouble(i, 150.0 + (i % 10) * 0.1);
    volCol->setLong(i, 10000 + i * 100);
}

// 构造表
std::vector<ConstantSP> cols = {tsCol, symCol, priceCol, volCol};
TableSP data = Util::createTable(colNames, cols);

// 上传并写入分布式表
conn.upload("data", data);
conn.run("loadTable('dfs://trade_db','trades').append!(data)");
```

---

## 4. MultithreadedTableWriter（高性能并发写入）

```cpp
#include "MultithreadedTableWriter.h"

// 创建多线程写入器
MultithreadedTableWriter writer(
    "localhost",           // host
    8848,                  // port
    "admin",               // user
    "123456",              // password
    "dfs://trade_db",      // 数据库路径
    "trades",              // 表名
    false,                 // useSSL
    false,                 // enableHighAvailability
    nullptr,               // highAvailabilitySites
    100,                   // batchSize（每批行数）
    1.0f,                  // throttle（最大等待秒）
    4,                     // threadCount（写入线程数）
    "TradeTime"            // partitionCol（分区列）
);

// 写入数据（线程安全）
ErrorCodeInfo errInfo;
for (int i = 0; i < 1000; i++) {
    writer.insert(errInfo,
        (long long)(1704067200000LL + i) * 1000000LL,  // NANOTIMESTAMP
        "AAPL",                                          // SYMBOL
        150.0 + (i % 10) * 0.1,                         // DOUBLE
        (long long)(10000 + i * 100)                     // LONG
    );
    if (errInfo.hasError()) {
        std::cerr << "写入错误: " << errInfo.errorInfo << std::endl;
    }
}

// 等待写入完成
writer.waitForThreadCompletion();

// 获取写入状态
MultithreadedTableWriter::Status status;
writer.getStatus(status);
std::cout << "已写入: " << status.sendingRows << " 行" << std::endl;
```

---

## 5. 流数据客户端（订阅）

```cpp
#include "StreamingClient.h"

// 消息处理回调
void onMessage(Message msg) {
    ConstantSP data = msg.getBody();
    if (data->isTable()) {
        TableSP t = (TableSP)data;
        std::cout << "收到 " << t->rows() << " 行" << std::endl;
    }
}

// 创建订阅客户端
StreamingClient client(8849);  // 本地监听端口

// 订阅
client.subscribe(
    "localhost",   // 服务端地址
    8848,          // 服务端端口
    onMessage,     // 回调函数
    "trades",      // 流表名
    "cppSub",      // 动作名
    -1             // offset（-1=最新）
);

// 阻塞等待
std::this_thread::sleep_for(std::chrono::seconds(60));

// 取消订阅
client.unsubscribe("localhost", 8848, "trades", "cppSub");
```

---

## 6. 主要 API 速查

| 函数/类 | 说明 |
|---------|------|
| `DBConnection::connect()` | 建立连接 |
| `DBConnection::run()` | 执行脚本/SQL |
| `DBConnection::upload()` | 上传变量 |
| `DBConnection::close()` | 关闭连接 |
| `Util::createInt/Double/String/Timestamp()` | 创建标量 |
| `Util::createVector(type, size)` | 创建向量 |
| `Util::createTable(names, cols)` | 创建表 |
| `Util::createDictionary()` | 创建字典 |
| `MultithreadedTableWriter` | 高并发多线程写入 |
| `StreamingClient` | 流数据订阅 |
| `ConstantSP::getInt/Double/String()` | 读取标量值 |
| `VectorSP::setInt/setDouble()` | 设置向量元素 |
| `TableSP::rows()/columns()` | 表行列数 |
| `TableSP::getColumn(name)` | 按列名获取列向量 |

---

## 7. 与 Python API 的对比映射

从 Python 迁移到 C++ 时，常用 API 的对应关系：

| 场景 | Python API | C++ 等价 |
|------|-----------|----------|
| 创建连接 | `ddb.session()` + `.connect()` | `DBConnection conn; conn.connect()` |
| 执行脚本 | `s.run("...")` | `conn.run("...")` |
| 上传变量 | `s.upload({'x': df})` | `conn.upload("x", tableObj)` |
| 普通写入 | `tableAppender` | `conn.upload() + conn.run("append!")` |
| 高并发写入 | `PartitionedTableAppender` | `MultithreadedTableWriter` |
| 流数据订阅 | `s.subscribe()` + `s.enableStreaming()` | `StreamingClient.subscribe()` |
| 类型转换 | `pd.DataFrame` 自动转换 | 手动逐列构建 `VectorSP` → `Util::createTable()` |
| 查询返回 | `s.run(sql)` → `pd.DataFrame` | `conn.run(sql)` → `TableSP` |
| 读取标量 | Python 原生类型 | `.getInt()` / `.getDouble()` / `.getString()` |

> 💡 **选择建议**：
> - **延迟敏感**（tick 级写入 < 1ms）→ C++ `MultithreadedTableWriter`
> - **开发效率优先**（因子计算、数据分析）→ Python `PartitionedTableAppender`
> - **C++ 类型系统陷阱**：`DT_TIMESTAMP` 单位是**毫秒**，`DT_NANOTIMESTAMP` 是**纳秒**，与 Python `pd.Timestamp` 的对应需注意精度

