# DolphinDB 参考：快速上手

> 来源：`getstarted/chap1_getstarted.html`

## 概述

本节展示如何快速完成：部署、建数据库、建表、写入数据、导入数据、查询数据等基本操作。

---

## 1. 安装使用

### 1.1 下载与安装

1. 前往 [DolphinDB 官网](https://www.dolphindb.cn/product#downloads) 下载 server 压缩包
2. 将压缩包解压到本地目录

### 1.2 启动 server

**Linux 系统（前台运行）：**
```bash
cd path_to_DolphinDB_server   # 修改为实际路径
chmod +x dolphindb            # 首次启动需修改权限
./dolphindb
```

**Windows 系统：** 双击 `dolphindb.exe` 文件，或右键→打开。

**注意事项：**
- 默认端口号为 `8848`，如需修改请编辑 `dolphindb.cfg`，将 `localSite` 改为如 `localSite=localhost:8890:datanode`
- 如使用商业 license，将 license 文件命名为 `dolphindb.lic` 并替换社区版同名文件

### 1.3 连接客户端

DolphinDB 提供多种客户端：
- **VSCode Extension**（推荐）
- **Web Console**：浏览器输入 `http://localhost:8848`
- **Java GUI**

**Web Console 登录步骤：**
1. 浏览器输入 `http://<ip>:<port>`（本机测试用 `http://localhost:8848`）
2. 点击"去登录"，首次登录账号 `admin`，密码 `123456`

---

## 2. 建库建表

DolphinDB 支持通过 `database`/`table` 函数 **或** SQL 语句建库建表。

**创建按值分区 + 哈希分区的数据库：**
```sql
CREATE DATABASE "dfs://db"
PARTITIONED BY VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 4])
```

**在数据库中创建数据表（分区列为 TradeTime 和 SecurityID）：**
```sql
CREATE TABLE "dfs://db"."tb"(
    TradeTime TIMESTAMP,
    SecurityID SYMBOL,
    TotalVolumeTrade LONG,
    TotalValueTrade DOUBLE
)
PARTITIONED BY TradeTime, SecurityID
```

**查看分区表结构：**
```sql
loadTable("dfs://db", "tb").schema().colDefs
```

---

## 3. 导入数据

将 CSV 文件导入分布式数据表：
```sql
tmp = loadTable("dfs://db", "tb").schema().colDefs
schemaTB = table(tmp.name as name, tmp.typeString as type)
loadTextEx(
    dbHandle=database("dfs://db"),
    tableName="tb",
    partitionColumns=`TradeTime`SecurityID,
    filename="path_to_testdata.csv",
    schema=schemaTB
)
```

查看导入结果：
```sql
select * from loadTable("dfs://db", "tb")
```

---

## 4. 查询数据

**例 1：查询总记录数**
```sql
SELECT count(*) FROM loadTable("dfs://db", "tb")
```

**例 2：条件筛选**
```sql
SELECT * FROM loadTable("dfs://db", "tb") WHERE SecurityID = `A00001
```

**例 3：分组聚合（group by）**
```sql
SELECT avg(TotalVolumeTrade), max(TotalValueTrade)
FROM loadTable("dfs://db", "tb")
GROUP BY SecurityID
```

**例 4：context by（DolphinDB 独有）**—保留原始行数，按组取最后 2 行：
```sql
SELECT * FROM loadTable("dfs://db", "tb") CONTEXT BY SecurityID LIMIT -2
```

**例 5：pivot by（DolphinDB 独有）**—将一列转为矩阵行/列：
```sql
SELECT TotalValueTrade FROM loadTable("dfs://db", "tb")
PIVOT BY TradeTime, SecurityID
```

---

## 5. 学习建议

| 角色 | 重点模块 |
|------|----------|
| 数据分析工程师 | SQL 查询、流数据框架（订阅/引擎级联） |
| 数据库开发工程师 | 数据库核心概念、DolphinDB 编程语言 |
| 数据库运维工程师 | 系统运维、故障排查 |

---

## 6. 技术支持

| 渠道 | 说明 |
|------|------|
| ASK 社区 | https://ask.dolphindb.cn — 1 个工作日内回复 |
| 教程文档 | https://docs.dolphindb.cn |
| 电子邮件 | support@dolphindb.com |
| 知乎 | 智臾科技公司主页（量化金融、物联网、测评报告专栏） |
| 技术交流群 | 客服微信：13306510479 |
