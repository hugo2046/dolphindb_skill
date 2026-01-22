# getPersistenceMeta

**URL**: https://docs.dolphindb.cn/zh/funcs/g/getPersistenceMeta.html

**来源**: DolphinDB 官方文档

---

getPersistenceMeta
语法
getPersistenceMeta(table)
详情
查询启用了持久化的共享流数据表的元数据。
参数
table
是一个表。
返回值
返回一个字典，包含以下 key：
lastLogSeqNum：最新的 Raft 日志的逻辑序列号
sizeInMemory：内存中保留的记录数
asynWrite：持久化是否采用异步的方式
totalSize：流数据表中的总记录数
raftGroup：高可用流数据表所属 Raft 组的 ID。对于普通的流数据表，该值为-1
compress：是否采用压缩存储
memoryOffset：当前内存中数据相对总记录数的偏移量，memoryOffset = totalSize - sizeInMemory
sizeOnDisk：已经持久化到磁盘的记录数
retentionMinutes：日志文件的保留时间，默认值是1440分钟，即一天
persistenceDir：持久化路径
hashValue：对本表做持久化的工作线程标识，当配置项
                        persistenceWorkerNum>1时，hashValue 可能不为0
diskOffset：当前磁盘上数据相对总记录数的偏移量
例子
colName=["time","x"]
colType=["timestamp","int"]
t = streamTable(100:0, colName, colType);
enableTableShareAndPersistence(table=t, tableName=`st, cacheSize=1200000)

go;

for(s in 0:200){
   n=10000
   time=2019.01.01T00:00:00.000+s*n+1..n
   x=rand(10.0, n)
   insert into st values(time, x)
}

getPersistenceMeta(st);

// output
astLogSeqNum->-1
sizeInMemory->800000
asynWrite->true
totalSize->2000000
raftGroup->-1
compress->true
memoryOffset->1200000
retentionMinutes->1440
sizeOnDisk->2000000
persistenceDir->/dolphindb/server/streamPersistDir/st
hashValue->0
diskOffset->0