# forceTriggerOrderBookSnapshot

**URL**: https://docs.dolphindb.cn/zh/funcs/f/forceTriggerOrderBookSnapshot.html

**来源**: DolphinDB 官方文档

---

forceTriggerOrderBookSnapshot
语法
forceTriggerOrderBookSnapshot(engine, [time])
详情
该函数用于强制触发订单簿快照输出，适用于不满足常规触发条件（如时间窗口或数据）但仍需生成快照的场景。调用后，快照会立即生成并输出。
注意，仅在同时满足以下条件时可调用该函数：
createOrderBookSnapshotEngine
设置
triggerType
=
                    "mutual"，且
useSystemTime
= false。
订单簿快照引擎已事先共享。
参数
engine
订单簿引擎名称或引擎对象。
time
可选参数，TIME 类型标量，设置输出快照的时间戳，即输出表中该条记录的 timestamp
                值。若不指定该参数，则时间戳将为当前引擎中最后一条数据的时间戳。
返回值
无
例子
def createInputTable(){
	name = `SecurityID`Date`Time`SourceType`Type`Price`Qty`BSFlag`BuyNo`SellNo`ApplSeqNum`ChannelNo`ReceiveTime
	type = `STRING`DATE`TIME`INT`INT`LONG`LONG`INT`LONG`LONG`LONG`INT`NANOTIMESTAMP
	it = table(1:0, name, type)
	itMap = dict( `codeColumn`timeColumn`typeColumn`priceColumn`qtyColumn`buyOrderColumn`sellOrderColumn`sideColumn`msgTypeColumn`seqColumn`receiveTime, `SecurityID`Time`Type`Price`Qty`BuyNo`SellNo`BSFlag`SourceType`ApplSeqNum`ReceiveTime )
	return it, itMap
}

data = table(take("600010.SH", 3) join take("600020.SH", 3) join take("600010.SH", 2) join take("600020.SH", 3) join take("600010.SH", 2) join take("600020.SH", 1) as SecurityID, take(today(), 14) as date, [14:56:51.000, 14:56:52.000, 14:56:52.000, 14:56:53.000, 14:56:54.000, 14:56:55.000, 14:56:56.000, 14:56:57.000, 14:56:58.000, 14:56:59.000, 14:56:59.000, 14:57:00.000, 14:57:00.000, 14:57:01.000] as time, [0,0,1, 0,0,1,0,0,0,0,1,1,0,0] as SourceType, [2,2,0,2,2,0,2,2,2,2,0,0,2,2] as Type, [229000, 219000, 229000, 229000, 219000, 229000, 229000, 219000, 229000, 219000, 229000,  229000, 219000, 219000]as price, [2000, 1000, 1000, 2000, 1000, 1000, 2000, 1000, 2000, 1000, 1000, 1000, 2000, 2000] as qty, [1,2,2,1,2,2,1,2,1,2,2,2,1,1] as BSFlag, [1,2,1,6,7,6,3,4,8,9,6,1,5,10] as BuyNo, [1,2,2,6,7,7,3,4,8,9,9,4,5,10] as SellNo, 1..14 as ApplSeqNum, take(101, 14) as Channel, take(2023.02.20T04:01:42.499000000, 14) as receiveTime)


try{dropStreamEngine("engine1")} catch(exp){}

orderbookDepth = 10
outlist = lower(genOutputColumnsForOBSnapshotEngine(true, true, (10, true), true, true, true,5, true, true, true)[0])
outputTable = streamTable(genOutputColumnsForOBSnapshotEngine(true, true, (10, true), true, true, true,5, true, true, true)[1])
inputTable = createInputTable()[0]
inputColMap = createInputTable()[1]
dic = dict(["600010.SH", "600020.SH"], [1023, 1265])
maxdic = dict(["600010.SH", "600020.SH"], [3521, 3278])
mindic = dict(["600010.SH", "600020.SH"], [2531, 2210])
engine = createOrderBookSnapshotEngine(name="engine1", exchange="XSHE", orderbookDepth=orderbookDepth, intervalInMilli = 300, date=today(), startTime=09:15:00.000, prevClose=dic, maxPrice = maxdic, minPrice = mindic, priceNullFill = 0.0, precision = 3, orderBySeq = true, skipCrossedMarket = false, dummyTable=inputTable, outputTable=outputTable, inputColMap=inputColMap, outputColMap = outlist, orderBookAsArray=true, triggerType = "mutual", endTime = 15:30:00.000, outputCodeMap = ["600010.SH", "600020.SH"], orderBookDetailDepth = 5)
engine.append!(data)
select * from outputTable
查看订单簿快照结果，600010.SH 和 600020.SH 对应的最后一条快照的时间戳为 2025.05.26 14:57:00.000。其中，600020.SH
                的最后一条数据未能满足触发条件，导致未输出对应的快照。此时，可通过
forceTriggerOrderBookSnapshot
触发
                600020.SH 的最后一条数据输出。
share engine as engine1
forceTriggerOrderBookSnapshot(engine1)
select * from outputTable
执行后，可见输出了时间戳为 2025.05.26 15:00:00.000 的 600010.SH 和 600020.SH 的订单簿快照。