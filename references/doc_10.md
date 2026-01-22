# decompress

**URL**: https://docs.dolphindb.cn/zh/funcs/d/decompress.html

**来源**: DolphinDB 官方文档

---

decompress
语法
decompress(X)
详情
对一个压缩后的向量进行解压缩。
参数
X
是一个压缩后的向量。
返回值
返回一个向量。
例子
x=1..100000000
y=compress(x, "delta");

y.typestr();
// output: HUGE COMPRESSED VECTOR

z=decompress(y);
z.size();
// output: 100000000
相关函数：
compress