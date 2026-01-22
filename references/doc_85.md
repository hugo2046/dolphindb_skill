# pnodeRun

**URL**: https://docs.dolphindb.cn/zh/funcs/p/pnodeRun.html

**来源**: DolphinDB 官方文档

---

pnodeRun
语法
pnodeRun(function, [nodes],
                [addNodeAlias=true])
详情
在集群指定节点或所有数据节点/计算节点上并行调用本地函数，然后合并结果。
当指定
nodes
参数时，在指定的节点上调用本地函数。
当未指定
nodes
参数时：若在指定了计算组的计算节点上执行该函数时，仅会在同组内的所有计算节点上调用本地函数；否则，在集群中所有的数据节点、未指定计算组的计算节点上调用本地函数。
参数
function
调用的本地函数（不能使用引号）。它可以是一个没有定义参数的函数，或者是没有参数的封装了初始函数和参数的部分应用。它可以是内置函数或用户自定义函数。
nodes
可选参数，字符串标量或向量，表示节点的别名。
addNodeAlias
是否在结果中加入节点的别名，默认值是
                true。如果返回的结果已经包含节点的别名，可设置为false。
返回值
若被调用函数返回标量、字典、表， 返回一个表。
若被调用函数返回向量， 返回一个矩阵。
若被调用函数返回字典， 返回一个表。
若被调用函数为命令（无返回值）， 返回空值。
其他情况，返回一个字典。
例子
例1. 函数 getChunksMeta 不指定参数。
pnodeRun(getChunksMeta,,false);
site
chunkId
path
dfsPath
type
flag
size
version
state
versionList
local8848
bd13090e-7177-01a7-4ac4-840e1b977dcf
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190605/GOOG
/compo/20190605/GOOG
1
0
0
1
0
cid : 40,pt2=>40:6729; #
local8848
b4935730-6372-b2a1-4f24-6c323037e576
e:data/CHUNKS/compo/20190605/AAPL
/compo/20190605/AAPL
1
0
0
1
0
cid : 40,pt2=>40:6613; #
local8848
f8ee72c9-dad3-f49e-430e-5ddb3c61ae18
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190604/MSFT
/compo/20190604/MSFT
1
0
0
1
0
cid : 40,pt2=>40:6664; #
local8848
08e26b5a-dfac-799f-4979-0dd3902eae6e
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190604/GOOG
/compo/20190604/GOOG
1
0
0
1
0
cid : 40,pt2=>40:6635; #
local8848
f9e53a3d-af3e-018d-4bfa-a2b4980f3561
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190604/AAPL
/compo/20190604/AAPL
1
0
0
1
0
cid : 40,pt2=>40:6783; #
local8848
417e49e9-5c61-cf9e-4b21-4b35f8e57273
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190601/MSFT
/compo/20190601/MSFT
1
0
0
1
0
cid : 40,pt2=>40:6602; #
local8848
3ee64942-1d72-bea7-4bc1-f720132d9288
D:130DolphinDB_Win64_Vserverlocal8848storage/CHUNKS/compo/20190602/AAPL
/compo/20190602/AAPL
1
0
0
1
0
cid : 40,pt2=>40:6749; #
例2. 在下例中，函数 sum 和参数1..10被封装成了部分应用 sum{1..10}。
pnodeRun(sum{1..10}, `nodeA`nodeB);
Node
Value
DFS_NODE2
55
DFS_NODE3
55
例3.
pnodeRun
对于集群管理非常方便。例如，在一个集群中有4个节点："DFS_NODE1", "DFS_NODE2", "DFS_NODE3" 和
                "DFS_NODE4"。在每个节点上执行以下脚本：
def jobDemo(n){
  s = 0
  for (x in 1 : n) {
      s += sum(sin rand(1.0, 100000000)-0.5)
      print("iteration " + x + " " + s)
  }
  return s
};

submitJob("jobDemo1","job demo", jobDemo, 10);
submitJob("jobDemo2","job demo", jobDemo, 10);
submitJob("jobDemo3","job demo", jobDemo, 10);
查看集群中每个节点最近完成的两个批处理作业的状态：
pnodeRun(getRecentJobs{2});
Node
UserID
JobID
JobDesc
ReceivedTime
StartTime
EndTime
ErrorMsg
DFS_NODE1
root
jobDemo2
job demo
2017.11.16T13:04:38.841
2017.11.16T13:04:38.841
2017.11.16T13:04:51.660
DFS_NODE1
root
jobDemo3
job demo
2017.11.16T13:04:38.841
2017.11.16T13:04:38.843
2017.11.16T13:04:51.447
DFS_NODE2
root
jobDemo2
job demo
2017.11.16T13:04:56.431
2017.11.16T13:04:56.432
2017.11.16T13:05:11.992
DFS_NODE2
root
jobDemo3
job demo
2017.11.16T13:04:56.432
2017.11.16T13:04:56.434
2017.11.16T13:05:11.670
DFS_NODE3
root
jobDemo2
job demo
2017.11.16T13:05:08.418
2017.11.16T13:05:08.419
2017.11.16T13:05:29.176
DFS_NODE3
root
jobDemo3
job demo
2017.11.16T13:05:08.419
2017.11.16T13:05:08.421
2017.11.16T13:05:29.435
DFS_NODE4
root
jobDemo2
job demo
2017.11.16T13:05:16.324
2017.11.16T13:05:16.325
2017.11.16T13:05:34.729
DFS_NODE4
root
jobDemo3
job demo
2017.11.16T13:05:16.325
2017.11.16T13:05:16.328
2017.11.16T13:05:34.716
pnodeRun(getRecentJobs{2}, `DFS_NODE3`DFS_NODE4);
Node
UserID
JobID
JobDesc
ReceivedTime
StartTime
EndTime
ErrorMsg
DFS_NODE3
root
jobDemo2
job demo
2017.11.16T13:05:08.418
2017.11.16T13:05:08.419
2017.11.16T13:05:29.176
DFS_NODE3
root
jobDemo3
job demo
2017.11.16T13:05:08.419
2017.11.16T13:05:08.421
2017.11.16T13:05:29.435
DFS_NODE4
root
jobDemo2
job demo
2017.11.16T13:05:16.324
2017.11.16T13:05:16.325
2017.11.16T13:05:34.729
DFS_NODE4
root
jobDemo3
job demo
2017.11.16T13:05:16.325
2017.11.16T13:05:16.328
2017.11.16T13:05:34.716
pnodeRun
合并多个节点的结果时遵循以下规则：
(1) 如果 function 返回一个标量：
返回一个表，它具有两列：节点别名和函数结果。
紧接上面的例子：
pnodeRun(getJobReturn{`jobDemo1});
Node
Value
DFS_NODE3
2,123.5508
DFS_NODE2
(42,883.5404)
DFS_NODE1
3,337.4107
DFS_NODE4
(2,267.3681)
(2) 如果 function 返回一个向量：
返回一个矩阵。矩阵中的每一列是函数在节点上返回的结果。矩阵的列标签是节点。
(3) 如果 function 返回键-值形式的字典：
返回一个表，每行代表函数在一个节点上的结果。
(4) 如果 function 返回一个表：
返回一个表，它是多个节点上的表的合并。
(5) 如果 function 是一个命令（该命令不返回任何内容）：
不返回任何内容。
(6) 对于其他情况：
返回一个字典。键是节点别名，值是函数的返回内容。