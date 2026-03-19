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
