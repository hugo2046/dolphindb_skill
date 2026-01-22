# mvarTopN

**URL**: https://docs.dolphindb.cn/zh/funcs/m/mvarTopN.html

**来源**: DolphinDB 官方文档

---

mvarTopN
语法
mvarTopN(X, S, window, top, [ascending=true],
                    [tiesMethod='oldest'])
参数说明和窗口计算规则请参考：
mTopN
详情
在给定长度（以元素个数衡量）的滑动窗口内，根据
ascending
指定的排序方式将
X
按照
S
进行稳定排序后，取前
top
个元素计算样本方差。
返回值
输入为向量时，返回一个与输入等长的 DOUBLE 类型向量。
输入为矩阵时，返回一个与输入矩阵同维度的矩阵，每列分别计算。
输入为表时，返回一个与输入表结构相同的表。
输入为元组时，返回对应的元组结构。
例子
X = 1..7
S = 0.3 0.5 0.1 0.1 0.5 0.2 0.4
mvarTopN(X, S, 4, 2)
// output: [,0.5,2,0.5,0.5,0.5,2]

X = NULL 1 2 3 4 NULL 5
S = 3 5 1 1 5 2 4
mvarTopN(X, S, 4, 2)
// output: [,,,0.5,0.5,0.5,]

X = matrix(1..5, 6..10)
S = 2022.01.01 2022.02.03 2022.01.23 2022.04.06 2021.12.29
mvarTopN(X, S, 3, 2)
#0
#1
0.5
0.5
2
2
0.5
0.5
2
2
X = matrix(1..5, 6..10)
S = matrix(2022.01.01 2022.02.03 2022.01.23 NULL 2021.12.29,NULL 2022.02.03 2022.01.23 2022.04.06 NULL)
mvarTopN(X, S, 3, 2)
#0
#1
0.5
0.5
2
2
0.5
0.5
2
2
相关函数：
mvar