# setIndexedMatrix!

**URL**: https://docs.dolphindb.cn/zh/funcs/s/setIndexedMatrix_.html

**来源**: DolphinDB 官方文档

---

setIndexedMatrix!
语法
setIndexedMatrix!(X, [on=true])
详情
将矩阵的行与列标签设为索引。
参数
X
是一个有行与列标签的矩阵。行与列标签均须严格递增，无重复项。
on
布尔值，表示普通矩阵和索引矩阵之间的转换。默认为 true，表示将普通矩阵转换成索引矩阵；若为 false，表示将索引矩阵转换为普通矩阵。
返回值
返回一个矩阵或索引矩阵。
例子
m=matrix(1..5, 11..15)
m.rename!(2020.01.01..2020.01.05, `A`B)
m.setIndexedMatrix!();
A
B
2020.01.01
1
11
2020.01.02
2
12
2020.01.03
3
13
2020.01.04
4
14
2020.01.05
5
15