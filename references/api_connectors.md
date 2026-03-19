# DolphinDB 参考：API & 连接器

> 来源：`api/connapi_intro.html`，各语言 API 文档

## 支持的语言 API

| 语言/工具 | 包名/方式 |
|-----------|-----------|
| Python | `dolphindb` (pip) |
| Java | `dolphindb-jdbc` / DolphinDB Java API |
| C++ | DolphinDB C++ API |
| C# | DolphinDB C# API |
| Go | dolphindb/go-dolphindb |
| JavaScript | DolphinDB JavaScript API |
| JDBC | `com.dolphindb:dolphindb-jdbc` |
| ODBC | DolphinDB ODBC 驱动 |
| R | DolphinDB R API |

---

## Python API

### 安装

```bash
pip install dolphindb
```

### 基础连接

```python
import dolphindb as ddb
import numpy as np
import pandas as pd

# 创建会话
s = ddb.session()
s.connect("localhost", 8848, "admin", "123456")

# 执行脚本
result = s.run("1+1")
print(result)  # 2

# 执行 SQL
df = s.run("select * from loadTable('dfs://trade_db','trades') limit 100")
print(type(df))  # pandas.DataFrame
```

### 上传数据到 DolphinDB

```python
import pandas as pd
import numpy as np
from datetime import datetime

# 创建 DataFrame
df = pd.DataFrame({
    'TradeTime': pd.date_range('2024-01-01', periods=1000, freq='S'),
    'SecurityID': np.random.choice(['AAPL', 'MSFT', 'GOOG'], 1000),
    'Price': np.random.uniform(100, 500, 1000),
    'Volume': np.random.randint(1000, 100000, 1000)
})

# 上传变量
s.upload({'data': df})

# 向分布式表写入
s.run("loadTable('dfs://trade_db','trades').append!(data)")

# 一步式写入（tableAppender）
appender = ddb.tableAppender(
    dbPath="dfs://trade_db",
    tableName="trades",
    ddbSession=s
)
appender.append(df)
```

### 流数据订阅（Python）

```python
import dolphindb as ddb

# 创建会话并订阅
s = ddb.session()
s.connect("localhost", 8848, "admin", "123456")

# 启用流数据
s.enableStreaming(port=8849)

# 定义消息处理函数
def on_message(msg):
    print(msg)

# 订阅流表
s.subscribe(
    host="localhost",
    port=8848,
    handler=on_message,
    tableName="trades",
    actionName="myPythonSub",
    msgAsTable=True,
    batchSize=100,
    throttle=0.5
)

# 取消订阅
s.unsubscribe("localhost", 8848, "trades", "myPythonSub")
```

### 连接池（高并发场景）

```python
# 创建连接池
pool = ddb.SessionPool(
    host="localhost",
    port=8848,
    userid="admin",
    password="123456",
    initialPoolSize=5,
    maxPoolSize=20
)

# 在连接池中运行
result = pool.run("select count(*) from loadTable('dfs://t','t')")
```

---

## Java API

### Maven 依赖

```xml
<dependency>
    <groupId>com.dolphindb</groupId>
    <artifactId>dolphindb-jdbc</artifactId>
    <version>1.30.22.1</version>
</dependency>
```

### 基础连接与查询

```java
import com.xxdb.*;

// 连接
DBConnection conn = new DBConnection();
conn.connect("localhost", 8848, "admin", "123456");

// 执行脚本
BasicInt result = (BasicInt) conn.run("1+1");

// 查询并转为 EntityList
EntityList rows = (EntityList) conn.run(
    "select * from loadTable('dfs://trade_db','trades') limit 100"
);

// 写入数据
// 构造数据
List<String> colNames = Arrays.asList("TradeTime", "SecurityID", "Price");
List<Entity> cols = Arrays.asList(tsVector, symVector, priceVector);
BasicTable bt = new BasicTable(colNames, cols);

// 调用服务端追加
conn.run("trades.append!", Arrays.asList(bt));
```

### BatchTableWriter（高性能批量写入）

```java
import com.xxdb.streaming.client.*;

// 创建 BatchTableWriter（异步批量，推荐高频写入）
BatchTableWriter writer = new BatchTableWriter(
    "localhost", 8848, "admin", "123456",
    true   // enableHighAvailability
);

// 添加表（dbPath + tableName）
writer.addTable("dfs://trade_db", "trades");

// 写入一行（类型需与表结构完全匹配）
writer.insert("dfs://trade_db", "trades",
    Utils.countMilliseconds(2024, 1, 1, 9, 30, 0, 0),  // TIMESTAMP
    "000001",   // SYMBOL
    10.50,      // DOUBLE
    1000        // INT
);

// 关闭（等待所有数据写入完成）
writer.waitForThreadCompletion();
```

### PartitionedTableAppender（多线程并发写入分区表）

```java
import com.xxdb.*;

// 适合大批量历史数据导入（多线程并发写不同分区）
DBConnectionPool pool = new ExclusiveDBConnectionPool(
    "localhost", 8848, "admin", "123456", 
    5,     // 线程数（建议 = 分区数 or CPU核数）
    true,  // loadBalance
    false  // highAvailability
);

PartitionedTableAppender appender = new PartitionedTableAppender(
    "dfs://trade_db",  // dbPath
    "trades",          // tableName
    "TradeDate",       // partitionColumnName（分区列）
    pool
);

// 批量追加（BasicTable 格式）
appender.append(bt);
pool.shutdown();
```

### Java 侧流数据订阅

```java
import com.xxdb.streaming.client.*;

// 创建流订阅客户端
ThreadedClient subscriber = new ThreadedClient("localhost", 8849);

// 消息处理器
MessageHandler handler = new MessageHandler() {
    @Override
    public void doEvent(IMessage msg) {
        // msg 是一行流数据
        System.out.println(msg.getEntity(1).getString()); // 第2列值
    }
};

// 订阅流表
subscriber.subscribe(
    "localhost",  // DolphinDB 节点地址
    8848,
    "trades",     // 流表名
    "javaSub",    // actionName
    handler,
    -1            // offset=-1 表示从新增数据开始
);

// 取消订阅
subscriber.unsubscribe("localhost", 8848, "trades", "javaSub");
```

---

## C++ API

```cpp
#include "DolphinDB.h"

// 连接
DBConnection conn;
conn.connect("localhost", 8848, "admin", "123456");

// 执行脚本
ConstantSP result = conn.run("1+1");
std::cout << result->getInt() << std::endl;

// 查询
TableSP table = (TableSP)conn.run(
    "select * from loadTable('dfs://trade_db','trades') limit 100"
);
std::cout << table->rows() << " rows" << std::endl;

// 关闭
conn.close();
```

---

## JDBC 连接

```java
// JDBC URL 格式
String url = "jdbc:dolphindb://localhost:8848?user=admin&password=123456";

// 加载驱动
Class.forName("com.dolphindb.jdbc.Driver");

// 建立连接
Connection conn = DriverManager.getConnection(url);

// 执行查询
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(
    "SELECT * FROM loadTable('dfs://trade_db','trades') LIMIT 10"
);

while (rs.next()) {
    System.out.println(rs.getString("SecurityID") + " " + rs.getDouble("Price"));
}
```
