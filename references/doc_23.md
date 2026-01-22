# insert into

**URL**: https://docs.dolphindb.cn/zh/progr/sql/insertInto.html

**来源**: DolphinDB 官方文档

---

insert into
用于使用
VALUES
子句向表插入数据。
注：
若要向分布式表中插入数据，请先调整配置项 enableInsertStatementForDFSTable。
语法
insert into
 
  table_name1 (colName1 [, colName2, ...])
 
  values (X [, Y, ...]) | select col_name(s) from table_name2
其中， colName 指定目标表中的列名，可以有以下三种方式：
不加引号的列名
colName
双引号括起来的列名
"colName"
前加下划线的双引号列名
_"colName"
例子
t=table(`XOM`GS`FB as ticker, 100 80 120 as volume);
t;
ticker
volume
XOM
100
GS
80
FB
120
insert into t values(`GOOG, 200);
t;
ticker
volume
XOM
100
GS
80
FB
120
GOOG
200
insert into t values(`AMZN`NFLX, 300 250);
t;
ticker
volume
XOM
100
GS
80
FB
120
GOOG
200
AMZN
300
NFLX
250
insert into t values(('AMD','NVDA'), (60 400));
t;
ticker
volume
XOM
100
GS
80
FB
120
GOOG
200
AMZN
300
NFLX
250
AMD
60
NVDA
400
上例还有另一种写法，即按照 SQL 标准写法，直接向表 t 传入多行数据。可得结果一致。
insert into t values ('AMD', 60), ('NVDA', 400);
t;
ticker
volume
XOM
100
GS
80
FB
120
GOOG
200
AMZN
300
NFLX
250
AMD
60
NVDA
400
只往部分列插入新的记录：
insert into t(ticker, volume) values(`UBER`LYFT, 0 0);
t;
ticker
price
volume
XOM
98.5
100
GS
12.3
80
FB
40.6
120
GOOG
100.6
200
AMZN
120
300
NFLX
56.6
250
AMD
78.6
60
NVDA
33.1
400
UBER
0
LYFT
0