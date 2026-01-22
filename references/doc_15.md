# nullIf

**URL**: https://docs.dolphindb.cn/zh/funcs/n/nullIf.html

**来源**: DolphinDB 官方文档

---

nullIf
语法
nullIf(X, Y)
参数
X
和
Y
可以是标量或向量，也可以是返回标量或向量的表达式。
详情
若
X
和
Y
都是标量，比较
X
和
Y
的类型和值是否相同，若相同返回 NULL 值，否则返回 X。
若
X
和
Y
是等长的向量，则将
X
和
Y
的元素一一进行上述比较。
若
X
和
Y
一个是标量一个是向量，则将标量和向量中的元素一一进行上述比较。
例子
t = table(`APPL`IBM`AMZN`IBM`APPL`AMZN as sym, 10.1 11.2 11.3 12 10.6 10.8 as val)
select nullIf(sym, `AMZN) from t
输出返回：
nullIf_sym
APPL
IBM
IBM
APPL
select * from t where nullIf(sym, `AMZN)!=NULL
输出返回：
sym
val
APPL
10.1
IBM
11.2
IBM
12
APPL
10.6