# tmstdp

**URL**: https://docs.dolphindb.cn/zh/funcs/t/tmstdp.html

**来源**: DolphinDB 官方文档

---

tmstdp
语法
tmstdp(T, X, window)
参数说明和窗口计算规则请参考：
tmFunctions
详情
在给定长度（以时间
T
衡量）的滑动窗口内计算
X
的总体标准差。
返回值
DOUBLE 类型向量。
例子
T = 1 1 3 5 8 15 15 20
X = 5 2 4 1 2 8 9 10
m=table(T as t, X as x)
select *, tmstdp(t, x, 3) from m
t
x
tmstdp_t
1
5
0
1
2
1.5
3
4
1.2472
5
1
1.5
8
2
0
15
8
0
15
9
0.5
20
10
0
T = 2021.01.02 2021.01.02  2021.01.04  2021.01.05 2021.01.07 2021.01.08
X = NULL 4 NULL -1 2 4
m = table(T as t,X as x)
select *, tmstdp(t, x, 3d) from m
t
x
tmstdp_t
2021.01.02
2021.01.02
4
0
2021.01.04
0
2021.01.05
-1
0
2021.01.07
2
1.5
2021.01.08
4
1
select *, tmstdp(t, x, 1w) from m
t
x
tmstdp_t
2021.01.02
2021.01.02
4
0
2021.01.04
0
2021.01.05
-1
2.5
2021.01.07
2
2.0548
2021.01.08
4
2.0463
相关函数：
mstdp
,
stdp