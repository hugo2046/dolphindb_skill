# DolphinDB 参考：流计算实战教程



---

> 来源: `streaming_tutorial.html`


# 流数据功能应用

实时流处理是指将业务系统产生的持续增长的动态数据进行实时的收集、清洗、统计、入库，并对结果进行实时的展示。在金融交易、物联网、互联网/移动互联网等应用场景中，复杂的业务需求对大数据处理的实时性提出了极高的要求。面向静态数据表的传统计算引擎无法胜任流数据领域的分析和计算任务。
DolphinDB内置的流数据框架支持流数据的发布、订阅、预处理、实时内存计算、复杂指标的滚动窗口计算、实时关联、异常数据检测等，是一个运行高效，使用便捷的流数据处理框架。
与其它流数据系统相比，DolphinDB流数据处理系统的优点在于：
- 吞吐量大，低延迟，高可用。
- 与DolphinDB时序数据库无缝集成，提供一站式解决方案。
- 天然具备流数据表对偶性，支持使用SQL语句进行数据注入和查询分析。
DolphinDB流数据处理系统提供了多种方便的功能，例如：
- 内置流数据时间序列、横截面、异常检测、响应式状态、连接引擎
- 高频数据回放
- 流数据过滤

## 1. 流程图及相关概念

DolphinDB流数据模块采用发布-订阅-消费的模式。流数据首先注入流数据表中，通过流数据表来发布数据，数据节点或者第三方的应用可以通过DolphinDB脚本或API来订阅及消费流数据。
上图展示了DolphinDB的流数据处理框架。把实时数据注入到发布节点流数据表后，发布的数据可同时供多方订阅消费：
- 可由数据仓库订阅并保存，作为分析系统与报表系统的数据源。
- 可由流数据计算引擎订阅，进行计算，并将结果输出到流数据表。计算结果既可以由Grafana等平台进行实时展示，也可以作为数据源再次发布，供二次订阅做事件处理。
- 可由API订阅，例如第三方的Java应用程序可以通过Java API订阅流数据进行业务操作。

### 1.1. 流数据表

流数据表是一种特殊的内存表，用以存储及发布流数据。与普通内存表不同，流数据表支持同时读写，且只能添加记录，不可修改或删除记录。数据源发布一条消息等价于向流数据表插入一条记录。与普通内存表相同，可使用SQL语句对流数据表进行查询和分析。

### 1.2. 发布与订阅

采用经典的发布订阅模式。每当有新的流数据注入负责发布消息的流数据表时，会通知所有的订阅方处理新的流数据。数据节点使用 subscribeTable 函数来订阅流数据。

### 1.3. 流数据计算引擎

流数据计算引擎是专门用于处理流数据实时计算和分析的模块。DolphinDB提供 createTimeSeriesEngine , createDailyTimeSeriesEngine , createSessionWindowEngine , createAnomalyDetectionEngine , createReactiveStateEngine , createCrossSectionalEngine , createAsofJoinEngine , createEquiJoinEngine , createWindowJoinEngine , createLookupJoinEngine 等函数创建流数据计算引擎对流数据进行实时计算，并将计算结果持续输出到指定的数据表中。
- 注：自 1.30.21/2.00.9 版本起， createEqualJoinEngine 更名为 createEquiJoinEngine ，原函数名可继续使用。*

## 2. 核心功能

要开启支持流数据功能的模块，必须对发布节点指定 maxPubConnections 配置参数，并对订阅节点指定 subPort 配置参数。以下为所有流数据相关配置参数。
- maxPubConnections: 发布节点可以连接的订阅节点数量上限，默认值为0。只有指定maxPubConnections为正整数后，该节点才可作为发布节点。
- persistenceDir: 保存发布消息的流数据表的文件夹路径。若需要保存流数据表，必须指定该参数。所有生产环境中都强烈推荐设定此参数。若不设定此参数，随着消息的积累，内存会最终耗尽。
- persistenceWorkerNum: 负责以异步模式保存流数据表的工作线程数。默认值为0。
- maxPersistenceQueueDepth: 以异步模式保存流数据表时消息队列的最大深度（记录数量）。默认值为10,000,000。
- maxMsgNumPerBlock: 发布消息时，每个消息块中最多可容纳的记录数量。默认值为1024。
- maxPubQueueDepthPerSite: 发布节点消息队列的最大深度（记录数量）。默认值为10,000,000。
- subPort: 订阅线程监听的端口号，默认值为0。只有指定该参数后，该节点才可作为订阅节点。
- subExecutors: 订阅节点中消息处理线程的数量。默认值为0，表示解析消息线程也处理消息。
- maxSubConnections: 该订阅节点可以连接的的发布节点数量上限。默认值为64。
- subExecutorPooling: 表示执行流计算的线程是否处于pooling模式的布尔值。默认值是false。
- maxSubQueueDepth: 订阅节点消息队列的最大深度（记录数量）。默认值为10,000,000。

### 2.1. 流数据发布

使用 streamTable 函数定义一个流数据表。实时数据写入该表后，向所有订阅端发布。由于通常有多个会话中的多个订阅端订阅同一个发布端，所以必须使用 share 函数将流数据表在所有会话中共享后才可发布流数据。未被共享的流数据表无法发布流数据。
定义并共享流数据表pubTable：

```
share streamTable(10000:0,`timestamp`temperature, [TIMESTAMP,DOUBLE]) as pubTable
```

streamTable 函数创建的流数据表是可以包含重复记录的。如果要创建包含主键的流数据表，可以使用 keyedStreamTable 函数。包含主键的流数据表中，一旦写入某键值的数据，后续相同键值的数据不会写入此流数据表，将被丢弃。

```
share keyedStreamTable(`timestamp, 10000:0,`timestamp`temperature, [TIMESTAMP,DOUBLE]) as pubTable
```

可以用undef函数或者dropStreamTable删除上述语句创建的共享流数据表pubTable：

```
undef(`pubTable, SHARED)
dropStreamTable(`pubTable)
```

undef函数能够将变量或者函数定义从内存中释放。但是，若要删除2.6节的持久化流数据表，则必须使用dropStreamTable函数。此外，用户需要在取消所有订阅后才能删除相应的流数据表，取消订阅请参考 取消订阅 。

### 2.2. 流数据订阅

订阅流数据通过 subscribeTable 函数来实现。

```
subscribeTable([server],tableName,[actionName],[offset=-1],handler,[msgAsTable=false],[batchSize=0],[throttle=1],[hash=-1],[reconnect=false],[filter],[persistOffset=false],[timeTrigger=false],[handlerNeedMsgId=false],[raftGroup],[userId=""],[password=""])
```

- 只有tableName和handler两个参数是必需的，其它所有参数均为可选参数。
- server 为字符串，表示流数据所在服务器的别名或远程连接handle。如果未指定或者为空字符串，表示流数据所在服务器是本地实例。
实际情况中，发布者与订阅者所在节点的关系有以下三种可能。这三种情况下的server参数设置分别为：
- 发布者与订阅者是同一节点，均为本地实例：参数server不设置或使用空字符串。 subscribeTable(tableName="pubTable", actionName="act1", offset=0, handler=subTable, msgAsTable=true)
- 发布者与订阅者是同一集群内的不同节点：参数server使用发布节点别名。 subscribeTable(server="NODE2", tableName="pubTable", actionName="act1", offset=0, handler=subTable, msgAsTable=true)

### 2.3. 断线重连

DolphinDB的流数据订阅提供了自动重连的功能。如果要启用自动重连，发布端必须对流数据持久化。当reconnect参数设为true时，订阅端会记录流数据的offset，连接中断时订阅端会从offset开始重新订阅。如果订阅端关闭或者发布端没有对流数据持久化，订阅端无法自动重连。

### 2.4. 发布端数据过滤

发布端可以过滤数据，只发布符合条件的数据。使用 setStreamTableFilterColumn 指定流数据表的过滤列（目前仅支持对一个列进行过滤），过滤列的值在filter中的数据会发布到订阅端，不在filter指定值中的数据不会发布。有关filter参数的介绍请见2.2小节。
下例中，值过滤的filter值是一个向量。发布端上的流数据表trades只发布symbol为IBM或GOOG的数据：

```
share streamTable(10000:0,`time`symbol`price, [TIMESTAMP,SYMBOL,INT]) as trades
setStreamTableFilterColumn(trades, `symbol)
trades_1=table(10000:0,`time`symbol`price, [TIMESTAMP,SYMBOL,INT])

filter=symbol(`IBM`GOOG)

subscribeTable(tableName="trades", actionName="trades_1", handler=append!{trades_1}, msgAsTable=true, filter=filter)
```

范围过滤的filter值是一个数据对。发布端上的流数据表trades只发布price大于等于1且小于100的数据：
哈希过滤的filter值是一个元组。发布端上的流数据表trades对于symbol列使用哈希函数分为10个bucket，bucket索引从0开始，只发布索引大于等于1且小于5的数据：

### 2.5. 取消订阅

每一次订阅都由一个订阅主题topic作为唯一标识。如果订阅时topic已存在，那么会订阅失败，需要通过 unsubscribeTable 函数取消订阅才能再次订阅。

```
unsubscribeTable(tableName="pubTable", actionName="act1")
```


```
unsubscribeTable(server="NODE_1", tableName="pubTable", actionName="act1")
```

取消订阅一个本地表，但保留offset，以便下次从这个offset继续订阅：

```
unsubscribeTable(tableName="pubTable", actionName="act1", removeOffset=false)
```

从节点的内存中删除给定topic的offset：

```
removeTopicOffset(topic)
```


### 2.6. 流数据持久化

默认情况下，流数据表把所有数据保存在内存中。基于以下三点考量，可将流数据持久化到磁盘。
- 流数据的备份和恢复。当节点出现异常重启时，持久化的数据会在重启时自动载入到流数据表。
- 避免内存不足。
- 可以从任意位置开始重新订阅数据。
可事先设定一个界限值。若流数据表的行数达到设定的界限值，前面一半的记录行会持久化到磁盘。持久化的数据支持重订阅，当订阅指定offset时，offset的计算包含持久化的数据。
要持久化流数据表，在发布节点首先需要设置持久化路径参数persistenceDir:

```
persistenceDir = /data/streamCache
```

然后执行 enableTableShareAndPersistence 函数。下面的示例将pubTable共享为sharedPubTable，并把sharedPubTable持久化到磁盘。其中参数cacheSize=1000000，asynWrite与compress默认值均为true，表示当流数据表数据量达到100万行时启用持久化，将其中50%的数据采用异步方式压缩保存到磁盘。

```
pubTable=streamTable(10000:0,`timestamp`temperature, [TIMESTAMP,DOUBLE])
enableTableShareAndPersistence(table=pubTable, tableName=`sharedPubTable, cacheSize=1000000, preCache=500000)
```

若执行 enableTableShareAndPersistence 时，磁盘上已经存在sharedPubTable表的持久化数据，那么系统会加载最新的preCache=500000行记录到内存中。
对于持久化是否启用异步，需要在持久化数据一致性和性能之间作权衡。当流数据的一致性要求较高时，可以使用同步方式，这样可以保证持久化完成以后，数据才会进入发布队列；若对实时性要求较高，不希望磁盘IO影响到流数据的实时性，则可启用异步方式。只有启用异步方式时，持久化工作线程数persistenceWorkerNum配置项才会起作用。若有多个发布表需要持久化，增加persistenceWorkerNum的配置值可以提升异步保存的效率。
当不需要保存在磁盘上的流数据时，通过 clearTablePersistence 函数可以删除持久化数据：

```
clearTablePersistence(pubTable)
```

关闭持久化，可以使用 disableTablePersistence 函数：

```
disableTablePersistence(pubTable)
```

使用 getPersistenceMeta 函数获取流数据表的持久化细节情况：

```
getPersistenceMeta(pubTable);
```

输出的结果是一个字典，有以下内容：

```
//内存中的数据记录数
sizeInMemory->0
//启用异步持久化
asynWrite->true
//流数据表总记录数
totalSize->0
//启用压缩存储
compress->true
//当前内存中数据相对总记录数的偏移量，在持久化运行过程中遵循公式 memoryOffset = totalSize - sizeInMemory
memoryOffset->0
//已经持久化到磁盘的数据记录数
sizeOnDisk->0
//日志文件的保留时间，默认值是1440分钟，即一天。
retentionMinutes->1440
//持久化路径
persistenceDir->/hdd/persistencePath/pubTable
//hashValue是对本表做持久化的工作线程标识。
hashValue->0
//磁盘上第一条数据相对总记录数的偏移量。例如，若diskOffset=10000，表示目前磁盘上的持久化流数据从第10000条记录开始。
diskOffset->0
```

调用dropStreamTable函数删除持久化流数据表，内存中和磁盘上的流数据均会被清除：

```
dropStreamTable(`pubTable);
```


## 3. 数据回放

DolphinDB提供了 replay 函数，可以将历史数据按照时间顺序导入流数据表中。具体教程请参考 流数据回放教程 。

## 4. 流数据计算引擎

DolphinDB提供 createTimeSeriesEngine , createDailyTimeSeriesEngine , createSessionWindowEngine , createAnomalyDetectionEngine , createReactiveStateEngine , createCrossSectionalEngine , createAsofJoinEngine , createEquiJoinEngine , createWindowJoinEngine , createLookupJoinEngine 等函数创建流数据计算引擎对流数据进行实时计算。

```
rse = createReactiveStateEngine(name="reactiveDemo", metrics =<cumsum(price)>, dummyTable=tickStream, outputTable=result, keyColumn="sym", snapshotDir= "/home/data/snapshot", snapshotIntervalInMsgCount=20000)
```

调用dropStreamEngine函数释放流数据引擎：

```
dropStreamEngine("reactiveDemo")
```

此外，DolphinDB流计算引擎还包括流水线处理、并行处理、快照机制等重要特性。
注意 ：DolphinDB 1.30.21/2.00.9 及之前的版本不支持对流计算引擎的并发访问（例如，两个输入表并发写入同一引擎）。 如需使用此功能，要求 server 版本号必须高于 1.30.21/2.00.9，且必须使用 share 语句/函数共享引擎，例如 share(engine, name) 或 share engien as name （要求引擎共享后的名称和引擎名称相同）。

### 4.1. 流水线处理

DolphinDB内置的流计算引擎均实现了数据表（table）的接口，因此多个引擎流水线处理变得异常简单，只要将后一个引擎作为前一个引擎的输出即可。引入流水线处理，可以解决更为复杂的因子计算问题。譬如，因子计算经常需要使用面板数据，完成时间序列和横截面两个维度的计算，只要把响应式状态引擎和横截面两个引擎串联处理即可完成。
下面的例子是World Quant 101个Alpha因子中的1号因子公式的流数据实现。rank函数是一个横截面操作。rank的参数部分用响应式状态引擎实现。rank函数本身用横截面引擎实现。横截面引擎作为状态引擎的输出。

```
Alpha#001公式：rank(Ts_ArgMax(SignedPower((returns<0?stddev(returns,20):close), 2), 5))-0.5

//创建横截面引擎，计算每个股票的rank
dummy = table(1:0, `sym`time`maxIndex, [SYMBOL, TIMESTAMP, DOUBLE])
resultTable = streamTable(10000:0, `time`sym`factor1, [TIMESTAMP, SYMBOL, DOUBLE])
ccsRank = createCrossSectionalAggregator(name="alpha1CCS", metrics=<[sym, rank(maxIndex, percent=true) - 0.5]>,  dummyTable=dummy, outputTable=resultTable,  keyColumn=`sym, triggeringPattern='keyCount', triggeringInterval=3000, timeColumn=`time, useSystemTime=false)

@state
def wqAlpha1TS(close){
    ret = ratios(close) - 1
    v = iif(ret < 0, mstd(ret, 20), close)
    return mimax(signum(v)*v*v, 5)
}

//创建响应式状态引擎，输出到前面的横截面引擎ccsRank
input = table(1:0, `sym`time`close, [SYMBOL, TIMESTAMP, DOUBLE])
rse = createReactiveStateEngine(name="alpha1", metrics=<[time, wqAlpha1TS(close)]>, dummyTable=input, outputTable=ccsRank, keyColumn="sym")
```

流水线处理（也称为引擎多级级联）和多个流数据表的级联处理有很大的区别。两者可以完成相同的任务，但是效率上有很大的区别。后者涉及多个流数据表与多次订阅。前者实际上只有一次订阅，所有的计算均在一个线程中依次顺序完成，因而有更好的性能。
上面的例子是由用户来区分哪一部分是横截面操作，哪一部分是时间序列操作以实现多个引擎的流水线。在1.30.16/2.00.4及之后的版本中，新增函数 streamEngineParser ，支持将metrics自动分解成多个内置流计算引擎的流水线。在 streamEngineParser 中以行函数（rowRank，rowSum等）表示横截面操作的语义，以rolling函数表示时间序列操作，从而系统能够自动识别一个因子中的横截面操作和时间序列操作，进一步自动构建引擎流水线。因此，上述因子可以用 streamEngineParser 更简洁的实现，metrics几乎等同于因子的数学公式表达，而不需要考虑不同类型引擎的选择：

```
@state
def wqAlpha1TS(close){
    ret = ratios(close) - 1
    v = iif(ret < 0, mstd(ret, 20), close)
    return mimax(signum(v)*v*v, 5)
}

//构建计算因子
metrics=<[sym, rowRank(wqAlpha1TS(close), percent=true)- 0.5]>

streamEngine=streamEngineParser(name=`alpha1_parser, metrics=metrics, dummyTable=input, outputTable=resultTable, keyColumn=`sym, timeColumn=`time, triggeringPattern='keyCount', triggeringInterval=3000)
```


### 4.2. 并行处理

当需要处理大量消息时，可在DolphinDB消息订阅函数 subscribeTable 中指定可选参数filter与hash，让多个订阅客户端并行处理消息。
下面是响应式状态引擎并行计算因子的例子。假设配置参数subExecutors=4，创建4个状态引擎，每个状态引擎根据流表的股票代码的哈希值来订阅不同股票的数据，并且指定不同的订阅线程来处理，最终将结果输出到同一个输出表中。

```
share streamTable(1:0, `sym`price, [STRING,DOUBLE]) as tickStream
setStreamTableFilterColumn(tickStream, `sym)
share streamTable(1000:0, `sym`factor1, [STRING,DOUBLE]) as resultStream

for(i in 0..3){
    rse = createReactiveStateEngine(name="reactiveDemo"+string(i), metrics =<cumsum(price)>, dummyTable=tickStream, outputTable=resultStream, keyColumn="sym")
    subscribeTable(tableName=`tickStream, actionName="sub"+string(i), handler=tableInsert{rse}, msgAsTable = true, hash = i, filter = (4,i))
}
```

需要注意的是，如果多个状态引擎是同一个输出表，该输出表必须是一个共享表。没有共享的表不是线程安全的，并行写入可能会导致系统崩溃。

### 4.3. 快照机制

为了满足生产环境业务持续性的需要，DolphinDB内置的流式计算引擎除连接引擎外均支持快照（snapshot）输出。
以响应式状态引擎为例，该引擎的快照包括已处理的最后一条消息的ID以及引擎当前的状态（中间计算结果）。当系统出现异常，重新初始化状态引擎时，可恢复到最后一个快照的状态，并且从已处理的消息的下一条开始订阅。

```
share streamTable(1:0, `sym`price, [STRING,DOUBLE]) as tickStream
result = table(1000:0, `sym`factor1, [STRING,DOUBLE])
rse = createReactiveStateEngine(name="reactiveDemo", metrics =<cumsum(price)>, dummyTable=tickStream, outputTable=result, keyColumn="sym", snapshotDir= "/home/data/snapshot", snapshotIntervalInMsgCount=20000)
msgId = getSnapshotMsgId(rse)
if(msgId >= 0) msgId += 1
subscribeTable(tableName=`tickStream, actionName="factors", offset=msgId, handler=appendMsg{rse}, handlerNeedMsgId=true)
```

响应式状态引擎要启用快照机制，创建时需要指定两个额外的参数snapshotDir和snapshotIntervalInMsgCount。snapshotDir用于指定存储快照的目录。snapshotIntervalInMsgCount指定处理多少条消息后产生一个快照。引擎初始化时，系统会检查快照目录下是否存在一个以引擎名称命名，后缀为snapshot的文件。以上面的代码为例，如果存在文件/home/data/snapshot/reactiveDemo.snapshot，加载这个快照。函数getSnapshotMsgId可以获取最近一个快照对应的msgId。如果不存在快照，返回-1。
状态引擎要启用快照机制，调用 subscribeTable 函数也需相应的修改：
- 首先必须指定消息的offset。
- 其次，handler必须使用appendMsg函数。appendMsg函数接受两个参数，msgBody和msgId。
- 再次，参数handlerNeedMsgId必须指定为true。
上例为普通订阅在宕机后重新提交订阅并从快照处恢复流数据处理。在5高可用章节中，通过高可用流订阅自定恢复订阅时将不再需要通过getSnapshotMsgId函数获取msgId来指定offset，也不需要使用appendMsg函数。

## 5. 高可用

为满足流数据服务不中断的需求，DolphinDB采用了基于Raft协议的高可用多副本架构，以提供流数据的高可用功能。具体教程请参考 流数据高可用教程 。

## 6. 流数据API

流数据的消费者可能是DolphinDB内置的计算引擎，也可能是第三方的消息队列或者第三方程序。DolphinDB提供了streaming API供第三方程序来订阅流数据。当有新数据注入时，API的订阅者能够及时接收到通知，这使得DolphinDB的流数据框架可与第三方的应用进行深入的整合。DolphinDB的API（Java, Python, C++, C#）提供了接口来订阅流数据。

### 6.1. Python API

Python API提供流数据订阅的相关方法，用于订阅DolphinDB服务端的数据。

#### 6.1.1. Python客户端流数据订阅示例

下面简单介绍一下Python API提供的流数据订阅的相关方法与使用示例。
- 指定客户端的订阅端口号 使用Python API提供的 enableStreaming 函数启用流数据功能： import dolphindb as ddb conn = ddb.session() conn.enableStreaming(8000)
- 取消订阅 conn.unsubscribe("192.168.1.103", 8941,"trades","sub_trades")

#### 6.1.2. DolphinDB服务端流数据订阅示例

DolphinDB可以订阅来自Python客户端的流数据。下面的例子中，我们在Python客户端订阅第三方数据到多个DataFrame中，通过DolphinDB的流数据订阅功能将多个表中的数据写入到分布式表中。
首先，在DolphinDB服务端执行以下脚本，创建数据库和表：

```
login('admin','123456')

// 定义表结构
n=20000000
colNames =`Code`Date`DiffAskVol`DiffAskVolSum`DiffBidVol`DiffBidVolSum`FirstDerivedAskPrice`FirstDerivedAskVolume`FirstDerivedBidPrice`FirstDerivedBidVolume
colTypes = [SYMBOL,DATE,INT,INT,INT,INT,FLOAT,INT,FLOAT,INT]

// 创建数据库与分布式表
dbPath= "dfs://ticks"
if(existsDatabase(dbPath))
   dropDatabase(dbPath)
db=database(dbPath,VALUE, 2000.01.01..2030.12.31)
dfsTB=db.createPartitionedTable(table(n:0, colNames, colTypes),`tick,`Date)
```

下面，我们将定义两个流数据表 mem_stream_d 和 mem_stream_f ，客户端往流数据表写入数据，由服务端订阅数据。

```
// 定义mem_tb_d表,并开启流数据持久化，将共享表命名为mem_stream_d
mem_tb_d=streamTable(n:0, colNames, colTypes)
enableTableShareAndPersistence(mem_tb_d,'mem_stream_d',false,true,n)

// 定义mem_tb_f表,并开启流数据持久化，将共享表命名为mem_stream_f
mem_tb_f=streamTable(n:0,colNames, colTypes)
enableTableShareAndPersistence(mem_tb_f,'mem_stream_f',false,true,n)
```

注意 ：由于表的分区字段是按照日期进行分区，而客户端往 mem_stream_d 和 mem_stream_f 表中写的数据会有日期上的重叠，若直接由分布式表 tick 同时订阅这两个表的数据，就会造成这两个表同时往同一个日期分区写数据，导致写入失败。因此，我们需要定义另一个流表 ticks_stream 来汇集 mem_stream_d 和 mem_stream_f 表的数据，最后串行写入 tick 分布式表。

```
// 定义ftb表，并开启流数据持久化，将共享表命名为ticks_stream
ftb=streamTable(n:0, colNames, colTypes)
enableTableShareAndPersistence(ftb,'ticks_stream',false,true,n)
go

// ticks_stream订阅mem_stream_d表的数据
def saveToTicksStreamd(mutable TB, msg): TB.append!(select * from msg)
subscribeTable(, 'mem_stream_d', 'action_to_ticksStream_tde', 0, saveToTicksStreamd{ticks_stream}, true, 100)

// ticks_stream同时订阅mem_stream_f表的数据
def saveToTicksStreamf(mutable TB, msg): TB.append!(select * from msg)
subscribeTable(, 'mem_stream_f', 'action_to_ticksStream_tfe', 0, saveToTicksStreamf{ticks_stream}, true, 100)

// dfsTB订阅ticks_stream表的数据
def saveToDFS(mutable TB, msg): TB.append!(select * from msg)
subscribeTable(, 'ticks_stream', 'action_to_dfsTB', 0, saveToDFS{dfsTB}, true, 100, 5)
```

上述几个步骤中，我们定义了一个数据库并创建分布式表 tick ，以及三个流数据表，分别为 mem_stream_d 、 mem_stream_f 和 ticks_stream 。客户端将第三方订阅而来的数据不断地追加到 mem_stream_d 和 mem_stream_f 表中，而写入这两个表的数据会被汇集到 ticks_stream 表。最后， ticks_stream 表内的数据顺序地写入分布式表 tick 中。
下面，我们将第三方订阅到的数据上传到DolphinDB，通过DolphinDB流数据订阅功能将数据追加到分布式表。我们假定Python客户端从第三方订阅到的数据已经保存在两个名为 dfd 和 dff 的DataFrame中：

```
n = 10000
dfd = pd.DataFrame({'Code': np.repeat(['SH000001', 'SH000002', 'SH000003', 'SH000004', 'SH000005'], n/5),
                    'Date': np.repeat(pd.date_range('1990.01.01', periods=10000, freq='D'), n/10000),
                    'DiffAskVol': np.random.choice(100, n),
                    'DiffAskVolSum': np.random.choice(100, n),
                    'DiffBidVol': np.random.choice(100, n),
                    'DiffBidVolSum': np.random.choice(100, n),
                    'FirstDerivedAskPrice': np.random.choice(100, n)*0.9,
                    'FirstDerivedAskVolume': np.random.choice(100, n),
                    'FirstDerivedBidPrice': np.random.choice(100, n)*0.9,
                    'FirstDerivedBidVolume': np.random.choice(100, n)})

n = 20000
dff = pd.DataFrame({'Code': np.repeat(['SZ000001', 'SZ000002', 'SZ000003', 'SZ000004', 'SZ000005'], n/5),
                    'Date': np.repeat(pd.date_range('1990.01.01', periods=10000, freq='D'), n/10000),
                    'DiffAskVol': np.random.choice(100, n),
                    'DiffAskVolSum': np.random.choice(100, n),
                    'DiffBidVol': np.random.choice(100, n),
                    'DiffBidVolSum': np.random.choice(100, n),
                    'FirstDerivedAskPrice': np.random.choice(100, n)*0.9,
                    'FirstDerivedAskVolume': np.random.choice(100, n),
                    'FirstDerivedBidPrice': np.random.choice(100, n)*0.9,
                    'FirstDerivedBidVolume': np.random.choice(100, n)})
```

注意 ： 在向流数据表追加一个带有时间列的表时，我们需要对时间列进行时间类型转换：首先将整个DataFrame上传到DolphinDB服务器，再通过select语句将其中的列取出，并转换时间类型列的数据类型，最后通过 tableInsert 语句追加表。具体原因与向内存表追加一个DataFrame类似，请参见DolphinDB Python API教程。

```
dbDir = "dfs://ticks"
tableName = 'tick'
s.upload({'dfd': dfd, 'dff': dff})
inserts = """tableInsert(mem_stream_d,select Code,date(Date) as Date,DiffAskVol,DiffAskVolSum,DiffBidVol,DiffBidVolSum,FirstDerivedAskPrice,FirstDerivedAskVolume,FirstDerivedBidPrice,FirstDerivedBidVolume from dfd);
tableInsert(mem_stream_f,select Code,date(Date) as Date,DiffAskVol,DiffAskVolSum,DiffBidVol,DiffBidVolSum,FirstDerivedAskPrice,FirstDerivedAskVolume,FirstDerivedBidPrice,FirstDerivedBidVolume from dff)"""
s.run(inserts)
s.run("select count(*) from loadTable('{dbPath}', `{tbName})".format(dbPath=dbDir,tbName=tableName))

// output
   count
0  30000
```

在DolphinDB 服务端执行以下脚本结束订阅：

```
def clears(tbName,action)
{
	unsubscribeTable(tableName=tbName, actionName=action)
	clearTablePersistence(objByName(tbName))
	undef(tbName,SHARED)
}
clears(`ticks_stream, `action_to_dfsTB)
clears(`mem_stream_d,`action_to_ticksStream_tde)
clears(`mem_stream_f,`action_to_ticksStream_tfe)
```


## 7. 状态监控

当通过订阅方式对流数据进行实时处理时，所有的计算都在后台进行，用户无法直观的看到运行的情况。DolphinDB提供以下函数监控流数据处理及流计算引擎的状态：
- getStreamingStat：全方位监控流数据处理过程。
- getStreamEngineStat：可以查看系统中定义的全部流计算引擎、各个引擎的内存占用等状态，每一类引擎对应一张表。

### 7.1. 流数据处理状态

getStreamingStat 函数返回一个dictionary，包含以下五个表：
- pubConns：列出该节点所有的订阅节点信息，发布队列情况，以及流数据表名称。
- subConns：列出每个本地节点订阅的所有发布节点的连接状态和有关接收消息的统计信息。
- persistWorkers：只有持久化启用后，才能通过 getStreamingStat 获取persistWorkers表。这张表的记录数等于persistenceWorkerNum配置值。若要并行处理持久化数据表的任务，可设置persistenceWorkerNum>1。
- subWorkers：表监控流数据订阅工作线程。当有流数据进入时，可以通过该表观察到已处理数据的信息。
- pubTables：表监控流数据表被订阅情况
在调用 subscribeTable 函数后，通过 getStreamingStat().pubTables 可以立刻查看到对应的订阅任务。在工作线程实际处理到数据后，才可以在 getStreamingStat().subWorkers 中查看到对应的工作线程的状态。

### 7.2. 流计算状态

在调用 subscribeTable 函数后，用 getStreamingStat().pubTables 可以立刻查看到对应的订阅任务，在工作线程实际处理到数据后，才可以在 getStreamingStat().subWorkers 中查看到对应的工作线程的状态。

#### 7.2.1. pubConns表

pubConns表监控本地发布节点和它的所有订阅节点之间的连接状态。每一行表示本地发布节点的一个订阅节点。它包含以下列：
| 列名称 | 说明 |
| --- | --- |
| client | 订阅节点的IP和端口信息 |
| queueDepthLimit | 发布节点消息队列允许的最大深度（消息数）。每个发布节点只有一个发布消息队列。 |
| queueDepth | 发布节点消息队列深度（消息数） |
| tables | 该节点上的所有共享的流数据表。若多表，彼此通过逗号分隔。 |
在GUI中运行getStreamingStat().pubConns查看表内容：
pubConns表会列出该节点所有的订阅节点信息，发布队列情况，以及流数据表名称。

#### 7.2.2. subConns表

subConns表监控本地订阅节点与其订阅的发布节点之间的连接状态。每个订阅的发布节点为表中一行。
| 列名称 | 说明 |
| --- | --- |
| publisher | 发布节点别名 |
| cumMsgCount | 累计接收消息数 |
| cumMsgLatency | 累计接收消息的平均延迟时间(毫秒)。延迟时间指的是消息从进入发布队列到进入订阅队列的耗时。 |
| lastMsgLatency | 最后一次接收数据延迟时间(毫秒) |
| lastUpdate | 最后一次接收数据时刻 |
在GUI中运行getStreamingStat().subConns查看表内容：
这张表列出每个本地节点订阅的所有发布节点的连接状态和有关接收消息的统计信息。

#### 7.2.3. persistWorkers表

persistWorkers表监控流数据表持久化工作线程，每个工作线程为一行。
| 列名称 | 说明 |
| --- | --- |
| workerId | 工作线程编号 |
| queueDepthLimit | 持久化消息队列深度限制 |
| queueDepth | 持久化消息队列深度 |
| tables | 持久化表名。若多表，彼此通过逗号分隔。 |
只有持久化启用后，才能通过 getStreamingStat 获取persistWorkers表。这张表的记录数等于persistenceWorkerNum配置值。以下例子在GUI中运行getStreamingStat().persistWorkers查看持久化两张数据表的线程。
当persistenceWorkerNum=1时：
当persistenceWorkerNum=3时：
从图上可以直观的看出，若要并行处理持久化数据表的任务，可设置persistenceWorkerNum>1。

#### 7.2.4. subWorkers表

subWorkers表监控流数据订阅工作线程，每条记录代表一个订阅主题。
| 列名称 | 说明 |
| --- | --- |
| workerId | 工作线程编号 |
| topic | 订阅主题 |
| queueDepthLimit | 订阅消息队列最大限制 |
| queueDepth | 订阅消息队列深度 |
| processedMsgCount | 已进入handler的消息数量 |
| failedMsgCount | handler处理异常的消息数量 |
| lastErrMsg | 上次handler处理异常的信息 |
配置项subExecutors与subExecutorPooling这两个配置项的对流数据处理的影响，在这张表上可以得到充分的展现。在GUI中使用getStreamingStat().subWorkers查看。
当subExecutorPooling=false,subExecutors=1时，内容如下：
此时，所有表的订阅消息共用一个线程队列。
当subExecutorPooling=false,subExecutors=2时，内容如下：
此时，各个表订阅消息分配到两个线程队列独立处理。
当subExecutorPooling=true,subExecutors=2时，内容如下：
此时，各个表的订阅消息共享由两个线程组成的线程池。
当有流数据进入时，可以通过这个表观察到已处理数据量等信息：

#### 7.2.5. pubTables表

pubTables表监控流数据表被订阅情况，每条记录代表流数据表一个订阅连接。
| 列名称 | 说明 |
| --- | --- |
| tableName | 发布表名称 |
| subscriber | 订阅方的host和port |
| msgOffset | 订阅线程当前订阅消息的offset |
| actions | 订阅的action。若有多个action，此处用逗号分割 |
比如存流数据发布表名称为pubTable1，发布了100条记录。 有一个订阅从offset=0开始，action名称为" act_getdata"。那么当订阅完成之后，用getStreamingStat().pubTables 查看内容为：

### 7.3. 流数据引擎状态

调用 getStreamEngineStat 会返回一个字典，其key为引擎类型名称，value为一个表，包含key对应引擎的状态。
以getStreamEngineStat().DailyTimeSeriesEngine为例，查看内容为：
在上例中，系统中仅有一个DailyTimeSeriesEngine，其引擎名为engine1，目前占用了大约32KB内存。引擎的内存占用主要是因为随着订阅的流数据不断注入引擎，存放在内存中的数据越来越多。在创建引擎时可以通过参数 garbageSize 控制清理历史数据的频率以控制引擎中的内存占用。
对于不再使用的引擎建议及时释放。通过dropStreamEngine函数释放引擎会释放掉对应的内存，若流数据引擎的句柄仍在内存中也需要释放：创建引擎时返回的句柄变量 = NULL。

## 8. 性能调优

当数据流量极大而系统来不及处理时，系统监控中会看到订阅端subWorkers表的queueDepth数值极高，此时系统"../tools/grafana.md"入端逐级反馈数据压力。当订阅端队列深度达到上限时开始阻止发布端数据进入，此时发布端的队列开始累积。当发布端的队列深度达到上限时，系统会阻止流数据注入端写入数据。
可以通过以下几种方式来优化系统对流数据的处理性能：
- 调整订阅参数中的batchSize和throttle参数，来平衡发布端和订阅端的缓存，让流数据输入速度与数据处理速度达到一个动态的平衡。若要充分发挥数据批量处理的性能优势，可以设定batchSize参数等待流数据积累到一定量时才进行消费，但是这样会带来一定程度的内存占用，而且当batchSize参数较大的时候，可能会发生数据量没有达到batchSize而长期滞留在缓冲区的情况。对于这个问题，可以选择一个合适的throttle参数值。它的作用是即使batchSize未满足，也能将缓冲区的数据消费掉。
- 通过调整subExecutors配置参数增加订阅端计算的并行度，以加快订阅端队列的消费速度。
- 当有多个executor时，若每个executor处理不同的订阅，而且不同订阅的数据流的频率或者处理复杂度差异极大，容易导致低负载的executor资源闲置。通过设置subExecutorPooling=true，可以让所有executor作为一个共享线程池，共同处理所有订阅的消息。在这种共享池模式下，所有订阅的消息进入同一个队列，多个executor从队列中读取消息并行处理。需要指出，共享线程池处理流数据的一个副作用是不能保证消息按到达的时间顺序处理。当消息需要按照抵达时间顺序被处理时，不应开启此设置。系统默认采用哈希算法为每一个订阅分配一个executor。若需要保证两个流数据表的时序同步，可在订阅函数subscribeTable中对两个订阅使用相同的hash值，来指定用同一个线程来处理这两个订阅数据流。
- 若流数据表启用同步持久化，那么磁盘的I/O可能会成为瓶颈。可参考2.6小节采用异步方式持久化数据，同时设置一个合理的持久化队列(maxPersistenceQueueDepth参数，默认值为1000万条消息)。也可使用SSD硬盘替换HDD硬盘以提高写入性能。
- 如果数据发布端(publisher)成为系统的瓶颈，譬如订阅的客户端太多可能导致发布瓶颈，可以采用以下两种处理办法。首先通过多级级联降低每一个发布节点的订阅数量，对延迟不敏感的应用可以订阅二级甚至三级的发布节点。其次调整部分参数来平衡延迟和吞吐量两个指标。参数maxMsgNumPerBlock设置批量发送消息时批的大小，默认值是1024。一般情况下，较大的批量值能提升吞吐量，但会增加网络延迟。
- 若输入流数据的流量波动较大，高峰期导致消费队列积压至队列峰值(默认1000万)，那么可以修改配置项maxPubQueueDepthPerSite和maxSubQueueDepth以增加发布端和订阅端的最大队列深度，提高系统数据流大幅波动的能力。鉴于队列深度增加时内存消耗会增加，应估算并监控内存使用量以合理配置内存。

## 9. 可视化

流数据可视化可分为两种类型：
- 实时值监控：定时刷新流数据在滑动窗口的聚合计算值，通常用于指标的监控和预警。
- 趋势监控：把新产生的数据附加到原有的数据上并以可视化图表的方式实时更新。
很多数据可视化的平台都能支持流数据的实时监控，比如当前流行的开源数据可视化框架Grafana。DolphinDB database 已经实现了Grafana的服务端和客户端的接口，具体配置可以参考 Grafana教程 。


---

> 来源: `OHLC.html`


# K 线计算

DolphinDB 提供了功能强大的内存计算引擎，内置时间序列函数，分布式计算以及流数据处理引擎，在众多场景下均可高效的计算K线。本教程将介绍DolphinDB如何通过批量处理和流式处理计算K线。
- 计算历史数据 K 线：可以指定K线窗口的起始时间；一天中可以存在多个交易时段，包括隔夜时段；K线窗口可重叠；使用交易量作为划分K线窗口的维度。需要读取的数据量特别大并且需要将结果写入数据库时，可使用DolphinDB内置的Map-Reduce函数并行计算。
- 实时计算 K 线：使用API实时接收市场数据，并使用DolphinDB内置的流数据时序计算引擎(time-series aggregator)进行实时计算得到K线数据。

## 1. 历史数据 K 线计算

使用历史数据计算K线，可使用DolphinDB的内置函数 bar ， dailyAlignedBar 或 wj 。

### 1.1. 不指定 K 线窗口的起始时刻

这种情况可使用 bar 函数。bar(X,Y)返回X减去X除以Y的余数(X-mod(X,Y))，一般用于将数据分组。如下例所示。

```
date = 09:32m 09:33m 09:45m 09:49m 09:56m 09:56m;
bar(date, 5);
```


```
[09:30m,09:30m,09:45m,09:45m,09:55m,09:55m]
```

例子1 ：使用以下数据模拟美国股票市场：

```
n = 1000000
date = take(2019.11.07 2019.11.08, n)
time = (09:30:00.000 + rand(int(6.5*60*60*1000), n)).sort!()
timestamp = concatDateTime(date, time)
price = 100+cumsum(rand(0.02, n)-0.01)
volume = rand(1000, n)
symbol = rand(`AAPL`FB`AMZN`MSFT, n)
trade = table(symbol, date, time, timestamp, price, volume).sortBy!(`symbol`timestamp)
undef(`date`time`timestamp`price`volume`symbol)
```


```
barMinutes = 5
OHLC = select first(price) as open, max(price) as high, min(price) as low, last(price) as close, sum(volume) as volume from trade group by symbol, date, bar(time, barMinutes*60*1000) as barStart
```

请注意，以上数据中，time列的精度为毫秒。若time列精度不是毫秒，则应当将 barMinutes*60*1000 中的数字做相应调整。

### 1.2. 指定 K 线窗口的起始时刻

需要指定K线窗口的起始时刻，可使用 dailyAlignedBar 函数。该函数可处理每日多个交易时段，亦可处理隔夜时段。
请注意，使用 dailyAlignedBar 函数时，时间列必须含有日期信息，包括 DATETIME, TIMESTAMP 或 NANOTIMESTAMP 这三种类型的数据。指定每个交易时段起始时刻的参数 timeOffset 必须使用相应的去除日期信息之后的 SECOND，TIME 或 NANOTIME 类型的数据。
例子2 （每日一个交易时段）：计算美国股票市场 7 分钟 K 线。数据沿用例子 1 中的 trade 表。

```
barMinutes = 7
OHLC = select first(price) as open, max(price) as high, min(price) as low, last(price) as close, sum(volume) as volume from trade group by symbol, dailyAlignedBar(timestamp, 09:30:00.000, barMinutes*60*1000) as barStart
```

例子3 （每日两个交易时段）：中国股票市场每日有两个交易时段，上午时段为 9:30 至 11:30，下午时段为 13:00 至 15:00。
使用以下脚本产生模拟数据：

```
n = 1000000
date = take(2019.11.07 2019.11.08, n)
time = (09:30:00.000 + rand(2*60*60*1000, n/2)).sort!() join (13:00:00.000 + rand(2*60*60*1000, n/2)).sort!()
timestamp = concatDateTime(date, time)
price = 100+cumsum(rand(0.02, n)-0.01)
volume = rand(1000, n)
symbol = rand(`600519`000001`600000`601766, n)
trade = table(symbol, timestamp, price, volume).sortBy!(`symbol`timestamp)
undef(`date`time`timestamp`price`volume`symbol)
```

计算 7 分钟 K 线：

```
barMinutes = 7
sessionsStart=09:30:00.000 13:00:00.000
OHLC = select first(price) as open, max(price) as high, min(price) as low, last(price) as close, sum(volume) as volume from trade group by symbol, dailyAlignedBar(timestamp, sessionsStart, barMinutes*60*1000) as barStart
```

例子4 （每日两个交易时段，包含隔夜时段）：某些期货每日有多个交易时段，且包括隔夜时段。本例中，第一个交易时段为上午 8:45 到下午 13:45，另一个时段为隔夜时段，从下午 15:00 到第二天上午 05:00。
使用以下脚本产生模拟数据:

```
daySession =  08:45:00.000 : 13:45:00.000
nightSession = 15:00:00.000 : 05:00:00.000
n = 1000000
timestamp = rand(concatDateTime(2019.11.06, daySession[0]) .. concatDateTime(2019.11.08, nightSession[1]), n).sort!()
price = 100+cumsum(rand(0.02, n)-0.01)
volume = rand(1000, n)
symbol = rand(`A120001`A120002`A120003`A120004, n)
trade = select * from table(symbol, timestamp, price, volume) where timestamp.time() between daySession or timestamp.time()>=nightSession[0] or timestamp.time()<nightSession[1] order by symbol, timestamp
undef(`timestamp`price`volume`symbol)
```


```
barMinutes = 7
sessionsStart = [daySession[0], nightSession[0]]
OHLC = select first(price) as open, max(price) as high, min(price) as low, last(price) as close, sum(volume) as volume from trade group by symbol, dailyAlignedBar(timestamp, sessionsStart, barMinutes*60*1000) as barStart
```


### 1.3. 重叠 K 线窗口

以上例子中，K 线窗口均不重叠。若要计算重叠K线窗口，可以使用 wj 函数对左表中的每一行，在右表中截取一段窗口，进行计算。
例子5 （每日两个交易时段，重叠的K线窗口）：模拟中国股票市场数据，每5分钟计算30分钟K线。

```
n = 1000000
sampleDate = 2019.11.07
symbols = `600519`000001`600000`601766
trade = table(take(sampleDate, n) as date, 
	(09:30:00.000 + rand(7200000, n/2)).sort!() join (13:00:00.000 + rand(7200000, n/2)).sort!() as time, 
	rand(symbols, n) as symbol, 
	100+cumsum(rand(0.02, n)-0.01) as price, 
	rand(1000, n) as volume)
```

首先生成窗口，并且使用 cj 函数来生成股票和交易窗口的组合。

```
barWindows = table(symbols as symbol).cj(table((09:30:00.000 + 0..23 * 300000).join(13:00:00.000 + 0..23 * 300000) as time))
```

然后使用 wj 函数计算重叠窗口的K线数据：

```
OHLC = wj(barWindows, trade, 0:(30*60*1000), 
		<[first(price) as open, max(price) as high, min(price) as low, last(price) as close, sum(volume) as volume]>, `symbol`time)
```


### 1.4. 使用交易量划分K线窗口

上面的例子我们均使用时间作为划分K线窗口的维度。在实践中，也可以使用其他变量，譬如用累计的交易量作为划分K线窗口的依据。
例子6 （使用累计的交易量计算K线）：交易量每增加1000000计算一次 K 线。
代码采用了嵌套查询的方法。子查询为每个股票生成累计的交易量cumvol，然后在主查询中根据累计的交易量用 bar 函数生成窗口。

### 1.5. 使用 MapReduce 函数加速

若需从数据库中提取较大量级的历史数据，计算K线，然后存入数据库，可使用DolphinDB内置的Map-Reduce函数 mr 进行数据的并行读取与计算。这种方法可以显著提高速度。
本例使用美国股票市场的交易数据。原始数据存于"dfs://TAQ"数据库的"trades"表中。"dfs://TAQ"数据库采用复合分区：基于交易日期Date的值分区与基于股票代码Symbol的范围分区。
(1) 将存于磁盘的原始数据表的元数据载入内存：

```
login(`admin, `123456)
db = database("dfs://TAQ")
trades = db.loadTable("trades")
```

(2) 在磁盘上创建一个空的数据表，以存放计算结果。以下代码建立一个模板表（model），并根据此模板表的schema在数据库"dfs://TAQ"中创建一个空的 OHLC 表以存放K线计算结果：

```
model=select top 1 Symbol, Date, Time.second() as bar, PRICE as open, PRICE as high, PRICE as low, PRICE as close, SIZE as volume from trades where Date=2007.08.01, Symbol=`EBAY
if(existsTable("dfs://TAQ", "OHLC"))
	db.dropTable("OHLC")
db.createPartitionedTable(model, `OHLC, `Date`Symbol)
```

(3) 使用 mr 函数计算K线数据，并将结果写入 OHLC 表中：

```
def calcOHLC(inputTable){
	tmp=select first(PRICE) as open, max(PRICE) as high, min(PRICE) as low, last(PRICE) as close, sum(SIZE) as volume from inputTable where Time.second() between 09:30:00 : 15:59:59 group by Symbol, Date, 09:30:00+bar(Time.second()-09:30:00, 5*60) as bar
	loadTable("dfs://TAQ", `OHLC).append!(tmp)
	return tmp.size()
}
ds = sqlDS(<select Symbol, Date, Time, PRICE, SIZE from trades where Date between 2007.08.01 : 2019.08.01>)
mr(ds, calcOHLC, +)
```

- ds是函数 sqlDS 生成的一系列数据源，每个数据源代表从一个数据分区中提取的数据。
- 自定义函数 calcOHLC 为Map-Reduce算法中的map函数，对每个数据源计算K线数据，并将结果写入数据库，返回写入数据库的K线数据的行数。
- "+"是Map-Reduce算法中的reduce函数，将所有map函数的结果，亦即写入数据库的K线数据的行数相加，返回写入数据库的K线数据总数。

## 2. 实时K线计算

DolphinDB 中计算实时 K 线的流程如下图所示：
实时数据供应商一般会提供基于Python、Java或其他常用语言的API的数据订阅服务。本例中使用Python来模拟接收市场数据，通过DolphinDB Python API写入流数据表中。DolphinDB的流数据时序聚合引擎可按照指定的频率与移动窗口实时计算K线。
本例使用的模拟实时数据源为文本文件 trades.csv 。该文件包含以下4列（附带一行样本数据）：
| Symbol | Datetime | Price | Volume |
| --- | --- | --- | --- |
| 000001 | 2018.09.03T09:30:06 | 10.13 | 4500 |
以下三小节介绍实时K线计算的三个步骤：

### 2.1. 使用 Python 接收实时数据，并写入DolphinDB 流数据表

- DolphinDB 中建立流数据表 share streamTable(100:0, `Symbol`Datetime`Price`Volume,[SYMBOL,DATETIME,DOUBLE,INT]) as Trade

### 2.2. 实时计算K线

可使用移动窗口实时计算K线数据，一般分为以下2种情况：
- 仅在每次时间窗口结束时触发计算。 时间窗口完全不重合，例如每隔5分钟计算过去5分钟的K线数据。 时间窗口部分重合，例如每隔1分钟计算过去5分钟的K线数据。
- 在每个时间窗口结束时触发计算，同时在每个时间窗口内数据也会按照一定频率更新。 例如每隔1分钟计算过去1分钟的K线数据，但最近1分钟的K线不希望等到窗口结束后再计算。希望每隔1秒钟更新一次。
下面针对上述的几种情况分别介绍如何使用 createTimeSeriesAggregator 函数实时计算K线数据。请根据实际需要选择相应场景创建时间序列聚合引擎。

#### 2.2.1. 仅在每次时间窗口结束时触发计算

时间窗口不重合，可将 createTimeSeriesAggregator 函数的windowSize参数和step参数设置为相同值。时间窗口部分重合，可将windowSize参数设为大于step参数。请注意，windowSize必须是step的整数倍。
场景一：每隔 5 分钟计算过去 5 分钟的 K 线数据。

```
share streamTable(100:0, `datetime`symbol`open`high`low`close`volume,[DATETIME,SYMBOL,DOUBLE,DOUBLE,DOUBLE,DOUBLE,LONG]) as OHLC1
tsAggr1 = createTimeSeriesAggregator(name="tsAggr1", windowSize=300, step=300, metrics=<[first(Price),max(Price),min(Price),last(Price),sum(volume)]>, dummyTable=Trade, outputTable=OHLC1, timeColumn=`Datetime, keyColumn=`Symbol)
subscribeTable(tableName="Trade", actionName="act_tsAggr1", offset=0, handler=append!{tsAggr1}, msgAsTable=true);
```

场景二：每隔 1 分钟计算过去 5 分钟的 K 线数据。

#### 2.2.2. 在每个窗口内进行多次计算

以每分钟计算 vwap 价格为例，当前窗口内即使发生了多次交易，窗口结束前都不会触发任何使用当前窗口数据的计算。某些用户希望在当前窗口结束前频繁使用已有数据计算K线，这时可指定 createTimeSeriesAggregator 函数的updateTime参数。指定updateTime参数后，当前窗口结束前可能会发生多次针对当前窗口的计算。这些计算触发的规则为：
(1) 将当前窗口分为 windowSize/updateTime 个小窗口，每个小窗口长度为 updateTime。一个小窗口结束后，若有一条新数据到达，且在此之前当前窗口内有未参加计算的的数据，会触发一次计算。请注意，该次计算不包括这条新数据。
(2) 一条数据到达聚合引擎之后经过2*updateTime（若2*updateTime不足2秒，则设置为2秒），若其仍未参与计算，会触发一次计算。该次计算包括当时当前窗口内的所有数据。
若进行分组计算，以上规则在每组之内应用。在使用updateTime参数时，step必须是updateTime的整数倍。必须使用键值表作为输出表。若未指定keyColumn参数，主键为timeColumn列；若指定了keyColumn参数，主键为timeColumn列和keyColumn列。有关updateTime参数更多细节，请参考 时序聚合引擎教程
例如，要计算1分钟窗口的K线，但当前1分钟的K线不希望等到窗口结束后再计算，而是希望新数据进入后最迟2秒钟就计算。可通过如下步骤实现。
首先，创建一个键值表作为输出表，并将时间列和股票代码列作为主键。

```
share keyedTable(`datetime`Symbol, 100:0,  `datetime`Symbol`open`high`low`close`volume,[DATETIME,SYMBOL,DOUBLE,DOUBLE,DOUBLE,DOUBLE,LONG]) as OHLC
```

使用以下脚本定义时序聚合引擎。其中指定updateTime参数取值为1(秒)。useWindowStartTime参数设为true，表示输出表第一列为数据窗口的起始时间。

```
tsAggr = createTimeSeriesAggregator(name="tsAggr", windowSize=60, step=60, metrics=<[first(Price),max(Price),min(Price),last(Price),sum(volume)]>, dummyTable=Trade, outputTable=OHLC, timeColumn=`Datetime, keyColumn=`Symbol, updateTime=1, useWindowStartTime=true)
```


```
subscribeTable(tableName="Trade", actionName="act_tsaggr", offset=0, handler=append!{tsAggr}, msgAsTable=true);
```


### 2.3. 在 Python 中展示 K 线数据

在本例中，聚合引擎的输出表也定义为流数据表，客户端可以通过 Python API 订阅输出表，并将计算结果展现到 Python 终端。
以下代码使用 Python API 订阅实时聚合计算的输出结果表 OHLC，并将结果通过 print 函数打印出来。

```
from threading import Event
import dolphindb as ddb
import pandas as pd
import numpy as np
s=ddb.session()
#设定本地端口20001用于订阅流数据
s.enableStreaming(20001)
def handler(lst):         
    print(lst)
# 订阅DolphinDB(本机8848端口)上的OHLC流数据表
s.subscribe("127.0.0.1", 8848, handler, "OHLC","python_api_subscribe",0)
Event().wait()
```

也可通过 Grafana 等可视化系统来连接 DolphinDB，对输出表进行查询并将结果以图表方式展现。


---

> 来源: `reactive_state_engine.html`


# 响应式状态引擎

量化金融的研究和实盘中，越来越多的机构需要根据高频的行情数据（L1/L2以及逐笔委托数据）来计算量价因子，每只股票的每一条新数据的注入都会更新该只股票的所有因子值。这些因子通常是有状态的：不仅与当前的多个指标有关，而且与多个指标的历史状态相关。以国内的股票市场为例，每3秒收到一个快照，每个股票每天4800个快照，计算因子时可能会用到之前若干个快照的数据，甚至之前若干天的数据。绝大多数机构的研发环境系统（例如Python）与生产环境系统（例如C++）不同，要维护两套代码，是非常沉重的负担。
本教程介绍如何使用DolphinDB 1.30.3版本发布的响应式状态引擎（Reactive State Engine），实现流批统一计算带有状态的高频因子。状态引擎接受在历史数据批量处理（研发阶段）中编写的表达式或函数作为输入进行流式计算，并确保流式计算的结果与批量计算完全一致。只要在历史数据的批量计算中验证正确，即可保证流数据的实时计算正确。这避免了在生产环境中重写研发代码的高额成本，以及维护研发和生产两套代码的负担。

## 1. 金融高频因子计算示例

本节通过一个金融高频因子计算的示例，介绍高频因子计算中的挑战。以下的因子表达式以DolphinDB脚本语言编写，使用了用户自定义函数 sum\_diff 和内置函数 ema (exponential moving average)。 sum_diff 是一个无状态函数， ema 是一个有状态的函数，依赖历史数据。更为棘手的是，如以下计算分解图所示，需要使用 ema 函数的多重嵌套。

```
def sum_diff(x, y){
    return (x-y)/(x+y)
}

ema(1000 * sum_diff(ema(price, 20), ema(price, 40)),10) -  ema(1000 * sum_diff(ema(price, 20), ema(price, 40)), 20)
```

面对此类场景，我们需要解决以下几个问题：
- 投研阶段能否使用历史数据快速为每只股票计算100~1000个类似的因子？
- 实盘阶段能否在每个行情tick数据到来时为每只股票计算100~1000个类似的因子？
- 批处理和流计算的代码实现是否高效？批和流能否统一代码？正确性校验是否便捷？

## 2. 现有解决方案的优缺点

Python pandas/numpy目前是研究阶段最常用的高频因子解决方案。pandas对历史面板数据处理有非常成熟的解决方案，而且内置了大部分高频因子计算需要用到的算子，可快速开发高频因子。但pandas无论对历史数据计算，还是对实时数据计算的性能都较差。对历史数据计算时，单线程的计算性能也存在较大提升空间，此外，由于python的Global interpreter lock的限制，无法进行并行计算；对实时数据进行计算时，由于python仅支持全量计算，不支持增量计算，所以无法达到实时计算的性能要求。
为生产环境中的性能考虑，很多机构会用C++重新实现研究（历史数据）代码。不过，这种方法需要维护两套代码，开发成本（时间和人力）会大幅增加。此外，还要耗费大量精力确保两套系统的结果完全一致。
Flink是一种批流统一的解决方案。Flink支持SQL和窗口函数，高频因子用到的基本算子在Flink中已经内置实现。因此，简单的因子用Flink实现比较高效，运行性能也较好。但Flink最大的问题是无法实现复杂的高频因子计算。如前一章中提到的例子，需要多个窗口函数的嵌套，无法直接用Flink实现。这也正是DolphinDB开发响应式状态引擎的动机所在。

## 3. 响应式状态引擎（Reactive State Engine)

响应式状态引擎实际上是一个计算黑盒，输入在历史数据上已经验证的DolphinDB因子代码（表达式或函数）以及实时行情数据，输出实时因子值。由于在静态的历史数据集上开发和验证高频因子远比在流数据上开发更为简单，响应式状态引擎显著降低了流式高频因子的开发成本和难度。

```
def sum_diff(x, y){
    return (x-y)/(x+y)
}
factor1 = <ema(1000 * sum_diff(ema(price, 20), ema(price, 40)),10) -  ema(1000 * sum_diff(ema(price, 20), ema(price, 40)), 20)>

share streamTable(1:0, `sym`price, [STRING,DOUBLE]) as tickStream
result = table(1000:0, `sym`factor1, [STRING,DOUBLE])
rse = createReactiveStateEngine(name="reactiveDemo", metrics=factor1, dummyTable=tickStream, outputTable=result, keyColumn="sym")
subscribeTable(tableName=`tickStream, actionName="factors", handler=tableInsert{rse})
```

以上代码在DolphinDB中实现前述因子的流式计算。factor1是前述因子在历史数据上的实现，不做任何改变，直接传递给响应式状态引擎rse，即可实现流式计算。通过订阅函数 subscribeTable ，将流数据表tickStream与状态引擎rse进行关联。每一批次实时数据的注入，都会触发状态引擎的计算，并输出因子值到结果表result。以下代码产生随机数据，并注入到流数据表。结果与通过SQL语句计算的结果完全相同。

```
data = table(take("A", 100) as sym, rand(10.0, 100) as price)
tickStream.append!(data)
factor1Hist = select sym, ema(1000 * sum_diff(ema(price, 20), ema(price, 40)),10) -  ema(1000 * sum_diff(ema(price, 20), ema(price, 40)), 20) as factor1 from data context by sym
assert each(eqObj, result.values(), factor1Hist.values())
```


### 3.1. 工作原理

如图1所示，一个有状态的高频因子计算过程可以分解成有一个有向无环图（DAG）。图中的节点有3种：（1）数据源，如price（2）有状态的算子，如a, b, d, e （3）无状态的算子，如c和result。从数据源节点开始，按照既定的路径，层层推进，得到最后的因子输出。这非常类似excel中的单元格链式计算。当一个单元格的数据发生变化时，相关联的单元格依次发生变化。响应式状态引擎的名称也是从这一点引申出来的。
无状态的算子比较简单，使用DolphinDB已有的脚本引擎，就可以表示和计算。因此，问题转化为两点：（1）如何解析得到一个优化的DAG，（2）如何优化每个有状态的算子的计算。

### 3.2. 解析和优化

DolphinDB的脚本语言是支持向量化和函数化的多范式编程语言。通过函数的调用关系，不难得到计算步骤的DAG。在解析的时候，因为输入消息的schema是已知的，我们可以快速推断出每一个节点的输入数据类型和输出数据类型。输入参数类型确定，函数名称确定，每个状态算子的具体实例就可以创建出来。
每一个算子（有状态和无状态）在DolphinDB中都可以转化为一个唯一的字符串序列。据此，我们可以删除重复的算子，提高计算效率。

### 3.3. 内置的状态函数

状态算子计算时需要用到历史状态。如果每一次计算都使用全量数据，性能不佳。状态函数的优化，也就是增量方式的流式实现非常关键。下列状态函数在 DolphinDB 的响应式状态引擎中的实现均得到了优化。目前，状态引擎不允许使用未经优化的状态函数，且需避免使用聚合函数。
- 累计窗口函数：cumavg, cumsum, cumprod, cumcount, cummin, cummax, cumvar, cumvarp, cumstd, cumstdp, cumcorr, cumcovar, cumbeta, cumwsum, cumwavg, cumfirstNot, cumlastNot, cummed, cumpercentile, cumPositiveStreak
- 序列相关函数：deltas, ratios, ffill, move, prev, iterate, ewmMean, ewmVar, ewmStd, ewmCov, ewmCorr, prevState
- topN相关函数：msumTopN, mavgTopN, mstdpTopN, mstdTopN, mvarpTopN, mvarTopN, mcorrTopN, mbetaTopN, mcovarTopN, mwsumTopN
- 高阶函数：segmentby(参数 func 暂支持 cumsum, cummax, cummin, cumcount, cumavg, cumstd, cumvar, cumstdp, cumvarp)
- 其他函数：talibNull, dynamicGroupCumsum, dynamicGroupCumcount, topRange, lowRange, trueRange
注意，talib 作为状态函数时，第一个参数 func 只能是响应式状态引擎支持的状态函数。在后续的版本中，DolphinDB将允许用户用插件来开发自己的状态函数，注册后即可在状态引擎中使用。

### 3.4. 自定义状态函数

响应式状态引擎中可使用自定义状态函数。需要注意以下几点：
- 函数定义前，使用 @state 表示函数是自定义的状态函数。
- 自定义状态函数中只能使用赋值语句和return语句。return语句必须是最后一个语句，可返回多个值。
- 使用iif函数表示if...else的逻辑。
如果仅允许使用一个表达式来表示一个因子，会带来很多局限性。首先，在某些情况下，仅使用表达式，无法实现一个完整的因子。下面的例子返回线性回归的alpha，beta和residual。

```
@state
def slr(y, x){
    alpha, beta = mslr(y, x, 12)
    residual = mavg(y, 12) - beta * mavg(x, 12) - alpha
    return alpha, beta, residual
}
```

其次，很多因子可能会使用共同的中间结果，定义多个因子时，代码会更简洁。自定义函数可以同时返回多个结果。下面的函数multiFactors定义了5个因子。

```
@state
def multiFactors(lowPrice, highPrice, volumeTrade, closePrice, buy_active, sell_active, tradePrice, askPrice1, bidPrice1, askPrice10, agg_vol, agg_amt){
    a = ema(askPrice10, 30)
    term0 = ema((lowPrice - a) / (ema(highPrice, 30) - a), 50)
    term1 = mrank((highPrice - a) / (ema(highPrice, 5) - a), true,  15)
    term2 = mcorr(askPrice10, volumeTrade, 10) * mrank(mstd(closePrice, 20, 20), true, 10)
    buy_vol_ma = mavg(buy_active, 6)
    sell_vol_ma = mavg(sell_active, 6)
    zero_free_vol = iif(agg_vol==0, 1, agg_vol)
    stl_prc = ffill(agg_amt \ zero_free_vol \ 20).nullFill(tradePrice)
    buy_prop = stl_prc
	
    spd = askPrice1 - bidPrice1
    spd_ma = round(mavg(iif(spd < 0, 0, spd), 6), 5)
    term3 = buy_prop * spd_ma
    term4 = iif(spd_ma == 0, 0, buy_prop / spd_ma)
    return term0, term1, term2, term3, term4
}
```

最后，某些表达式冗长，缺乏可读性。第一节中的因子表达式改为下面的自定义状态函数factor1后，计算逻辑简洁明了。

```
@state
def factor1(price) {
    a = ema(price, 20)
    b = ema(price, 40)
    c = 1000 * sum_diff(a, b)
    return  ema(c, 10) - ema(c, 20)
}
```


### 3.5. 输出结果过滤

状态引擎会对输入的每一条消息做出计算响应，产生一条记录作为结果，计算的结果在默认情况下都会输出到结果表，也就是说输入n个消息，输出n条记录。如果希望仅输出一部分结果，可以启用过滤条件，只有满足条件的结果才会输出。
下面的例子检查股票价格是否有变化，只有价格变化的记录才会输出。

```
share streamTable(1:0, `sym`price, [STRING,DOUBLE]) as tickStream
result = table(1000:0, `sym`price, [STRING,DOUBLE])
rse = createReactiveStateEngine(name="reactiveFilter", metrics =[<price>], dummyTable=tickStream, outputTable=result, keyColumn="sym", filter=<prev(price) != price>)
subscribeTable(tableName=`tickStream, actionName="filter", handler=tableInsert{rse})
```


### 3.6. 快照机制

为了满足生产环境业务持续性的需要，DolphinDB内置的流式计算引擎包括响应式状态引擎均支持快照（snapshot）输出。
响应式状态引擎的快照包括已处理的最后一条消息的ID以及引擎当前的状态（中间计算结果）。当系统出现异常，重新初始化状态引擎时，可恢复到最后一个快照的状态，并且从已处理的消息的下一条开始订阅。
响应式状态引擎要启用快照机制，创建时需要指定两个额外的参数snapshotDir和snapshotIntervalInMsgCount。snapshotDir用于指定存储快照的目录。snapshotIntervalInMsgCount指定处理多少条消息后产生一个快照。引擎初始化时，系统会检查快照目录下是否存在一个以引擎名称命名，后缀为snapshot的文件。以上面的代码为例，如果存在文件/home/data/snapshot/reactiveDemo.snapshot，加载这个快照。函数getSnapshotMsgId可以获取最近一个快照对应的msgId。如果不存在快照，返回-1。
状态引擎要启用快照机制，调用subscribeTable函数也需相应的修改：
- 首先必须指定消息的offset。
- 其次，handler必须使用appendMsg函数。appendMsg函数接受两个参数，msgBody和msgId。
- 再次，参数handlerNeedMsgId必须指定为true。

### 3.7. 并行处理

当需要处理大量消息时，可在DolphinDB消息订阅函数 subscribeTable 中指定可选参数filter与hash，让多个订阅客户端并行处理消息。
- 参数filter用于指定消息过滤逻辑。目前支持三种过滤方式，分别为值过滤，范围过滤和哈希过滤。
- 参数hash可以指定一个哈希值，确定这个订阅由哪个线程来执行。例如，配置参数subExecutors为4，用户指定了哈希值5，那么该订阅的计算任务将由第二个线程来执行。
下面是响应式状态引擎并行计算因子的例子。假设配置参数subExecutors=4，创建4个状态引擎，每个状态引擎根据流表的股票代码的哈希值来订阅不同股票的数据，并且指定不同的订阅线程来处理，最终将结果输出到同一个输出表中。
需要注意的是，如果多个状态引擎是同一个输出表，该输出表必须是一个共享表。没有共享的表不是线程安全的，并行写入可能会导致系统崩溃。

## 4. 流批统一解决方案

金融高频因子的流批统一处理在DolphinDB中有两种实现方法。
第一种方法，使用函数或表达式实现金融高频因子，代入不同的计算引擎进行历史数据或流数据的计算。代入SQL引擎，可以实现对历史数据的计算；代入响应式状态引擎，可以实现对流数据的计算。这在第3章的序言部分已经举例说明。在这种模式下用DolphinDB脚本语言表示的表达式或函数实际上是对因子语义的一种描述，而不是具体的实现。因子计算的具体实现交由相应的计算引擎来完成，从而实现不同场景下的最佳性能。
第二种方法，历史数据通过回放，转变成流数据，然后使用流数据计算引擎来完成计算。我们仍然以教程开始部分的因子为例，唯一的区别是流数据表tickStream的数据源来自于历史数据库的replay。使用这种方法计算历史数据的因子值，效率不高是一个缺点。

## 5. 性能测试

我们测试了响应式状态引擎计算因子的性能。测试使用模拟数据，并使用warmupStreamEngine函数模拟状态引擎已经处理部分数据的情况。测试共包括20个不同复杂度度的因子，其中两个自定义状态函数分别返回3个和5个因子。为方便测试，计算仅使用单线程处理。
我们统计了10次的总耗时，取平均值作为单次的耗时。测试使用的服务器CPU为Intel(R) Xeon(R) Silver 4216 CPU @ 2.10GHz。单线程情况下，测试结果如下：
| 股票个数 | 因子个数 | 耗时(单位:ms) |
| --- | --- | --- |
| 4000 | 20 | 6 |
| 1 | 20 | 0.07 |
| 4000 | 1 | 0.8 |
| 200 | 20 | 0.2 |

## 6. 多个引擎的流水线处理

DolphinDB内置的流计算引擎包括响应式状态引擎，时间序列聚合引擎，横截面引擎和异常检测引擎。这些引擎均实现了数据表（table）的接口，因此多个引擎流水线处理变得异常简单，只要将后一个引擎作为前一个引擎的输出即可。引入流水线处理，可以解决更为复杂的因子计算问题。譬如，因子计算经常需要使用面板数据，完成时间序列和横截面两个维度的计算，只要把响应式状态引擎和横截面两个引擎串联处理即可完成。
下面的例子是World Quant 101个Alpha因子中的1号因子公式的流数据实现。rank函数是一个横截面操作。rank的参数部分用响应式状态引擎实现。rank函数本身用横截面引擎实现。横截面引擎作为状态引擎的输出。

```
Alpha#001公式：rank(Ts_ArgMax(SignedPower((returns<0?stddev(returns,20):close), 2), 5))-0.5

//创建横截面引擎，计算每个股票的rank
dummy = table(1:0, `sym`time`maxIndex, [SYMBOL, TIMESTAMP, DOUBLE])
resultTable = streamTable(10000:0, `time`sym`factor1, [TIMESTAMP, SYMBOL, DOUBLE])
ccsRank = createCrossSectionalAggregator(name="alpha1CCS", metrics=<[sym, rank(maxIndex, percent=true) - 0.5]>,  dummyTable=dummy, outputTable=resultTable,  keyColumn=`sym, triggeringPattern='keyCount', triggeringInterval=3000, timeColumn=`time)

@state
def wqAlpha1TS(close){
    ret = ratios(close) - 1
    v = iif(ret < 0, mstd(ret, 20), close)
    return mimax(signum(v)*v*v, 5)
}

//创建响应式状态引擎，输出到前面的横截面引擎ccsRank
input = table(1:0, `sym`time`close, [SYMBOL, TIMESTAMP, DOUBLE])
rse = createReactiveStateEngine(name="alpha1", metrics=<[time, wqAlpha1TS(close)]>, dummyTable=input, outputTable=ccsRank, keyColumn="sym")
```

流水线处理（也称为引擎多级级联）和多个流数据表的级联处理有很大的区别。两者可以完成相同的任务，但是效率上有很大的区别。后者涉及多个流数据表与多次订阅。前者实际上只有一次订阅，所有的计算均在一个线程中依次顺序完成，因而有更好的性能。
上面的例子是由用户来区分哪一部分是横截面操作，哪一部分是时间序列操作以实现多个引擎的流水线。在1.30.16/2.00.4及之后的版本中，新增函数 streamEngineParser ，支持将metrics自动分解成多个内置流计算引擎的流水线。在 streamEngineParser 中以行函数（rowRank，rowSum等）表示横截面操作的语义，以rolling函数表示时间序列操作，从而系统能够自动识别一个因子中的横截面操作和时间序列操作，进一步自动构建引擎流水线。因此，上述因子可以用 streamEngineParser 更简洁的实现，metrics几乎等同于因子的数学公式表达，而不需要考虑不同类型引擎的选择：

```
@state
def wqAlpha1TS(close){
	ret = ratios(close) - 1
	v = iif(ret < 0, mstd(ret, 20), close)
	return mimax(signum(v)*v*v, 5)
}

//构建计算因子
metrics=<[sym, rowRank(wqAlpha1TS(close), percent=true)- 0.5]>

streamEngine=streamEngineParser(name="alpha1_parser", metrics=metrics, dummyTable=input, outputTable=resultTable, keyColumn=`sym, timeColumn=`time, triggeringPattern='keyCount', triggeringInterval=3000)
```


## 7. 展望

响应式状态引擎内置了大量常用的状态算子，支持自定义状态函数，也可与其他流式计算引擎以流水线的方式任意组合，方便开发人员快速实现复杂的金融高频因子。后续的版本中，将开放接口允许用户用C++插件开发状态函数，满足定制的需要。
内置的状态算子全部使用C++开发实现，算法上经过了大量的优化，以增量方式实现状态算子的流式计算，因而在单个线程上的计算达到了非常好的性能。对于规模较大的任务，可以通过订阅过滤的方式，拆分成多个子订阅，由多个节点以及每个节点的多个CPU并行完成订阅计算。后续的版本将完善计算子作业的创建、管理和监控功能，从手动转变为自动。


---

> 来源: `stream_aggregator.html`


# 时序聚合引擎

DolphinDB提供了多种轻量且使用方便的流数据引擎。本教程讲述流数据时序引擎。

## 1. 创建时序引擎

流数据时序引擎由函数 createTimeSeriesEngine 创建。该函数返回一个抽象的表对象，为时序引擎入口。向这个抽象表写入数据，就意味着数据进入时序引擎进行计算。
createTimeSeriesEngine 函数必须与 subscribeTable 函数配合使用。通过 subscribeTable 函数，时序引擎入口订阅一个流数据表。新数据进入流数据表后会被推送到时序引擎入口，按照指定规则进行计算，并将计算结果输出。

### 1.1. 语法

createTimeSeriesEngine(name, windowSize, step, metrics, dummyTable, outputTable, [timeColumn], [useSystemTime=false], [keyColumn], [garbageSize], [updateTime], [useWindowStartTime], [roundTime=true], [snapshotDir], [snapshotIntervalInMsgCount], [fill='none'])

### 1.2. 参数介绍

本节对各参数进行简单介绍。在下一小节中，对部分参数结合实例进行详细介绍。
- name
必选参数，表示时序引擎的名称，是时序引擎在一个数据节点上的唯一标识。可包含字母，数字和下划线，但必须以字母开头。
- useSystemTime
可选参数，表示时序引擎计算的触发方式。
当参数值为true时，引擎会按照数据进入引擎的时刻（毫秒精度的本地系统时间，与数据中的时间列无关），每隔固定时间截取固定长度窗口的流数据进行计算。只要一个数据窗口中含有数据，数据窗口结束后就会自动进行计算。结果中的第一列为计算发生的时间戳，与数据中的时间无关。
当参数值为false（默认值）时，引擎根据流数据中的timeColumn列来截取数据窗口。一个数据窗口结束后的第一条新数据才会触发该数据窗口的计算。请注意，触发计算的数据并不会参与该次计算。
例如，一个数据窗口从10:10:10到10:10:19。若useSystemTime=true，则只要该窗口中至少有一条数据，该窗口的计算会在窗口结束后的10:10:20触发。若useSystemTime=false，且10:10:19后的第一条数据为10:10:25，则该窗口的计算会在10:10:25触发。
- windowSize
类型：整型或者整型的数组
必选参数，指定数据窗口长度。数据窗口包含左边界，但不包含右边界。
- step
必选参数，指定窗口每次移动的时间间隔。windowSize必须可被step整除。
windowSize与step的单位取决于useSystemTime参数。若useSystemTime=true，windowSize与step的单位是毫秒。若useSystemTime=false，windowSize与step的单位是timeColumn列的精度。例如，若timeColumn列是timestamp类型，那么windowSize与step的单位是毫秒；如果timeColumn列是datetime类型，那么windowSize与step的单位是秒。
为了方便观察和对比计算结果，系统会对第一个数据窗口的起始时刻进行规整。例如若第一条数据进入引擎的时刻为2018.10.10T03:26:39.178，且step=100，那么系统会将第一个窗口起始时间规整为2018.10.10T03:26:39.100。规整数据窗口的细节在第1.3.1小节中介绍。
当引擎使用分组计算时，所有分组的窗口均进行统一的规整。相同时刻的数据窗口在各组均有相同的边界。
- metrics
类型：元代码或者元代码的数组
必选参数。引擎的核心参数，以元代码的格式表示计算公式。它可以是一个或多个系统内置或用户自定义的聚合函数，比如<[sum(volume),avg(price)]>；可对聚合结果使用表达式，比如<[avg(price1)-avg(price2)]>；也可对列与列的计算结果进行聚合计算，例如<[std(price1-price2)]>；也可以是一个函数返回多个指标。若自定义函数 func(price)返回两个结果，则可以指定metrics为<[func(price) as ['res1','res2']> 。
如果windowSize为数组，则metrics必须和windowSize大小一致的数组，一一对应计算。比如定义windowSize=[3,6], metrics=[<[sum(volume),avg(price)]>, <std(volume)>], 则sum(volume)和avg(price)按windowSize=3聚合，std(volume)按windowSize=6聚合。
DolphinDB 针对部分内置的聚合函数在流数据时序引擎中的使用进行了优化，最大程度降低了重复计算，显著提高运行速度，详情参照用户手册createTimeSeriesEngine函数。
- dummyTable
必选参数。该表的唯一作用是为引擎提供流数据中每一列的数据类型，可以含有数据，亦可为空表。该表的schema必须与订阅的流数据表相同。
- outputTable
必选参数，为结果的输出表。
在使用 createTimeSeriesEngine 函数之前，需要将输出表预先设立为一个空表，并指定各列列名以及数据类型。流数据引擎会将计算结果插入该表。
输出表的schema需要遵循以下规范：
(1) 输出表的第一列必须是时间类型。若useSystemTime=true，为TIMESTAMP类型；若useSystemTime=false，数据类型与timeColumn列一致。
(2) 如果分组列keyColumn参数不为空，那么输出表的第二列必须是分组列。
(3) 最后保存计算结果。
输出表的schema为"时间列，分组列(可选)，结果列1，结果列2..."这样的格式。
- timeColumn
可选参数。当useSystemTime=false时，指定订阅的流数据表中时间列的名称。
- keyColumn
可选参数，表示分组字段名。若设置，则分组进行计算，例如以每支股票为一组进行计算。
- garbageSize
随着订阅的流数据在时序引擎中不断积累，存放在内存中的数据会越来越多，这时需要清理不再需要的历史数据。当内存中历史数据行数超过garbageSize值时，系统会清理本次计算不需要的历史数据。garbageSize的默认值是50,000。
如果指定了keyColumn，内存清理是各组内独立进行的。当一个组在内存中的历史数据记录数超出garbageSize时，会清理该组中本次计算中不需要的历史数据。
- updateTime
如果没有指定updateTime，一个数据窗口结束前，不会发生对该数据窗口数据的计算。若要求在当前窗口结束前对当前窗口已有数据进行计算，可指定updateTime。step必须是updateTime的整数倍。要设置updateTime，useSystemTime必须设为false。
如果指定了updateTime，当前窗口内可能会发生多次计算。这些计算触发的规则为：
(1) 将当前窗口分为windowSize/updateTime个小窗口，每个小窗口长度为updateTime。一个小窗口结束后，若有一条新数据到达，且在此之前当前窗口内有未参加计算的的数据，会触发一次计算。请注意，该次计算不包括这条新数据。
(2) 一条数据到达时序引擎之后经过2*updateTime（若2*updateTime不足2秒，则设置为2秒），若其仍未参与计算，会触发一次计算。该次计算包括当时当前窗口内的所有数据。
若分组计算，则每组内进行上述操作。
请注意，当前窗口内每次计算结果的时间戳均为当前数据窗口开始时间或开始时间 + windowSize（由参数useWindowStartTime决定），而非当前窗口内的时刻。
如果指定了updateTime，输出表必须是键值内存表（使用 keyedTable 函数创建）：如果没有指定keyColumn，输出表的主键是timeColumn；如果指定了keyColumn，输出表的主键是timeColumn和keyColumn。输出表若使用普通内存表或流数据表，每次计算均会增加一条记录，会产生大量带有相同时间戳的结果。输出表亦不可为键值流数据表，因为键值流数据表不可更新记录。
1.3.6小节使用例子详细介绍了指定updateTime参数后的计算过程。
- useWindowStartTime
可选参数，表示输出表中的时间是否为数据窗口起始时间。默认值为false，表示输出表中的时间为数据窗口起始时间 + windowSize。如果windowSize为多个窗口大小，则必须为fasle。
- roundTime
可选参数，是一个布尔值，表示若数据时间精度为毫秒或者秒且step>一分钟，如何对窗口边界值进行规整处理。默认值为true，表示按照既定的多分钟规则进行规整。若为false，则按一分钟规则进行窗口规整。细节请参阅 1.3.1 小节中有关 step 部分的alignmentSize取值规则。
- snapshotDir
可选参数，表示保存引擎快照的文件目录，可以用于系统出现异常之后，对引擎进行恢复。该目录必须存在，否则系统会提示异常。创建流数据引擎的时候，如果指定了snapshotDir，也会检查相应的快照是否存在。如果存在，会加载该快照，恢复引擎的状态。
- snapshotIntervalInMsgCount
可选参数，表示保存引擎快照的消息间隔。
- fill 'none': 不输出结果。 'null': 输出结果为NULL. 'ffill': 输出上一个有数据的窗口的结果。 '具体数值'：该值的数据类型需要和对应的metrics计算结果的类型保持一致。
可选参数，是一个标量或向量，表示若（某个key的）某个窗口无数据时，如何处理。可取如上值。
fill可以输入向量，为每个metrics指定不同的fill方式。其中，向量中各项只能是'null', 'ffill'或一个数值，不能是'none'。

### 1.3. 参数详细介绍及用例


#### 1.3.1. step

系统按照数据时间精度以及参数step的值确定一个规整尺度alignmentSize，对第一个数据窗口的边界值进行规整处理。
若timeColumn精度为秒，如DATETIME或SECOND类型，alignmentSize取值规则如下表：
| step | alignmentSize |
| --- | --- |
| 0~2 | 2 |
| 3 | 3 |
| 4~5 | 5 |
| 6~10 | 10 |
| 11~15 | 15 |
| 16~20 | 20 |
| 21~30 | 30 |
| 31~60 | 60 |
如果 roundTime = false, 对于step > 60, alignmentSize 都为60。 如果 roundTime = true，则alignmentSize取值规则如下表：
| step | alignmentSize |
| --- | --- |
| 61～120 | 120（2分钟） |
| 121~180 | 180（3分钟） |
| 181~300 | 300（5分钟） |
| 301~600 | 600（10分钟） |
| 601~900 | 900（15分钟） |
| 901~1200 | 1200（20分钟） |
| 1201~1800 | 1800（30分钟） |
| >1800 | 3600（1小时） |
若数据时间精度为毫秒，如TIMESTAMP或TIME类型，alignmentSize取值规则如下表：
| step | alignmentSize |
| --- | --- |
| 0~2 | 2 |
| 3~5 | 5 |
| 6~10 | 10 |
| 11~20 | 20 |
| 21~25 | 25 |
| 26~50 | 50 |
| 51~100 | 100 |
| 101~200 | 200 |
| 201~250 | 250 |
| 251~500 | 500 |
| 501~1000 | 1000（1秒） |
| 1001~2000 | 2000（2秒） |
| 2001~3000 | 3000（3秒） |
| 3001~5000 | 5000（5秒） |
| 5001~10000 | 10000（10秒） |
| 10001~15000 | 15000（15秒） |
| 15001~20000 | 20000（20秒） |
| 20001~30000 | 30000（30秒） |
| 30001~60000 | 60000（1分钟） |
如果 roundTime = false, 对于step > 60000, alignmentSize 都为60000。 如果 roundTime = true，则alignmentSize取值规则如下表：
| step | alignmentSize |
| --- | --- |
| 60001~120000 | 120000（2分钟） |
| 120001~180000 | 180000（3分钟） |
| 120001~300000 | 300000（5分钟） |
| 300001~600000 | 600000（10分钟） |
| 600001~900000 | 900000（15分钟） |
| 900001~1200000 | 1200000（20分钟） |
| 1200001~1800000 | 1800000（30分钟） |
| >= 1800001 | 3600000（1小时） |
DolphinDB系统将各种时间类型数据存储为以最小精度为单位的整形。例如，13:30:10存储为13*60*60+30*60+10=48610。系统将第一个数据窗口的左边界规整为第一条数据时刻之前最后一个可被alignmentSize整除的时刻。
若第一条数据时刻为x，数据类型为TIMESTAMP，那么第一个数据窗口的左边界经过规整后为timestamp(x/alignmentSize*alignmentSize)，其中 / 代表相除后取整。例如，若第一条数据的时间为2018.10.08T01:01:01.365，step为60000，那么alignmentSize为60000，第一个数据窗口的左边界为timestamp(2018.10.08T01:01:01.365/60000*60000)，即2018.10.08T01:01:00.000。
下例说明数据窗口如何规整以及流数据时序引擎如何进行计算。以下代码建立流数据表trades，包含time和volume两列。创建时序引擎streamAggr1，每3毫秒对过去6毫秒的数据计算sum(volume)。time列的精度为毫秒，模拟插入的数据流频率也设为每毫秒一条数据。

```
share streamTable(1000:0, `time`volume, [TIMESTAMP, INT]) as trades
outputTable = table(10000:0, `time`sumVolume, [TIMESTAMP, INT])
tradesAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=6, step=3, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=outputTable, timeColumn=`time)
subscribeTable(tableName="trades", actionName="append_tradesAggregator", offset=0, handler=append!{tradesAggregator}, msgAsTable=true)
```

向流数据表trades中写入10条数据，并查看流数据表trades内容：

```
def writeData(t, n){
    timev = 2018.10.08T01:01:01.001 + timestamp(1..n)
    volumev = take(1, n)
    insert into t values(timev, volumev)
}
writeData(trades, 10)

select * from trades;
```

| time | volume |
| --- | --- |
| 2018.10.08T01:01:01.002 | 1 |
| 2018.10.08T01:01:01.003 | 1 |
| 2018.10.08T01:01:01.004 | 1 |
| 2018.10.08T01:01:01.005 | 1 |
| 2018.10.08T01:01:01.006 | 1 |
| 2018.10.08T01:01:01.007 | 1 |
| 2018.10.08T01:01:01.008 | 1 |
| 2018.10.08T01:01:01.009 | 1 |
| 2018.10.08T01:01:01.010 | 1 |
| 2018.10.08T01:01:01.011 | 1 |
再查看结果表outputTable:

```
select * from outputTable;
```

| time | sumVolume |
| --- | --- |
| 2018.10.08T01:01:01.003 | 1 |
| 2018.10.08T01:01:01.006 | 4 |
| 2018.10.08T01:01:01.009 | 6 |
根据第一条数据时刻规整第一个窗口的起始时间后，窗口以step为步长移动。下面详细解释时序引擎的计算过程。为简便起见，以下提到时间时，省略相同的2018.10.08T01:01:01部分，只列出毫秒部分。基于第一行数据的时间002，第一个窗口的起始时间规整为000，到002结束，只包含002一条记录，计算被003记录触发，sum(volume)的结果是1；第二个窗口从000到005，包含了四条数据，计算被006记录触发，计算结果为4；第三个窗口从003到008，包含6条数据，计算被009记录触发，计算结果为6。虽然第四个窗口从006到011且含有6条数据，但是由于该窗口结束之后没有数据，所以该窗口的计算没有被触发。
若需要重复执行以上程序，应首先解除订阅，并将流数据表trades与时序引擎streamAggr1二者删除：

```
unsubscribeTable(tableName="trades", actionName="append_tradesAggregator")
undef(`trades, SHARED)
dropStreamEngine("streamAggr1")
```


#### 1.3.2. windowSize

DolphinDB时序引擎支持多个窗口。
下例说如何对相同的metrics按不同的windowSize聚合。以下代码建立流数据表trades，包含time和volume两列。创建时序引擎streamAggr1，每3毫秒对过去6毫秒和过去12毫秒的数据计算sum(volume)。time列的精度为毫秒，模拟插入的数据流频率也设为每毫秒一条数据。
再查看结果表outputTable:
| time | sumVolume1 | sumVolume2 |
| --- | --- | --- |
| 2018.10.08T01:01:01.003 | 1 | 1 |
| 2018.10.08T01:01:01.006 | 4 | 4 |
| 2018.10.08T01:01:01.009 | 6 | 7 |
| 2018.10.08T01:01:01.012 | 6 | 10 |
| 2018.10.08T01:01:01.015 | 6 | 12 |
| 2018.10.08T01:01:01.018 | 6 | 12 |
| 2018.10.08T01:01:01.021 | 6 | 12 |

#### 1.3.3. metrics

DolphinDB 时序引擎支持使用多种表达式进行实时计算。
- 一个或多个聚合函数

```
tsAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=6, step=3, metrics=<sum(ask)>, dummyTable=quotes, outputTable=outputTable, timeColumn=`time)
```

- 使用聚合结果进行计算
- 对列与列的操作结果进行聚合计算
- 输出多个聚合结果
- 使用多参数聚合函数
- 使用自定义函数

```
defg diff(x,y){
	return sum(x)-sum(y)
}
tsAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=6, step=3, metrics=<diff(ask, bid)>, dummyTable=quotes, outputTable=outputTable, timeColumn=`time)
```

- 使用多个返回结果的函数

```
defg sums(x){
	return [sum(x),sum2(x)]
}
tsAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=6, step=3, metrics=<sums(ask) as `sumAsk`sum2Ask>, dummyTable=quotes, outputTable=outputTable, timeColumn=`time)
```

注意：不支持聚合函数嵌套调用，例如sum(spread(ask,bid))。

#### 1.3.4. dummyTable

系统利用dummyTable的schema来决定订阅的流数据中每一列的数据类型。dummyTable有无数据对结果没有任何影响。

```
share streamTable(1000:0, `time`volume, [TIMESTAMP, INT]) as trades
modelTable = table(1000:0, `time`volume, [TIMESTAMP, INT])
outputTable = table(10000:0, `time`sumVolume, [TIMESTAMP, INT])
tradesAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=5, step=5, metrics=<[sum(volume)]>, dummyTable=modelTable, outputTable=outputTable, timeColumn=`time)
subscribeTable(tableName="trades", actionName="append_tradesAggregator", offset=0, handler=append!{tradesAggregator}, msgAsTable=true)    

def writeData(t,n){
    timev = 2018.10.08T01:01:01.001 + timestamp(1..n)
    volumev = take(1, n)
    insert into t values(timev, volumev)
}

writeData(trades, 6)

sleep(1)
select * from outputTable
```


#### 1.3.5. outputTable

计算结果可以输出到内存表或流数据表。输出到内存表的数据可以更新或删除，而输出到流数据表的数据无法更新或删除，但是可以通过流数据表将结果作为另一个引擎的数据源再次发布。
下例中，时序引擎electricityAggregator1订阅流数据表electricity，进行移动均值计算，并将结果输出到流数据表outputTable1。时序引擎electricityAggregator2订阅outputTable1表，并对移动均值计算结果求移动峰值。

```
share streamTable(1000:0,`time`voltage`current,[TIMESTAMP,DOUBLE,DOUBLE]) as electricity

//将第一个时序引擎的输出表定义为流数据表，可以再次订阅
share streamTable(10000:0,`time`avgVoltage`avgCurrent,[TIMESTAMP,DOUBLE,DOUBLE]) as outputTable1 

electricityAggregator1 = createTimeSeriesEngine(name="electricityAggregator1", windowSize=10, step=10, metrics=<[avg(voltage), avg(current)]>, dummyTable=electricity, outputTable=outputTable1, timeColumn=`time, garbageSize=2000)
subscribeTable(tableName="electricity", actionName="avgElectricity", offset=0, handler=append!{electricityAggregator1}, msgAsTable=true)

//订阅计算结果，再次进行聚合计算
outputTable2 =table(10000:0, `time`maxVoltage`maxCurrent, [TIMESTAMP,DOUBLE,DOUBLE])
electricityAggregator2 = createTimeSeriesEngine(name="electricityAggregator2", windowSize=100, step=100, metrics=<[max(avgVoltage), max(avgCurrent)]>, dummyTable=outputTable1, outputTable=outputTable2, timeColumn=`time, garbageSize=2000)
subscribeTable(tableName="outputTable1", actionName="maxElectricity", offset=0, handler=append!{electricityAggregator2}, msgAsTable=true);

//向electricity表中插入500条数据
def writeData(t, n){
        timev = 2018.10.08T01:01:01.000 + timestamp(1..n)
        voltage = 1..n * 0.1
        current = 1..n * 0.05
        insert into t values(timev, voltage, current)
}
writeData(electricity, 500);
```


```
select * from outputTable2;
```

| time | maxVoltage | maxCurrent |
| --- | --- | --- |
| 2018.10.08T01:01:01.100 | 8.45 | 4.225 |
| 2018.10.08T01:01:01.200 | 18.45 | 9.225 |
| 2018.10.08T01:01:01.300 | 28.45 | 14.225 |
| 2018.10.08T01:01:01.400 | 38.45 | 19.225 |
| 2018.10.08T01:01:01.500 | 48.45 | 24.225 |
若要对上述脚本进行重复使用，需先执行以下脚本以清除共享表、订阅以及流数据引擎：

```
unsubscribeTable(tableName="electricity", actionName="avgElectricity")
undef(`electricity, SHARED)
unsubscribeTable(tableName="outputTable1", actionName="maxElectricity")
undef(`outputTable1, SHARED)
dropStreamEngine("electricityAggregator1")
dropStreamEngine("electricityAggregator2")
```


#### 1.3.6. keyColumn

下例中，设定keyColumn参数为sym。

```
share streamTable(1000:0, `time`sym`volume, [TIMESTAMP, SYMBOL, INT]) as trades
outputTable = table(10000:0, `time`sym`sumVolume, [TIMESTAMP, SYMBOL, INT])
tradesAggregator = createTimeSeriesEngine(name="streamAggr1", windowSize=3, step=3, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=outputTable, timeColumn=`time, useSystemTime=false, keyColumn=`sym, garbageSize=50)
subscribeTable(tableName="trades", actionName="append_tradesAggregator", offset=0, handler=append!{tradesAggregator}, msgAsTable=true)    

def writeData(t, n){
    timev = 2018.10.08T01:01:01.001 + timestamp(1..n)
    symv =take(`A`B, n)
    volumev = take(1, n)
    insert into t values(timev, symv, volumev)
}

writeData(trades, 6)
```

为了方便观察，对"trades"表的sym列排序输出：

```
select * from trades order by sym;
```

| time | sym | volume |
| --- | --- | --- |
| 2018.10.08T01:01:01.002 | A | 1 |
| 2018.10.08T01:01:01.004 | A | 1 |
| 2018.10.08T01:01:01.006 | A | 1 |
| 2018.10.08T01:01:01.003 | B | 1 |
| 2018.10.08T01:01:01.005 | B | 1 |
| 2018.10.08T01:01:01.007 | B | 1 |

```
select * from outputTable;
```

| time | sym | sumVolume |
| --- | --- | --- |
| 2018.10.08T01:01:01.003 | A | 1 |
| 2018.10.08T01:01:01.006 | A | 1 |
| 2018.10.08T01:01:01.006 | B | 2 |
各组窗口规整后统一从000时间点开始，根据windowSize=3以及step=3，每个组的窗口会按照000-003-006划分。
- (1) 在003，B组有一条数据，但是由于B组在第一个窗口没有任何数据，不会进行计算也不会产生结果，所以B组第一个窗口没有结果输出。
- (2) 004的A组数据触发A组第一个窗口的计算。
- (3) 006的A组数据触发A组第二个窗口的计算。
- (4) 007的B组数据触发B组第二个窗口的计算。
如果进行分组聚合计算，流数据源中的每个分组中的'timeColumn'必须是递增的，但是整个数据源的'timeColumn'可以不是递增的；如果没有进行分组聚合，那么整个数据源的'timeColumn'必须是递增的，否则时序引擎的输出结果会与预期不符。

#### 1.3.7. updateTime

通过以下两个例子，可以理解updateTime的作用。
首先创建流数据表并写入数据：
- 不指定updateTime：

```
output1 = table(10000:0, `time`sym`sumVolume, [TIMESTAMP, SYMBOL, INT])
agg1 = createTimeSeriesEngine(name="agg1", windowSize=60000, step=60000, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=output1, timeColumn=`time, useSystemTime=false, keyColumn=`sym, garbageSize=50, useWindowStartTime=false)
subscribeTable(tableName="trades", actionName="agg1", offset=0, handler=append!{agg1}, msgAsTable=true)

sleep(10)

select * from output1;
```

| time | sym | sumVolume |
| --- | --- | --- |
| 2018.10.08T01:02:00.000 | A | 38 |
| 2018.10.08T01:03:00.000 | A | 25 |
| 2018.10.08T01:02:00.000 | B | 40 |
| 2018.10.08T01:03:00.000 | B | 9 |
- 将updateTime设为1000：

```
output2 = keyedTable(`time`sym,10000:0, `time`sym`sumVolume, [TIMESTAMP, SYMBOL, INT])
agg2 = createTimeSeriesEngine(name="agg2", windowSize=60000, step=60000, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=output2, timeColumn=`time, useSystemTime=false, keyColumn=`sym, garbageSize=50, updateTime=1000, useWindowStartTime=false)
subscribeTable(tableName="trades", actionName="agg2", offset=0, handler=append!{agg2}, msgAsTable=true)

sleep(2010)

select * from output2;
```

| time | sym | sumVolume |
| --- | --- | --- |
| 2018.10.08T01:02:00.000 | A | 38 |
| 2018.10.08T01:03:00.000 | A | 25 |
| 2018.10.08T01:02:00.000 | B | 40 |
| 2018.10.08T01:03:00.000 | B | 9 |
| 2018.10.08T01:05:00.000 | B | 55 |
| 2018.10.08T01:05:00.000 | A | 29 |
下面我们介绍以上两个例子在最后一个数据窗口（01:04:00.000到01:05:00.000）的区别。为简便起见，我们省略日期部分，只列出（小时:分钟:秒.毫秒）部分。假设time列时间亦为数据进入时序引擎的时刻。
(1) 在01:04:04.236时，A分组的第一条记录到达后已经过2000毫秒，触发一次A组计算，输出表增加一条记录(01:05:00.000, `A, 29)。
(2) 在01:04:05.152时的B组记录为01:04:04.412所在小窗口[01:04:04.000, 01:04:05.000)之后第一条记录，触发一次B组计算，输出表增加一条记录(01:05:00.000,"B",32)。
(3) 2000毫秒后，在01:04:07.152时，由于01:04:05.152时的B组记录仍未参与计算，触发一次B组计算，输出一条记录(01:05:00.000,"B",55)。由于输出表的主键为time和sym，并且输出表中已有(01:05:00.000,"B",32)这条记录，因此将该记录更新为(01:05:00.000,"B",55)。

#### 1.3.8. snapshot

通过以下这个例子，可以理解snapshotDir和snapshotIntervalInMsgCount的作用。如果启用snapshot，引擎订阅流数据表时，handler需是appendMsg函数，需指定handlerNeedMsgId=true，用来记录快照的消息位置。

```
share streamTable(10000:0,`time`sym`price`id, [TIMESTAMP,SYMBOL,INT,INT]) as trades
output1 =table(10000:0, `time`sumprice, [TIMESTAMP,INT]);
Agg1 = createTimeSeriesEngine(name=`Agg1, windowSize=100, step=50, metrics=<sum(price)>, dummyTable=trades, outputTable=output1, timeColumn=`time, snapshotDir="/home/server1/snapshotDir", snapshotIntervalInMsgCount=100)
subscribeTable(server="", tableName="trades", actionName="Agg1",offset= 0, handler=appendMsg{Agg1}, msgAsTable=true, handlerNeedMsgId=true)

n=500
timev=timestamp(1..n) + 2021.03.12T15:00:00.000
symv = take(`abc`def, n)
pricev = int(1..n)
id = take(-1, n)
insert into trades values(timev, symv, pricev, id)

select * from output1
```

| time | sumprice |
| --- | --- |
| 2021.03.12T15:00:00.050 | 1225 |
| 2021.03.12T15:00:00.100 | 4950 |
| 2021.03.12T15:00:00.150 | 9950 |
| 2021.03.12T15:00:00.200 | 14950 |
| 2021.03.12T15:00:00.250 | 19950 |
| 2021.03.12T15:00:00.300 | 24950 |
| 2021.03.12T15:00:00.350 | 29950 |
| 2021.03.12T15:00:00.400 | 34950 |
| 2021.03.12T15:00:00.450 | 39950 |
| 2021.03.12T15:00:00.500 | 44950 |

```
getSnapshotMsgId(Agg1)
>499
```

取消订阅并删除引擎来模拟系统异常

```
unsubscribeTable(, "trades", "Agg1")
dropStreamEngine("Agg1")
Agg1=NULL
```

此时发布端仍在写入数据

```
n=500
timev=timestamp(501..1000) + 2021.03.12T15:00:00.000
symv = take(`abc`def, n)
pricev = int(1..n)
id = take(-1, n)
insert into trades values(timev, symv, pricev, id)
```

再次创建aggr, 加载snapshot，从上次处理最后一条消息开始重新订阅

```
Agg1 = createTimeSeriesEngine(name=`Agg1, windowSize=100, step=50, metrics=<sum(price)>, dummyTable=trades, outputTable=output1, timeColumn=`time, snapshotDir="/home/server1/snapshotDir", snapshotIntervalInMsgCount=100)

ofst=getSnapshotMsgId(Agg1)
print(ofst)
>499

subscribeTable(server="", tableName="trades", actionName="Agg1",offset=ofst+1, handler=appendMsg{Agg1}, msgAsTable=true, handlerNeedMsgId=true)

select * from output1
```

| time | sumprice |
| --- | --- |
| 2021.03.12T15:00:00.050 | 1225 |
| 2021.03.12T15:00:00.100 | 4950 |
| 2021.03.12T15:00:00.150 | 9950 |
| 2021.03.12T15:00:00.200 | 14950 |
| 2021.03.12T15:00:00.250 | 19950 |
| 2021.03.12T15:00:00.300 | 24950 |
| 2021.03.12T15:00:00.350 | 29950 |
| 2021.03.12T15:00:00.400 | 34950 |
| 2021.03.12T15:00:00.450 | 39950 |
| 2021.03.12T15:00:00.500 | 44950 |
| 2021.03.12T15:00:00.550 | 25450 |
| 2021.03.12T15:00:00.600 | 5450 |
| 2021.03.12T15:00:00.650 | 9950 |
| 2021.03.12T15:00:00.700 | 14950 |
| 2021.03.12T15:00:00.750 | 19950 |
| 2021.03.12T15:00:00.800 | 24950 |
| 2021.03.12T15:00:00.850 | 29950 |
| 2021.03.12T15:00:00.900 | 34950 |
| 2021.03.12T15:00:00.950 | 39950 |
| 2021.03.12T15:00:01.000 | 44950 |
结果和订阅不中断一样。
| time | sumprice |
| --- | --- |
| 2021.03.12T15:00:00.050 | 1225 |
| 2021.03.12T15:00:00.100 | 4950 |
| 2021.03.12T15:00:00.150 | 9950 |
| 2021.03.12T15:00:00.200 | 14950 |
| 2021.03.12T15:00:00.250 | 19950 |
| 2021.03.12T15:00:00.300 | 24950 |
| 2021.03.12T15:00:00.350 | 29950 |
| 2021.03.12T15:00:00.400 | 34950 |
| 2021.03.12T15:00:00.450 | 39950 |
| 2021.03.12T15:00:00.500 | 44950 |
| 2021.03.12T15:00:00.550 | 25450 |
| 2021.03.12T15:00:00.600 | 5450 |
| 2021.03.12T15:00:00.650 | 9950 |
| 2021.03.12T15:00:00.700 | 14950 |
| 2021.03.12T15:00:00.750 | 19950 |
| 2021.03.12T15:00:00.800 | 24950 |
| 2021.03.12T15:00:00.850 | 29950 |
| 2021.03.12T15:00:00.900 | 34950 |
| 2021.03.12T15:00:00.950 | 39950 |
| 2021.03.12T15:00:01.000 | 44950 |
如果不开启snapshot，即使从上次中断的地方开始订阅，得到的结果也与订阅不中断不一样。
| time | sumprice |
| --- | --- |
| 2021.03.12T15:00:00.050 | 1225 |
| 2021.03.12T15:00:00.100 | 4950 |
| 2021.03.12T15:00:00.150 | 9950 |
| 2021.03.12T15:00:00.200 | 14950 |
| 2021.03.12T15:00:00.250 | 19950 |
| 2021.03.12T15:00:00.300 | 24950 |
| 2021.03.12T15:00:00.350 | 29950 |
| 2021.03.12T15:00:00.400 | 34950 |
| 2021.03.12T15:00:00.450 | 39950 |
| 2021.03.12T15:00:00.500 | 44950 |
| 2021.03.12T15:00:00.550 | 1225 |
| 2021.03.12T15:00:00.600 | 4950 |
| 2021.03.12T15:00:00.650 | 9950 |
| 2021.03.12T15:00:00.700 | 14950 |
| 2021.03.12T15:00:00.750 | 19950 |
| 2021.03.12T15:00:00.800 | 24950 |
| 2021.03.12T15:00:00.850 | 29950 |
| 2021.03.12T15:00:00.900 | 34950 |
| 2021.03.12T15:00:00.950 | 39950 |
| 2021.03.12T15:00:01.000 | 44950 |

## 2. 过滤流数据

使用 subscribeTable 函数时，可利用handler参数过滤订阅的流数据。
在下例中，传感器采集电压和电流数据并实时上传作为流数据源，其中电压voltage<=122或电流current=NULL的数据需要在进入时序引擎之前过滤掉。

```
share streamTable(1000:0, `time`voltage`current, [TIMESTAMP, DOUBLE, DOUBLE]) as electricity
outputTable = table(10000:0, `time`avgVoltage`avgCurrent, [TIMESTAMP, DOUBLE, DOUBLE])

//自定义数据处理过程，过滤 voltage<=122 或 current=NULL的无效数据。
def append_after_filtering(inputTable, msg){
	t = select * from msg where voltage>122, isValid(current)
	if(size(t)>0){
		insert into inputTable values(t.time,t.voltage,t.current)		
	}
}
electricityAggregator = createTimeSeriesEngine(name="electricityAggregator", windowSize=6, step=3, metrics=<[avg(voltage), avg(current)]>, dummyTable=electricity, outputTable=outputTable, timeColumn=`time, garbageSize=2000)
subscribeTable(tableName="electricity", actionName="avgElectricity", offset=0, handler=append_after_filtering{electricityAggregator}, msgAsTable=true)

//模拟产生数据
def writeData(t, n){
        timev = 2018.10.08T01:01:01.001 + timestamp(1..n)
        voltage = 120+1..n * 1.0
        current = take([1,NULL,2]*0.1, n)
        insert into t values(timev, voltage, current);
}
writeData(electricity, 10)
```


```
select * from electricity
```

| time | voltage | current |
| --- | --- | --- |
| 2018.10.08T01:01:01.002 | 121 | 0.1 |
| 2018.10.08T01:01:01.003 | 122 |
| 2018.10.08T01:01:01.004 | 123 | 0.2 |
| 2018.10.08T01:01:01.005 | 124 | 0.1 |
| 2018.10.08T01:01:01.006 | 125 |
| 2018.10.08T01:01:01.007 | 126 | 0.2 |
| 2018.10.08T01:01:01.008 | 127 | 0.1 |
| 2018.10.08T01:01:01.009 | 128 |
| 2018.10.08T01:01:01.010 | 129 | 0.2 |
| 2018.10.08T01:01:01.011 | 130 | 0.1 |

```
select * from outputTable
```

| time | avgVoltage | avgCurrent |
| --- | --- | --- |
| 2018.10.08T01:01:01.006 | 123.5 | 0.15 |
| 2018.10.08T01:01:01.009 | 125 | 0.15 |
由于voltage<=122或current=NULL的数据已经在进入时序引擎时被过滤了，所以第一个窗口[000,003)里没有数据，也就没有发生计算。

## 3. 多个引擎串联使用

outputTable参数除了可以是表之外还可以是其他流数据计算引擎：

```
share streamTable(1000:0, `time`sym`price`volume, [TIMESTAMP, SYMBOL, DOUBLE, INT]) as trades

share streamTable(1000:0, `time`sym`open`close`high`low`volume, [TIMESTAMP, SYMBOL, DOUBLE, DOUBLE, DOUBLE, DOUBLE, INT]) as kline

outputTable=table(1000:0, `sym`factor1, [SYMBOL, DOUBLE])

Rengine=createReactiveStateEngine(name="reactive", metrics=<[mavg(open, 3)]>, dummyTable=kline, outputTable=outputTable, keyColumn="sym")

Tengine=createTimeSeriesEngine(name="timeseries", windowSize=6000, step=6000, metrics=<[first(price), last(price), max(price), min(price), sum(volume)]>, dummyTable=trades, outputTable=Rengine, timeColumn=`time, useSystemTime=false, keyColumn=`sym)
//时间序列引擎的结果输入响应式状态引擎

subscribeTable(server="", tableName="trades", actionName="timeseries", offset=0, handler=append!{Tengine}, msgAsTable=true)
```


## 4. 流数据引擎管理函数

DolphinDB database提供流数据引擎的管理函数，方便查询和管理系统中已经存在的流数据引擎。
- 获取已定义的流数据引擎清单，可使用函数 getStreamEngineStat (deprecated name: getAggregatorStat) 。
- 获取流数据引擎的句柄，可使用函数 getStreamEngine (deprecated name: getAggregator) 。
- 删除流数据引擎，可使用函数 dropStreamEngine (deprecated name: dropAggregator) 。


---

> 来源: `stateful_stream_operators.html`


# 使用 DolphinDB Class 来开发流计算状态算子


## 前言

随着实时数据流处理需求的不断增长，高效、可扩展的流计算框架变得愈发重要。DolphinDB 作为一款高性能分布式时间序列数据库，不仅在数据存储和查询上表现出色，还通过引入面向对象编程（OOP）编程范式，使得开发者能够通过封装、继承、多态等特性，提升代码的灵活性、可维护性和复用性。本文通过两个实际应用案例，详细介绍如何利用 OOP 在 DolphinDB 中开发状态引擎算子，展示 OOP 在流计算中的应用。

## 1. 关于OOP

DolphinDB 从 3.00.0 版本开始支持面向对象编程。面向对象编程（OOP）是非常重要的编程范式，其通过封装、继承、多态等特性，提升代码的灵活性、可维护性和复用性，提升代码的模块化，实现低耦合高内聚。
在 DolphinDB 中，OOP 可应用于多种场景，例如：
- 用于开发流计算状态引擎中的状态算子。在未提供 OOP 时，某些状态算子的开发过程中需要使用复杂的高阶函数，不便于理解代码；某些状态算子需要通过状态函数插件进行开发，开发成本过高。而通过 OOP 编程自定义算子则可以使代码结构清晰，容易理解。
- 在复杂事件处理（CEP）引擎中，可以利用 OOP 定义事件和编写 Monitor。
本教程将主要介绍 OOP 在状态引擎中的应用。

## 2. DolphinDB OOP 编程概要


### 2.1 类的定义


```
class 类名 {
  属性1 :: 类型1
  属性2 :: 类型2
  // ...
  // 和类名同名的方法成为构造函数，有且只有一个
  def 类名(arg1, arg2 /*,...*/) {
    属性1 = arg1
    属性2 = arg2
    // ...
  }
  // 需要注意，成员变量和成员函数不能同名。
  def 方法(arg1, arg2 /*, ...*/) {
    // ...
  }
}
```

例如，我们要定义一个 Person 类，包含两个成员变量：name 和 age。其中，name 是字符串，age 是整数。该类还包含了一个构造函数和 name 成员变量的 getter/setter。

```
class Person {
  // 变量声明在方法声明之前
  name :: STRING
  age :: INT
  // 定义构造函数
  def Person(name_, age_) { // 参数名不能和属性名相同，否则会覆盖属性名
    name = name_
    age = age_
  }
  def setName(newName) {
    name = newName
  }
  def getName() {
    return name
  }
}
```


### 2.2 对象的使用

可以通过 object.method() 的形式调用对象的成员函数；通过 object.member 的形式访问对象的属性。需要注意，和 Python 等脚本语言不同，无法直接通过对 object.member 赋值修改成员变量。如果需要为成员变量赋值，需要创建并使用相应的 setter 方法进行赋值。

```
p = Person("Zhang San", 30)
print(p.getName())
// 调用对象的方法
p.setName("Li Si")
print(p.getName())
// 引用对象的属性
print(p.name)
p.name = "Wang Wu" // 报错：禁止对对象的属性直接赋值
```


### 2.3 对象属性类型标注

定义成员变量的格式为： 成员变量名 :: 类型标注 。
- 标量：包括所有的基本类型，如 INT, DOUBLE, STRING，以及时间类型。
- 向量：如 DOUBLE VECTOR, STRING VECTOR 或者 Array Vector：INT[] VECTOR。
- 其他形式/类型不限：如果是其他类型（如字典、函数等），或者不希望限定变量类型，可以把类型标注写成 ANY。

```
a :: INT
b :: DOUBLE VECTOR
handler :: ANY
```


### 2.4 变量解析

方法中使用到的变量的解析顺序：
- 方法参数
- 对象属性
- 共享变量

```
share table(1:0, `sym`val, [SYMBOL, INT]) as tbl
go
class Test2 {
  a :: INT
  b :: DOUBLE
  def Test2() {
    a = 1
    b = 2.0
  }
  def method(b) {
    print(b) // 解析为函数参数 b；如果需要访问成员变量 b，需要使用 self，见下一小节
    print(a) // 解析为对象属性 a
    print(tbl) // 解析为共享变量 tbl
    print(qwert) // 变量不存在，报错
  }
}
```


### 2.5 self 语法

通过 “self” 变量在类的方法中获取对象本身，其类似于 Python 中的 self，或者 Java、C++ 中的 this。

```
def doSomething(a) {
  // ...
}

class A{
	a :: INT
	b :: INT
	def A() {
		a = 1
		b = 2
	}
	def createCallback() {
		return doSomething{self}
	}
	def nameShadow(b) {
		print(b)
		print(self.b)
	}
}
a = A()
handler = a.createCallback()
a.nameShadow(3)
```


## 3. 应用案例

Reactive State Engine（RSE）是DolphinDB中的一个高性能、可扩展的计算框架，专门用于处理实时流数据。RSE 通过状态算子在数据流中捕捉并维护状态，从而实现增量计算和复杂事件处理。下面我们通过两个案例介绍一下如何利用 OOP 来开发状态引擎算子。

### 3.1 累计求和算子：MyCumSum

在状态引擎内部， cumsum 算子实现了累计求和的功能。这个功能在状态引擎内的实现非常简单，本节不展开说明，仅说明如何利用 OOP 重新实现该算子。
首先，定义一个类 MyCumSum ，并将算子的状态定义为类的成员变量。
其次，在类中实现 append 方法，用于实现累计求和的功能。 append 的参数是逐行输入的数据，返回的结果将作为计算结果输出。
最后，定义一个状态引擎，并指定 MyCumSum.append() 为引擎的算子。向引擎中输入数据，并查看计算结果。

```
class MyCumSum {
  sum :: DOUBLE
  def MyCumSum() {
    sum = 0.0
  }
  def append(value) {
    sum = sum + value
    return sum
  }
}

inputTable = table(1:0, `sym`val, [SYMBOL, DOUBLE])
result = table(1000:0, `sym`res, [SYMBOL, DOUBLE])

rse = createReactiveStateEngine(
          name="reactiveDemo",
          metrics = [<MyCumSum().append(val)>],
          dummyTable=inputTable,
          outputTable=result,
          keyColumn="sym")

data = table(take(`A, 100) as sym, rand(100.0, 100) as val)
rse.append!(data)

select * from data
select * from result
```

进行一次运行，随机生成出来的输入数据和对应输出为：
可以看到，我们编写的 OOP 算子实现了分组的累加求和，与状态引擎内置的 cumsum 算子的功能一致。

### 3.2 线性递归

在未提供 OOP 时，状态引擎计算线性递归时需要通过内置函数 stateIterate 实现。使用 stateIterate 需要指定用于迭代计算的函数，迭代结果和 X 的关联系数，使用的输入数据列，以及初始化窗口的长度，并最终在指定的输出列中输出结果。 stateIterate 的计算规则参考文档： statelterate 。实现代码如下：

```
trade = table(take("A", 6) join take("B", 6) as sym,  1..12 as val0,  take(10, 12) as val1)

inputTable = streamTable(1:0, `sym`val0`val1, [SYMBOL, INT, INT])
outputTable = table(100:0, `sym`factor, [STRING, DOUBLE])
engine = createReactiveStateEngine(
  name="rsTest",
  metrics=<[stateIterate(val0, val1, 3, msum{, 3}, [0.5, 0.5])]>,
  dummyTable=inputTable,
  outputTable=outputTable,
  keyColumn=["sym"],
  keepOrder=true)

engine.append!(trade)
select * from outputTable
```

上例中的 stateIterate(val0, val1, 3, msum{, 3}, [0.5, 0.5]) 基于 stateIterate 的计算规则，实现了线性递归，但仅从这一行代码无法理解其中的计算规则，对用户的使用可能造成困扰。如果通过 OOP 改写此函数的实现逻辑，则代码结构会清晰很多：
可以看到，当窗口长度小于3时，算子直接返回init中的结果；当窗口长度大于等于3时，算子的append方法将长度为3的窗口中数据的和与X中的值做加权平均；最终实现了和 stateIterate(val0, val1, 3, msum{, 3}, [0.5, 0.5]) 相同的功能。
虽然通过 OOP 实现线性递归的代码行数有所增加，但它结构清晰，提高了代码的可读性，同时简化了调试过程。

## 4. 小结和展望

通过本教程的两个应用案例，我们可以看到如何在 DolphinDB 中利用 OOP（面向对象编程）开发状态引擎算子。这种方式相比直接调用引擎提供的算子，具有可读性强、结构清晰的优点。在当前实现中，DolphinDB的 OOP 仍然采用解释执行的方式，相比原生 C++ 实现速度较慢。
目前，在响应式状态引擎中，一些有状态的高阶函数迭代算子和无状态的自定义函数算子已经可以通过即时编译（JIT）技术优化，用脚本编写的自定义函数算子的性能可以达到原生 C++ 实现的水平；但是，目前状态引擎中使用 OOP 编写的有状态的算子还没有支持 JIT 。未来，我们计划使用 JIT 技术进一步优化响应式状态引擎中的 OOP 的应用，直接将类中定义的有状态算子编译为机器码运行，以期望实现在当前较高的开发效率下也不会损失运行时的效率。


---

> 来源: `getting_started_with_cep_engine.html`


# CEP 引擎入门：初级高频量价因子策略的实现

- 编程语言
- 复杂事件处理（CEP）引擎官方文档
- 响应式状态引擎
- 流数据表
- CEP 引擎白皮书
高频交易（High-Frequency Trading, HFT）作为现代金融市场中的重要组成部分，以其高速、自动化和复杂的算法交易策略而著称。高频交易策略通过分析大量实时变化的市场数据，利用市场的微小价格波动迅速做出交易决策，从而在极短的时间内获取利润。这样的需求对数据基础设施提出了极高的要求，而DolphinDB 作为一款高性能分布式时序数据库，可以凭借其高效的计算能力和丰富的功能模块辅助高频策略的投研。
在本篇文章中，我们将详细介绍 DolphinDB 流数据框架中的复杂事件处理（Complex Event Processing, 下称CEP ）引擎。CEP 引擎能够从大规模和复杂的实时数据流中提取事件信息，对符合预定模式的事件进行实时的分析和决策，解决普通流数据处理无法满足的复杂需求。在本文中，我们将通过两个例子逐步介绍如何使用 CEP 引擎处理事件流数据。

## 1. 关键概念

在处理事件以及复杂事件数据前，首先要了解一些关于 CEP 引擎最基础、最核心的概念。这些概念将贯穿整个 CEP 引擎的使用过程。例如，在 DolphinDB 的框架下，什么是事件和复杂事件？

### 1.1 事件（Event） 和复杂事件（Complex Event）

在 CEP 系统中， 事件是一段时间内发生的特定事情或状态的数据表示，是描述对象变化的属性值的集合 。例如“以170.85美元的价格买入1股 AAPL 股票”和“BRK 在2024年6月12日上涨5%”都是事件。它源源不断地产生，并以流的形式存在。
在 CEP 系统中，定义一个事件时需要指定事件类型及其包含的每个属性的名称和类型。在 DolphinDB 中，使用类（Class）来定义事件，类名为事件类型，类的对象实例是一次具体的事件。同时，在 DolphinDB 中可以用流数据表来存储事件流，流数据表中的每一行记录存储了一次具体的事件。
通常，用户在事件流中除了查找并处理特定的事件外，还会对事件进行额外的规则限制。复杂事件（Complex Event）不仅仅指某一个事件类型，还往往涉及复杂的事件匹配规则。常见的匹配规则有：
- 事件顺序规则：如“MACD 出现金叉后 30 秒内 CCI 进入超买区间”
- 事件属性值规则：如“某一成交事件中，成交量超过 50000 股”
- 时间规则：如“收到 A 股票成交回报后1分钟内不再对 A 进行下单”
在 DolphinDB 中，主要通过 addEventListener 函数的各项参数来表达丰富的匹配规则。

### 1.2 监视器（Monitor）

监视器是 CEP 系统中 负责监控并响应事件 的组件，它包含了完整的业务逻辑。
在 DolphinDB 中，使用类（Class）来定义监视器。监视器在创建 CEP 引擎（ createCEPEngine ）时被实例化。一个监视器中主要包含以下内容：
- 一个或多个事件监听器（Event Listener)。事件监听器用于捕捉特定规则的事件。
- 一个或多个回调函数。回调函数是在捕捉到特定事件后执行的操作。
- 一个 onload 函数。 onload 函数可以被视为初始化监视器的操作，通常包括启动一个或多个事件监听器。该函数仅在监视器被实例化时被执行一次。

### 1.3 事件监听器（Event Listener）

事件监听器包含了需要捕捉的特定事件规则，并能够 进行实时地规则匹配，一旦条件满足就会触发预设的操作 。这些条件可以是简单的数据阈值判断，也可以是复杂的模式匹配（序列模式、计时器等）。
在 DolphinDB 中，执行监视器中的 addEventListener 函数即可启动特定的事件监听器。通过 addEventListener 函数不同的参数配置可以灵活地描述匹配规则。
假设 CEP 引擎具体的使用情景为量化交易，则监视器（Monitor）可以被视为一个包含一个或多个交易逻辑的容器，而事件监听器（Event Listener）则是容器中用于检测特定市场条件的组件。在实际应用中，一个监视器可能包含多个事件监听器，每个监听器负责监测一种特定类型的市场活动或条件。当所有相关条件都满足时，交易策略（即监视器）将执行预定的交易动作，如买入、卖出或调整仓位。

## 2. 案例 I ：监听股价并记入日志

我们先在一个简单的场景中应用 CEP 引擎。假设现在希望把实时的市场行情记录到日志中，应该如何通过 CEP 引擎实现呢？

### 2.1 定义事件类

在这个初级案例中，我们将对代码进行详细的解释，以帮助用户理解代码逻辑。而在后续的案例中，会减少一些不必要的代码解释。
CEP 引擎的处理对象为事件，因此要先定义引擎内需要的事件。此处定义了一个名为 StockTick 的 事件类 ，这个类包含股票的名称（name）和价格（price）两个属性，用来代表市场上股票的价格变动信息。

```
//定义一个事件类，表示每个tick的股票行情事件
class StockTick{
    name :: STRING 
    price :: FLOAT 
    def StockTick(name_, price_){
        name = name_
        price = price_
    }
}
```

- class StockTick ：定义一个名为 StockTick 的类，用来表示从市场数据源接收的股票行情。
- name :: STRING ：在类中定义一个名为 name 的字符串属性，用于存储股票的名称。
- price :: FLOAT ：在类中定义一个名为 price 的浮点属性，用于存储股票的价格。
- def StockTick(name_, price_) ：类的构造函数，用于实例化一个 StockTick 对象。

### 2.2 定义监视器和设置监听

其次，定义一个名为 SimpleShareSearch 的监视器类，该监视器通过变量 newTick 存储最新接收到的 StockTick 事件。同时，监视器在启动时会调用 onload 函数，随即注册一个事件监听器，它会捕获所有的 StockTick 事件，并在回调函数 processTick 中处理这些事件。处理过程中，监视器将接收到的股票信息记录到日志中。

```
class SimpleShareSearch : CEPMonitor {
	//保存最新的 StockTick 事件
	newTick :: StockTick 
	def SimpleShareSearch(){
		newTick = StockTick("init", 0.0)
	}
	def processTick(stockTickEvent)
	
	def onload() {
		addEventListener(handler=processTick, eventType="StockTick", times="all")
	} 
	def processTick(stockTickEvent) { 
		newTick = stockTickEvent
		str = "StockTick event received" + 
			" name = " + newTick.name + 
			" Price = " + newTick.price.string()
		writeLog(str)
	}
}
```

- class SimpleShareSearch : CEPMonitor ：定义一个名为 SimpleShareSearch 的监视器类。需要注意的是，监视器类在创建时需要继承 CEPMonitor类。
- newTick :: StockTick ：定义一个属性 newTick，类型为 StockTick，用于存储最新接收到的 StockTick 事件。
- def SimpleShareSearch() ：类的构造函数。
- def onload() ： 监视器必须包含的方法 ，当 CEP 引擎被创建时将会实例化监视器，这将调用 onload 方法对监视器进行初始化。在本例中启动了一个事件监听器 。
- addEventListener(handler=processTick, eventType="StockTick", times="all") ：该事件监听器在 onload 中注册，在创建引擎时立刻开始监听所有类型为 StockTick 的事件，并对所有 StockTick 事件使用 processTick 方法处理。times 参数的默认设置为 "all"，表示监听并响应每一次匹配的事件。如果 times=1，事件监听器将在找到一个对应事件后终止 。更为详细的参数设置规则可以参考 相应文档 。
- def processTick(stockTickEvent) ：定义事件处理方法，接收一个 StockTick 类型的事件作为参数。类的成员方法需要至少先声明或者定义之后才能被调用，因此需要先声明 processTick 方法以防止 onload 中的 addEventListener 函数调用报错。在收到 StockTick 事件之后执行 processTick ，将在日志中记录 StockTick 相关信息。
- newTick = stockTickEvent ：更新监视器中的属性 newTick 为最新接收到的事件。在本例中，这一操作并不是必选的，仅仅为了展示监视器中可以存储运行过程中的变量这一功能。例如，基于此功能可以扩展本例为同时打印出上一条行情与最新的行情。

### 2.3 创建 CEP 引擎

创建一个 CEP 引擎，配置好相关的表结构和监视器，以便引擎能够接收和处理 StockTick 类型的事件。

```
dummyTable = table(array(STRING, 0) as eventType, array(BLOB, 0) as eventBody)
try {dropStreamEngine(`simpleMonitor)} catch(ex) {}
createCEPEngine(name="simpleMonitor", monitors=<SimpleShareSearch()>, 
dummyTable=dummyTable, eventSchema=[StockTick])
```

- dummyTable ：包含 eventType 和 eventBody 两列的表作为引擎的输入表，这个表将被用作 CEP 引擎创建时的 dummyTable 参数。在 2.4 小节将进一步介绍该参数的含义。
- try {dropStreamEngine(`simpleMonitor)} catch(ex) {} ：这是 CEP 引擎脚本中常见的清理环境方法。若已存在同名引擎，则引擎无法被成功创建，因此在创建引擎前尝试删除叫 simpleMonitor 的流引擎，并捕获可能因为引擎并不存在而返回的异常。
- createCEPEngine ：创建 CEP 引擎，命名为 simpleMonitor。同时，指定引擎将使用 SimpleShareSearch 类的实例作为其监视器，dummyTable 为输入表，最后指定引擎将输入表中的数据解析为 StockTick 事件类。

### 2.4 向引擎输入模拟数据

在 DolphinDB 中可以通过两种不同的方法，向 CEP 引擎注入事件：
- appendEvent(engine, events) ：直接将事件实例写入 CEP 引擎的方法。
- append! / tableInsert / insert into ：通过向 CEP 引擎写入一张表的方式来向引擎输入数据。这种方式在事件注入引擎之前，需要先将其序列化为 BLOB 格式写入异构流数据表中，再通过上述几个函数之一向引擎输入数据。因此，在上文中创建的 CEP 引擎输入表 dummyTable 中进行了相应的设置，eventType 列储存事件类型，eventBody 列储存对应事件的序列化结果。该方法的好处是可以将多个不同的事件类型写入到一个异构流数据表中。然后通过订阅该异构流数据表，向引擎输入事件数据。
向 CEP 引擎输入数据，查看日志并检查引擎运行结果是否符合预期。

```
stockTick1 = StockTick('600001',6.66)
getStreamEngine(`simpleMonitor).appendEvent(stockTick1)
stockTick2 = StockTick('300001',1666.66)
getStreamEngine(`simpleMonitor).appendEvent(stockTick2)
```

- 在上面的代码中，模拟了2个 StockTick 事件。事件1的股票代码为600001，股价为6.66；事件2的股票代码为300001，股价为1666.66元。
- 通过 getStreamEngine 函数返回名为"simpleMonitor"的 CEP 引擎的句柄，然后通过 appendEvent 方法直接向 CEP 引擎写入两个事件实例。
- 在 DolphinDB 的 Web 端选择日志查看并在页面右上角刷新，即可看到 CEP 引擎已经识别了输入的事件，并且执行了写入日志的操作。

### 2.5 实现流程

- 定义 CEP 引擎将要监听或处理的事件类。在本例中，定义了一个有名称和价格两个属性的 StockTick 类作为监控的对象，用来代表市场上股票的价格变动信息。
- 定义一个监视器类，其中必须包含 onload 方法，该方法将在创建 CEP 引擎时被首先调用。本例中 onload 方法中调用了 addEventListener 函数，即在引擎创建时就启动监听，并在每一次监听到对应的事件后交由指定回调函数处理。
- 定义回调函数，即处理事件的方法。本例中该函数将股票价格变动写入了日志。
- 创建 CEP 引擎。在本例中，创建引擎时对命名了引擎，指定了引擎的监视器（monitors）、输入表结构（dummyTable）和要处理的事件类型（eventSchema）。
- 模拟事件数据并输入引擎，查看日志检查结果是否符合设计预期。
至此，便实现了对市场行情的实时监控，并且以一定的格式记录到了日志中。通过这种方式，DolphinDB 的 CEP 引擎提供了一种高效的机制来实时监测和响应事件流中的事件，使得用户可以根据实时数据变动快速地进行应对，而上文仅仅是一个简单的金融领域的例子。
在上面的例子中，我们对 CEP 引擎和它的关键概念、基本的代码运行逻辑有了初步的了解，并且可以实现单一事件流监控并记录日志的操作。但这样的单一类型流数据计算和监控用 DolphinDB 已有的流计算框架也能够实现，那么还为什么需要额外的 CEP 引擎呢？
因为上文中的案例不涉及复杂的事件匹配规则。当所需要处理的事件类型越多、越复杂，所需监控的事件模式和特定模式下需执行的响应操作越多、越复杂，则在 CEP 引擎框架下，通过 DolphinDB 脚本直接进行事件描述和匹配等操作进行的程序开发效率就越高。CEP 引擎既保留了流计算引擎高吞吐、低时延、流批一体等强大优势，又在灵活性与事件描述语言等方面进行了改进。

## 3. 案例 II ：初级高频量价因子策略实现

在金融高频量化场景下，CEP 引擎还可以为用户实现涉及多种事件的复杂交易策略，如套利交易、组合交易等策略，还可以进行风险控制、可视化监控等业务。我们可以再从一个初级的事件化量价因子驱动的交易策略开始，进一步感受 CEP 引擎的灵活性。

### 3.1 策略逻辑

基于股票逐笔成交数据，策略将根据每支股票最新的成交价涨幅和累计成交量判断是否执行下单或撤单等操作，具体流程如下所示。
具体的策略判断逻辑细节如下：
- 根据每一笔成交数据触发计算两个实时因子：最新成交价相对于15秒内最低成交价的涨幅 （变量 ROC）和过去1分钟的累计成交量 （变量 volume ）；
- 在策略启动时设定每支股票的两个因子阈值（ ROC0、volume0 ），每当实时因子值更新后判断是否 ROC > ROC0 且 volume > volume0。 若是，则触发下单； 若下单后1分钟内仍未成交，则触发对应的撤单。
比案例 I 更进一步之处
- 在 CEP 引擎内使用了 响应式状态引擎 （Reactive State Engine, RSE）计算因子。 响应式状态引擎支持实时数据的高性能增量计算。在本例中的 CEP 引擎内部，使用了响应式状态引擎及其内置的状态函数对两个因子值进行计算。
- 向外部系统发送事件。 在策略执行的过程中，当满足某些具体的条件时，CEP 引擎内部需要将特定事件（本例中为下单和撤单事件）通过 emitEvent 接口发送到 CEP 引擎外部。
- 使用了超时计时器。 在 addEventListener 函数中有关于事件监听器触发方式和触发次数的可选参数设置。通过这些可选参数可以实现不同的监听触发效果，如事件匹配、计时器、定时器等。本例在每次下单后开启一个新的计时器，若计时超时则触发对应操作。

### 3.2 代码实现

在本小节中将详细介绍案例的代码实现，完整代码附件见文末附录。

#### 3.2.1 定义事件类


```
//  定义股票逐笔成交事件
class StockTick {
	securityid :: STRING 
	time :: TIMESTAMP
	price ::  DOUBLE
	volume :: INT
	def StockTick(securityid_, time_, price_, volume_) {
		securityid = securityid_
		time = time_
		price = price_
		volume = volume_
	}
}
// 定义成交回报事件
class ExecutionReport { 
	orderid :: STRING 
	securityid :: STRING 
	price :: DOUBLE 
	volume :: INT
	def ExecutionReport(orderid_, securityid_, price_, volume_) {
		orderid = orderid_
		securityid = securityid_
		price = price_
		volume = volume_
	}
}
// 定义下单事件
class NewOrder { 
	orderid :: STRING 
	securityid :: STRING 
	price :: DOUBLE 
	volume :: INT
	side :: INT
	type :: INT
	def NewOrder(orderid_, securityid_, price_, volume_, side_, type_) { 
		orderid = orderid_
		securityid = securityid_
		price = price_
		volume = volume_
		side = side_
		type = type_
	}
}
// 定义撤单事件
class CancelOrder { 
	orderid :: STRING 
	def CancelOrder(orderid_) {
		orderid = orderid_
	}
}
```


#### 3.2.2 定义监视器和设置监听

定义一个监视器 StrategyMonitor，在此处封装整体的交易策略。一些策略相关的属性和变量，如策略编号和参数，可以在定义监视器类时进行定义和初始化，整体监视器的结构大致如下。

```
class StrategyMonitor : CEPMonitor { 
	strategyid :: INT // 策略编号
	strategyParams :: ANY // 策略参数：策略标的、标的参数配置	
	dataview :: ANY // Data View 监控	
	def StrategyMonitor(strategyid_, strategyParams_) {
		strategyid = strategyid_
		strategyParams = strategyParams_
	}
	def execReportExceedTimeHandler(orderid, exceedTimeSecurityid)
	def execReportHandler(execReportEvent)
	def handleFactorCalOutput(factorResult)
	def tickHandler(tickEvent)
	def initDataView()
	def createFactorCalEngine()
	def onload(){
		initDataView()
		createFactorCalEngine()
		securityids = strategyParams.keys()
		addEventListener(handler=tickHandler, eventType="StockTick", 
		condition=<StockTick.securityid in securityids>, times="all")
	}
}
```

- 在创建 CEP 引擎时，将首先调用 onload 方法，因此代码的解读也从该方法开始。本案例中， onload 调用了 initDataView 和 createFactorCalEngine 方法，并启动对 StockTick 事件的监听。其中，initDataView 方法将在第4节单独讲解，其主要功能是为 Web 端策略运行监控的数据视图更新指定的数据键值，并不是策略重点，因此不在本章详细讲解。
- 在 createFactorCalEngine 方法中，对监听到的事件数据进行因子计算后，将计算得到的结果交由 handleFactorCalOutput 方法处理。
- 在 handleFactorCalOutput 方法中启动了两个事件监听器，同时监听成交回报事件，但是设置两个不同的触发条件：
- 当60秒内没有成交事件时，则触发 execReportExceedTimeHandler 方法进行撤单；
- 每次成交事件发生时，触发 execReportHandler 将查询订单号和计算成交金额进行更新。
函数调用流程示意图如下所示。
创建因子计算引擎 ： createFactorCalEngine 方法具体包含相应的引擎创建和计算逻辑设定，同时规定了数据输入表 dummyTable 和输出表 factorResult 的结构、引擎的计算逻辑 metrics（计算最新成交价/15秒内最低成交价的值、1分钟累计成交量的值和最新成交价的值），创建了名叫 factorCal 的响应式计算引擎。

```
def createFactorCalEngine(){
	dummyTable = table(1:0, `securityid`time`price`volume, 
	`STRING`TIMESTAMP`DOUBLE`INT)
	metrics = [<(price\tmmin(time, price, 15s)-1)*100>, <tmsum(time, volume, 60s)>, 
	<price> ] // 最新成交价相对于15秒内最低成交价涨幅 ,1分钟累计成交量, 最新成交价
	factorResult = table(1:0, `securityid`ROC`volume`lastPrice, 
	`STRING`INT`LONG`DOUBLE) 
	createReactiveStateEngine(name="factorCal", metrics=metrics , 
	dummyTable=dummyTable, outputTable=factorResult, keyColumn=`securityid, 
	outputHandler=handleFactorCalOutput, msgAsTable=true)		
}
```

- 创建响应式状态引擎时，指定参数 outputHandler=handleFactorCalOutput ，意味着引擎计算结束后，不再将计算结果写到输出表（即使定义了输出表的结构），而是会调用 handleFactorCalOutput 方法处理计算结果。同时，设置参数 msgAsTable=true 表示引擎的计算结果将以表的形式呈现，且计算结果表的结构与引擎参数 outputTable 指定的表 factorResult 的格式一致。
- 在上述代码中创建的响应式状态引擎将接收与 dummyTable 格式一致的表数据作为输入，按照 metrics 指定的计算方法进行计算，最后把与输出表格式相同的计算结果传递给 handleFactorCalOutput 进行处理。

```
def handleFactorCalOutput(factorResult){
	factorSecurityid = factorResult.securityid[0]
	ROC = factorResult.ROC[0]
	volume = factorResult.volume[0]
	lastPrice = factorResult.lastPrice[0] 
	updateDataViewItems(engine=self.dataview, keys=factorSecurityid, 
	valueNames=["ROC","volume"], newValues=(ROC,volume))
	if (ROC>strategyParams[factorSecurityid][`ROCThreshold] 
	&& volume>strategyParams[factorSecurityid][`volumeThreshold]) {
		orderid = self.strategyid+"_"+factorSecurityid+"_"+long(now())
		newOrder = NewOrder(orderid , factorSecurityid, lastPrice*0.98, 100, 'B', 0) 
		emitEvent(newOrder) // 发送下单事件到外部
		newOrderNum = (exec newOrderNum from self.dataview where 
		securityid=factorSecurityid)[0] + 1
		newOrderAmount = (exec newOrderAmount from self.dataview where 
		securityid=factorSecurityid)[0] + lastPrice*0.98*10
		updateDataViewItems(engine=self.dataview, keys=factorSecurityid, 
		valueNames= ["newOrderNum", "newOrderAmount"], 
		newValues=(newOrderNum, newOrderAmount)) // 更新data view			
		addEventListener(handler=self.execReportExceedTimeHandler{orderid, 
		factorSecurityid}, eventType="ExecutionReport", 
		condition=<ExecutionReport.orderid=orderid>, times=1, exceedTime=60s) 
		addEventListener(handler=execReportHandler, eventType="ExecutionReport", 
		condition=<ExecutionReport.orderid=orderid>, times="all") // 启动成交回报监听
	}
}
```


```
def execReportExceedTimeHandler(orderid, exceedTimeSecurityid){
	emitEvent(CancelOrder(orderid)) // 发送撤单事件到外部
	timeoutOrderNum = (exec timeoutOrderNum from self.dataview 
	where securityid=exceedTimeSecurityid)[0] + 1
	updateDataViewItems(engine=self.dataview, keys=exceedTimeSecurityid, 
	valueNames=`timeoutOrderNum, newValues=timeoutOrderNum) // 更新data view
}
def execReportHandler(execReportEvent) {
	executionAmount = (exec executionAmount from self.dataview 
	where securityid=execReportEvent.securityid)[0] + 
	execReportEvent.price*execReportEvent.volume
	executionOrderNum = (exec executionOrderNum from self.dataview 
	where securityid=execReportEvent.securityid)[0] + 1
	updateDataViewItems(engine=self.dataview, keys=execReportEvent.securityid, 
	valueNames=["executionAmount","executionOrderNum"], 
	newValues=(executionAmount,executionOrderNum)) // 更新data view		
}
```

启动事件监听器 ： onload 中初始化了事件监听器监听 StockTick 事件，并且在事件的股票 id 与策略设定需监听的股票 id 相符合时，调用指定的 tickHandler 方法。

```
def tickHandler(tickEvent){
	factorCalEngine = getStreamEngine(`factorCal)
	insert into factorCalEngine values([tickEvent.securityid, 
	tickEvent.time, tickEvent.price, tickEvent.volume])
}
```

- tickHandler 方法中，用 getStreamEngine 取得因子计算引擎的句柄后，向引擎插入数据。数据为被事件监听器捕获的符合条件的 StockTick 事件中的股票 id、时间和量价数据。

#### 3.2.3 创建 CEP 引擎


```
dummy = table(array(STRING, 0) as eventType, array(BLOB, 0) as eventBody)
share(streamTable(array(STRING, 0) as eventType, array(BLOB, 0) as eventBody, 
array(STRING, 0) as orderid), "output")
outputSerializer = streamEventSerializer(name=`serOutput, 
eventSchema=[NewOrder,CancelOrder], outputTable=objByName("output"), 
commonField="orderid")
strategyid = 1
strategyParams = dict(`300001`300002`300003, 
[dict(`ROCThreshold`volumeThreshold, [1,1000]), 
dict(`ROCThreshold`volumeThreshold, [1,2000]), 
dict(`ROCThreshold`volumeThreshold, [2, 5000])])
engine = createCEPEngine(name='strategyDemo', monitors=<StrategyMonitor(strategyid, 
strategyParams)>, dummyTable=dummy, eventSchema=[StockTick,ExecutionReport], 
outputTable=outputSerializer)
```

- 通过 createCEPEngine 方法创建一个 CEP 引擎。
- 在创建引擎时，指定 StrategyMonitor 类作为 CEP 引擎的监视器，并传入 strategyid 和 strategyParams 作为实例化监视器方法的入参。
- 规定引擎的输入表 dummy 的结构，该表有 eventType 列表示事件类型，eventBody 列表示事件序列化的结果；创建一个叫 output 的共享流数据表作为 CEP 引擎的输出结果表。
- 若进入引擎的数据为序列化数据，则它将按照 CEP 引擎的 eventSchema 规定的事件形式进行反序列化。本案例中使用 appendEvent 接口直接向引擎输入事件实例，因此数据进入 CEP 引擎时不涉及数据的反序列化。
- 在上文中提到，如果监视器中调用了 emitEvent 接口，则需要指定 outputTable 参数为序列化器 StreamEventSerializer 的句柄。因此，此处需要创建一个事件序列化器，并通过它将接收到的事件序列化为 BLOB 格式，写入异构流数据表 output 中。序列化器将按对应的 eventSchema 指定的键的顺序把事件序列化为对应的值（此处的 eventSchema 为序列化器的参数）。在本例中，这个序列化处理器将接收 emitEvent 发送出来的事件实例并进行处理，因此序列化器的 eventSchema 事件参数应该设置为发送出 CEP 引擎的事件。

#### 3.2.4 向引擎输入模拟数据

构造 StockTick 事件实例，并且通过 appendEvent 输入CEP 引擎。

```
ids = `300001`300002`300003`600100`600800
for (i in 1..120) {
    sleep(500)
    tick = StockTick(rand(ids, 1)[0], now()+1000*i, 
10.0+rand(1.0,1)[0], 100*rand(1..10, 1)[0])
    getStreamEngine(`strategyDemo).appendEvent(tick)
}
```

- 定义了一组股票代码，生成120条逐笔成交数据。每次循环中创建一个 StockTick 实例模拟股票的逐笔成交数据，包含股票代码、成交时间、成交价格和成交量。将生成的 StockTick 事件发送到 CEP 引擎中，以便处理这些模拟的成交数据。

```
sleep(1000*20)
print("begin to append ExecutionReport")
for (orderid in (exec orderid from output where eventType="NewOrder")){
    sleep(250)
    if(not orderid in (exec orderid from output where eventType="CancelOrder")) {
        execRep = ExecutionReport(orderid, split(orderid,"_")[1], 10, 100)
        getStreamEngine(`strategyDemo).appendEvent(execRep) 
    }   
}
```

- sleep(1000*20) 是为了模拟已经有挂单，但还没成交的场景。
- 遍历所有的新订单，查询从输出表中获取的所有新订单 id。检查当前订单 id 是否未被撤销。若未被撤销，则生成一个成交回报事件。
执行代码，查看结果表 output 如下所示。其中 eventType 展示了被发送到输出事件队列的事件类型，eventBody 为事件序列化的结果。 orderid 并不是必选的，这里作为单个订单事件唯一的标识符。
- 事件实例进入引擎，经由监视器中的事件监听器进行监听和触发相应的回调函数。
- 随后，通过 emitEvent 方法将事件插入到引擎事件输出队列的尾部。
- 这些事件将会被发送到 outputTable 参数指定的 streamEventSerializer 返回的序列化器对象 outputSerializer 中进行序列化。
- 最后将序列化后的事件数据输出到异构流数据表 output 中，供各种 API 接口进行订阅和反序列化操作。 ​

## 4. 数据视图（DataView）

在案例 II 中，为了可以方便地监控策略运行时一些数据的最新状态，在初始化监视器时的 onload 方法中创建了一个数据视图引擎（DataView Engine）。本章介绍案例 II 中具体的数据视图创建和更新的代码。
数据视图（DataView） 是在某一时刻对特定数据的快照。在 CEP 系统中，DataView 用于持续追踪和显示 CEP 引擎处理的中间变量或监控值的最新状态。这些监控值可能是由事件触发并在事件处理过程中动态计算和更新的，例如交易量、价格等随事件不断变化的量。请注意，数据视图页面仅展示监控值的最新快照，不支持查看历史快照。若需要查看监控值的历史快照和趋势变化图，请使用 数据面板 功能（Dashboard）。
数据视图引擎（ DataView Engine ） 是一个特殊的 DolphinDB 引擎，负责维护和管理一个或多个 DataView 。它允许 CEP 引擎在运行过程中将监控值写入到这些 DataView 中。引擎保存每个监控值的最新快照，并负责将这些数据输出到目标表（通常是流数据表），使得这些数据可以被其他程序订阅或查询。在 CEP 系统中，可以创建多个 DataView Engine，并在创建时指定需要维护的监控值名称。

### 4.1 初始化数据视图


```
def onload() {
    // 初始化 data view
    initDataView()
    ......  
}
```

- 在 onload 中调用初始化 DataView 的函数，然后在监视器内完善 initDataView 方法。

```
def initDataView(){
    share(streamTable(1:0, `securityid`strategyid`ROCThreshold
`volumeThreshold`ROC`volume`newOrderNum`newOrderAmount`executionOrderNum
`executionAmount`timeoutOrderNum`updateTime, 
`STRING`INT`INT`INT`INT`INT`INT`DOUBLE`INT`DOUBLE`INT`TIMESTAMP), "strategyDV")
    dataview = createDataViewEngine(name="Strategy_"+strategyid, 
outputTable=objByName(`strategyDV), keyColumns=`securityId, timeColumn=`updateTime) 
    num = strategyParams.size()
    securityids = strategyParams.keys()
    ROCThresholds = each(find{,"ROCThreshold"}, strategyParams.values())
    volumeThresholds = each(find{,"volumeThreshold"}, strategyParams.values()) 
    dataview.tableInsert(table(securityids, take(self.strategyid, num) as 
strategyid, ROCThresholds, volumeThresholds, take(int(NULL), num) as ROC, 
take(int(NULL), num) as volume, take(0, num) as newOrderNum, 
take(0, num) as newOrderAmount, take(0, num) as executionOrderNum, 
take(0, num) as executionAmount, take(0, num) as timeoutOrderNum))
}
```

- 函数 initDataView 首先创建并共享了一个叫”strategyDV”的流数据表，用作 DataViewEngine 的输出表。该表规定了输出表的结构，指定了希望监控的相应数据。
- 通过 createDataViewEngine 创建了引擎，设定了引擎键值，对于每个键值，引擎都只保留最新的 1 条数据。
- find 函数从每个股票的策略参数字典中查找 ROCThreshold 和 volumeThreshold 键的值。
- each 函数把 find 函数应用到每个股票的策略参数字典上，为每个股票提取出策略因子的阈值。
- 最后将提取的阈值和初始状态值插入到 DataView 的表中。

### 4.2 更新数据视图指定键值的数据

updateDataViewItems 可以更新 DataView 引擎中指定键值对应的指定属性的值。它被多次用在不同的方法中。
- 更新超时订单数量：当一个成交回报事件超时未收到时，函数会增加超时订单的数量并更新相应的 DataView 。
- 更新成交金额和订单数量：先计算给定 securityid 的成交金额和订单数量，再用 updateDataViewItems 方法更新它们。

```
def execReportHandler(execReportEvent) {
	executionAmount = (exec executionAmount from self.dataview 
	where securityid=execReportEvent.securityid)[0] + 
	execReportEvent.price*execReportEvent.volume
	executionOrderNum = (exec executionOrderNum from self.dataview 
	where securityid=execReportEvent.securityid)[0] + 1
	updateDataViewItems(engine=self.dataview, keys=execReportEvent.securityid, 
	valueNames=["executionAmount","executionOrderNum"], 
	newValues=(executionAmount,executionOrderNum)) // 更新data view		
}
```

- 更新 ROC 和成交量数据：当因子计算引擎输出结果时，更新相关的 ROC 和成交量数据。

```
def handleFactorCalOutput(factorResult){
    factorSecurityid = factorResult.securityid[0]
    ROC = factorResult.ROC[0]
    volume = factorResult.volume[0]
    lastPrice = factorResult.lastPrice[0] 
    updateDataViewItems(engine=self.dataview, keys=factorSecurityid, 
    valueNames=["ROC","volume"], newValues=(ROC,volume))
    ......
}
```

- 更新新订单数量和金额：在创建新订单时，更新订单数量和金额。

```
if (ROC>strategyParams[factorSecurityid][`ROCThreshold] && volume
>strategyParams[factorSecurityid][`volumeThreshold]) {
    ...
    newOrderNum = (exec newOrderNum from self.dataview 
where securityid=factorSecurityid)[0] + 1
    newOrderAmount = (exec newOrderAmount from self.dataview 
where securityid=factorSecurityid)[0] + lastPrice*0.98*10
    updateDataViewItems(engine=self.dataview, keys=factorSecurityid, 
valueNames= ["newOrderNum", "newOrderAmount"], 
newValues=(newOrderNum, newOrderAmount)) // 更新data view
    ...
}
```

在策略运行时，某一时刻的 DataView 情况如下图所示。

## 5. 总结

在本篇文章中，我们首先详细介绍了 DolphinDB 的复杂事件处理引擎和一些关键概念，如复杂事件和事件监听器等。随后，文章介绍了两个初级的 CEP 引擎使用案例。通过在案例 I 中实现监听市场行情并记录到日志中的功能，我们介绍了创建并运行一个最简单结构的 CEP 引擎所需的步骤和模块。为了更进一步地靠近真实的金融场景，在案例 II 中我们实现了一个事件化量价因子驱动策略。在这个初级的量价策略中，我们联动了 DolphinDB 的响应式状态引擎进行高性能的实时计算，此外介绍了在 CEP 引擎中通过对 addEventListener 方法参数的调整来设置定时任务，以及通过 emitEvent 接口向引擎外部发送事件。
在 CEP 引擎系列教程的下一篇文章中，我们将介绍一个更加复杂、也更加贴近生产情况的量化策略。在这个策略中，不仅涉及到通过数据回放、 模拟撮合引擎对真实交易情况进行仿真，还包括 MACD 和 CCI 等复杂指标的计算和在策略参数寻优时进行并行计算等高阶用法，最终将通过 Dashboard 可视化策略运行的结果。

## 6. 代码附录

案例 I ：监听股价并计入日志
案例 II ：初级高频量价因子策略实现
