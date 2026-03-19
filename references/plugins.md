# DolphinDB 参考：插件

> 来源：`plugins/plg_intro.html`，各插件文档

## 插件安装方式

```dolphindb
// 在线安装（推荐）
installPlugin("kafka")

// 离线安装（指定 .txt 描述文件路径）
installPlugin("/data/plugins/kafka/PluginKafka.txt")

// 加载插件
loadPlugin("kafka")

// 查看已加载插件
plugin status

// 注意：大多数插件在节点重启后需要重新 loadPlugin
// 可写到启动脚本 poststart.dos 中
```

---

## 常用插件列表

| 插件名 | 说明 |
|--------|------|
| kafka | Apache Kafka 消息队列 |
| mysql | MySQL 数据库连接 |
| odbc | ODBC 通用数据库连接 |
| HBase | Apache HBase 连接 |
| MongoDB | MongoDB 连接 |
| Redis | Redis 连接 |
| AWS S3 | AWS S3 对象存储 |
| parquet | Parquet 文件读写 |
| orc | ORC 文件读写 |
| hdf5 | HDF5 文件读写 |
| mqtt | MQTT 消息协议 |
| zmq | ZeroMQ 消息 |
| Arrow | Apache Arrow 格式 |
| httpClient | HTTP 请求（REST API） |
| email | 邮件发送 |
| ops | 运维工具函数集 |
| formula | 数学公式解析 |
| DolphinDBPlugin | 自定义插件开发框架 |

---

## Kafka 插件

```dolphindb
// 加载
loadPlugin("kafka")

// 生产者（写入 Kafka）
producer = kafka::producer({
    "bootstrap.servers": "kafka-host:9092",
    "client.id": "ddb-producer"
})

// 发送消息
kafka::produce(producer, "my-topic", "key", "value")

// 消费者（从 Kafka 读取）
consumer = kafka::consumer({
    "bootstrap.servers": "kafka-host:9092",
    "group.id": "ddb-consumer",
    "auto.offset.reset": "earliest"
})
kafka::subscribe(consumer, ["my-topic"])

// 拉取消息（批量）
msgs = kafka::poll(consumer, 1000, 10)

// 关闭
kafka::close(producer)
kafka::close(consumer)
```

---

## MySQL 插件

```dolphindb
// 加载
loadPlugin("mysql")

// 连接
conn = mysql::connect("127.0.0.1", 3306, "root", "password", "mydb")

// 查询（返回 Table）
data = mysql::load(conn, "SELECT * FROM trades LIMIT 1000")

// 查询并直接导入 DolphinDB
mysql::loadEx(conn, database("dfs://mydb"), "trades", "TradeTime",
    "SELECT * FROM raw_trades WHERE date >= '2024-01-01'")

// 关闭
mysql::close(conn)
```

---

## ODBC 插件

```dolphindb
loadPlugin("odbc")

// 连接 SQL Server
conn = odbc::connect("Driver={SQL Server};
    Server=192.168.1.100;
    Database=FinanceDB;
    UID=sa;PWD=password")

// 执行查询
data = odbc::query(conn, "SELECT * FROM trades WHERE date='2024-01-01'")

// 执行非查询语句
odbc::execute(conn, "UPDATE trades SET flag=1 WHERE id=100")

// 关闭
odbc::close(conn)
```

---

## HTTP Client 插件（调用 REST API）

```dolphindb
loadPlugin("httpClient")

// GET 请求
resp = httpClient::httpGet(
    "https://api.example.com/data",
    {"Authorization": "Bearer token123"}
)

// POST 请求（JSON）
resp = httpClient::httpPost(
    "https://api.example.com/submit",
    "application/json",
    '{"key": "value"}',
    {"Authorization": "Bearer token123"}
)

// 解析响应
if (resp.responseCode == 200) {
    data = parseJSON(resp.text)
}
```

---

## Parquet 插件

```dolphindb
loadPlugin("parquet")

// 读取 Parquet 文件
data = parquet::loadParquet("/data/trades.parquet")

// 写入 Parquet 文件
parquet::saveParquet(data, "/output/trades.parquet")

// 加载并导入 DolphinDB
parquet::loadParquetEx(
    database("dfs://mydb"),
    "trades",
    "TradeTime",
    "/data/trades.parquet"
)
```

---

## 自定义插件开发（C++）

```cpp
// PluginMyPlugin.h
#include "CoreConcept.h"

extern "C" ConstantSP myFunction(Heap *heap, vector<ConstantSP> &args);

// 注册函数
// PluginMyPlugin.txt:
// myPlugin
// path/to/libMyPlugin.so
// myFunction,myFunction,void,0,0
```
