# DolphinDB 参考：运维故障排查



---

> 来源: `oom_settlement.html`


# Out of Memory

Out of Memory，简称 OOM，代表内存耗尽的一种异常状态。OOM 的表现形式千差万别，可能是服务异常终止，亦或是系统性能急剧下降。这一现象背后的根本问题在于内存的不足。造成 OOM 的原因有很多，其中包括数据量庞大、频繁的数据写入和查询操作，以及可能存在的内存泄漏问题。了解这些原因，能够帮助我们更好地规划、优化和维护系统，从而提高其稳定性和性能。
本文将针对在使用 DolphinDB 时遇到的 OOM 这一问题以及造成 OOM 的原因进行定位分析和归纳总结，并给出相应解决方案以及规避建议。

## 1. 常见 DolphinDB OOM 报错

DolphinDB 的内存分配机制基于 TCMalloc 实现，TCMalloc 是一个由 Google 开发的用户空间内存分配器，用于替代标准的 C 库中的 malloc 和 free 函数。TCMalloc 的设计目标是提高多线程环境下的内存分配性能，并降低内存碎片。
TCMalloc 在 DolphinDB 中主要负责管理动态内存，而静态内存的释放则由用户手动管理。这种方式可能导致多样化的内存需求，从而增加了内存消耗和内存压力。在某些情况下，这可能最终导致内存不足（OOM）的问题。
DolphinDB 内存不足的常见错误，错误示例包括：
- Java heap space
- OOM Killer
- ChunkCacheEngine is out of memory, possibly due to a transaction that tries to write data larger than the available memory
- C++:std::bad_alloc
- Failed to merge results of partitioned calls: Failed to append data to column 'ETFSellNumber' with error: Out of memory' script: 'select * from pt where date(DateTime) = 2021.12.29;
- Out of memory, BlockFileOutputStream failed to alloc memory and will retry.
- [xx plugin],Out of memory

## 2. 常见 DolphinDB OOM 原因

导致 DolphinDB OOM 的主要原因可分为以下三类：
- DolphinDB 内部组件 ：数据量、查询复杂度、并发用户数等因素影响着 DolphinDB 的工作负荷。因此，当用户使用 DolphinDB 处理大规模数据或者执行复杂查询时，内存需求会相应增加，当超出可分配内存后会造成 OOM。
- 外部组件 ：外部组件如应用程序（插件）内存管理策略以及 GUI、vscode 和 API 的使用不当也会导致 DolphinDB OOM 的发生。具体来说，内存未及时释放、内存泄漏、内存分配不合理等问题都可能导致内存资源的浪费和不足，进而导致后续执行时发生 OOM。
- 外部限制 ：操作系统的配置以及物理内存的可用性是影响 OOM 的关键因素之一。如果操作系统未正确配置，或者物理内存不足，DolphinDB 在执行大规模任务时可能面临内存不足的问题。

## 3. 故障常规排查方法

如果内存不足错误只是偶尔出现，或者在短时间内出现，这代表着系统正常运行中的短时异常，无需特别处理这种短暂的内存问题。然而，如果错误在多个连接上多次发生，并且持续数秒或更长时间，就需要采取一些诊断和解决方案来解决内存问题。

### 3.1. 诊断内存使用模式

使用监控工具来诊断 DolphinDB 内存使用模式。查看内存使用的波动情况，分析内存占用的高峰时刻。这有助于确定是什么导致了内存不足的问题。

### 3.2. 检查 DolphinDB 中组件运行任务

来自 DolphinDB 内部组件的内存压力也可能导致内存不足错误。比如数据查询计算功能、流计算引擎、写入任务、后台计算任务组件消耗内存，需要通过内存管理函数跟踪，了解这些组件在 DolphinDB 中如何分配内存。想要解决内存不足问题，必须确定哪些组件消耗的内存最大。例如，如果发现某个 session 的内存管理显示较大的内存分配，则需要了解当前 session 消耗这么多内存的原因。用户可能会发现某些查询没有走分区扫描，可以通过优化 SQL 语句来优化查询或者建立合适的 sortKey，减少内存的消耗。所以需要检查 DolphinDB 中正在执行的查询和任务，以确定是否有复杂或者资源密集型的操作。

### 3.3. 检查通过外部加载到 DolphinDB 中的插件内存使用情况

内存压力可能由 DolphinDB 内部引擎导致，也可能由 DolphinDB 进程中运行的某些插件导致，如 kafka、mysql、odbc 以及 hdfs 插件或者自定义插件等等。插件若设计不佳，容易消耗大量内存。例如，假设任意插件将来自外部源的 20 亿行数据缓存到 DolphinDB 内存中，DolphinDB 进程中消耗的内存会很高，从而导致内存不足错误。
插件的内存使用目前仅仅记录在日志中，你可以分析 DolphinDB 的日志，来查看是否有异常情况或者报错信息。

### 3.4. 更新或优化配置

有三大类外部限制导致 DolphinDB 内存不足：
- DolphinDB 许可证限制
- DolphinDB 配置文件限制
- OS 内存压力
OS 内存压力是指在同一操作系统中，一个或多个应用程序共同耗尽了可用的物理内存。为了响应新的应用程序对资源的请求，OS 会尝试释放一些内存。然而，当其他应用程序消耗完 OS 的内存，DolphinDB 可用内存将不足，由此产生 OOM 的报错。
针对此种情况，可以考虑更新 DolphinDB 的版本，或者调整配置参数以优化系统性能。

### 3.5. 内部组件排查方案

若要诊断来自 DolphinDB 引擎内部组件的内存压力，请使用以下方法：
- 使用函数查看内存使用情况： getSessionMemoryStat() ，获取当前节点所有连接会话的内存占用状态，返回一张表包含以下字段： userId：用户 ID 或缓存类型的标识符（形如：__xxx__）。 sessionId：会话 ID。 memSize：会话所占用的内存，单位为字节。 remoteIP：发起会话的客户端的 IP。 remotePort：发起会话的客户端的端口号。 createTime：会话创建的时间，为 TIMESTAMP 类型。 lastActiveTime：会话最近一次执行脚本的时间戳。
缓存类型对应解释见下表：
注意 ：2.00.11 和 1.30.23 版本启用了维度表自动回收机制。通过参数 warningMemSize 设置的定期内存检测机制，当内存使用超过该阈值时，会尝试释放部分维度表内存。
| 缓存类型 | 含义 | 影响大小因素 | 释放函数 |
| --- | --- | --- | --- |
| DimensionalTable | 维度表缓存，单位为 Bytes。 | 维度表大小以及数据量 | 目前暂时没有接口 |
| SharedTable | 共享表缓存，单位为 Bytes。 | 共享表大小以及数据量 | undef("sharedTable", SHARED) |
| OLAPTablet | OLAP 引擎数据库表的缓存，单位为 Bytes。 | OLAP 数据缓存大小，一般都是 MB 级别 | clearAllCache() |
| OLAPCacheEngine | OLAP 引擎 cache engine 的内存占用，单位为 Bytes。 | OLAP CacheEngine 参数有关 | flushOLAPCache() |
| OLAPCachedSymbolBase | OLAP 引擎 SYMBOL 类型字典编码的缓存，单位为 Bytes。 | OLAP，Symbol 数据类型大小，一般为 MB 级别 | 无需释放 |
| DFSMetadata | 分布式存储的元数据内存占用情况，单位为 Bytes。 | 分布式库数量以及大小，一般为 MB 级别 | 无需释放 |
| TSDBCacheEngine | TSDB 引擎 cache engine 的内存占用，单位为 Bytes。 | TSDB CacheEngine 参数有关 | flushTSDBCache() |
| TSDBLevelFileIndex | TSDB 引擎 level file 索引的缓存，单位为 Bytes。 | TSDB 常驻索引大小TSDBLevelFileIndexCacheSize有关，默认为 5% *maxMemSize | invalidateLevelIndexCache() |
| TSDBCachedSymbolBase | TSDB 引擎 SYMBOL 类型字典编码的缓存，单位为 Bytes。 | TSDB，Symbol 数据类型大小，一般为 MB 级别 | 无需释放 |
| StreamingPubQueue | 流数据发布队列里未处理的消息数。 | maxPubQueueDepthPerSite参数有关 | 无需释放 |
| StreamingSubQueue | 流数据订阅队列里未处理的消息数。 | 流数据队列内存大小，和引擎数量以及订阅数据有关 | 管理订阅以及引擎 |
定位数据内存占用过大的具体对象，然后使用上表对应函数中的第四列函数进行释放，另外假设为 session 内变量占用内存过高，可以联系团队 DBA 使用函数 closeSessions 关闭相应 session，操作步骤如下：

```
closeSessions(getSessionMemoryStat().sessionId[11]);
```

- 通过函数 getRecentJobs() 和 getConsoleJobs() 查看是否还有超过预期运行时长的后台或交互任务。
通过 cancelJob 以及 cancelConsoleJob 关闭对应 job。

### 3.6. 外部组件排查方案

若要诊断由 DolphinDB 插件引起的内存压力，请使用以下方法：
- 查看日志，如果有日志中存在 [xx plugin],Out of memory ，可以判断是 DolphinDB 插件引发的 OOM。如果为官方发布的插件，可以通过点击官网下载页面的社群入口，将问题反馈给工程师。如果是自定义开发插件，需要定位下哪个函数导致的内存使用过高，修改插件源代码，重新编译再加载到 DolphinDB 中。
- Java heap space 该异常通常出现在 DolphinDB GUI 客户端，通常情况下，直接原因为 GUI 客户端占用内存过高，而这是由于运行以下语句造成：

```
select * from xx
```

需要修改 SQL 语句为

```
t = select * from xx
select top 1000 * from xx
```

这样可以降低 GUI 客户端内存占用，从而避免上述问题。

### 3.7. 外部配置排查方案

面对 DolphinDB 进程之外的外部限制以及操作系统上的内存不足情况，请使用以下方法：
- Linux 内核提供了一种名为 OOM killer（Out Of Memory killer）的机制，用于监控占用内存过大的进程，尤其是瞬间占用大量内存的进程，以防止内存耗尽而自动把该进程杀掉。查看操作系统日志排查是否触发了 OOM Killer，操作步骤如下： 输入命令 dmesg -T|grep memory 如上图，若出现了“Out of memory: Kill process”，说明 DolphinDB 使用的内存超过了操作系统所剩余的空闲内存，导致操作系统杀死了 DolphinDB 进程。解决这种问题的办法是：通过参数 maxMemSize （单节点模式修改 dolphindb.cfg ，集群模式修改 cluster.cfg ）设定节点的最大内存使用量。需要合理设置该参数，设置太小会严重限制集群的性能；设置太大可能触发操作系统杀掉进程。若机器内存为 16 GB，并且只部署 1 个节点，建议将该参数设置为 12 GB（服务器内存的 80% - 90% 之间）左右。
- 收集 DolphinDB 许可证 lic 内存限制信息，操作步骤如下 license().maxMemoryPerNode //Output 8 如上所示，结果为 8，则当前 lic 限制最大使用内存为 8 GB（如果 lic 类型为社区版），DolphinDB 使用内存最大上限为 8 GB，如果需要申请使用更多内存，请联系 DolphinDB 技术支持工程师。
- 收集 DolphinDB 配置文件内存限制信息，操作步骤如下： getConfig(`maxMemSize) //Output 16 如上所示为系统配置内存上限，结果为 16，则当前配置文件限制最大使用内存为 16 GB，DolphinDB 使用内存最大上限为 16 GB，如果需要申请使用更多内存，通过参数 maxMemSize （单节点模式修改 dolphindb.cfg ，集群模式修改 cluster.cfg ）设定节点的最大内存使用量。 注意 ：配置文件最大内存不可超过 DolphinDB 许可证限制的内存以及不能超过服务器内存的 90%。 getMemLimitOfQueryResult() //Output 8 如上所示为查询内存上限配置，结果为 8，则当前配置文件限制最大单次查询结果占用的内存上限为 8 GB，如果需要增大单次查询的上限，通过函数 setMemLimitOfQueryResult 进行修改。
- 收集 Linux 操作系统对 DolphinDB 限制内存信息，操作步骤如下： ulimit -a 参数 描述 max memory size 一个任务的常驻物理内存的最大值 virtual memory 限制进程的最大地址空间 如上图所示，操作系统会限制 DolphinDB 进程的资源上限，图上代表为不限制进程的最大使用内存，如果需要修改，修改方式见 ulimit 修改方式 。
- 查看应用程序事件日志中与应用程序相关的内存问题。解决不太重要的应用程序或服务的任何代码或配置问题，以减少其内存使用量，操作步骤如下： top 如上图所示，有其他进程占用操作系统资源，需要将其关闭，以减少其内存使用量。

## 4. 规避建议

在企业生产环境下，DolphinDB 往往作为流数据中心以及历史数据仓库，为业务人员提供数据查询和计算。当用户较多时，不当的使用容易频繁造成 OOM 影响业务使用，甚至宕机。为尽量减少此类问题的发生，现给出以下五类规避 DolphinDB OOM 的建议。

### 4.1. 优化服务器以及数据库配置

- 打开操作系统对进程的资源限制 ：Linux 对于每个进程有限制最大内存机制。为提高性能，可以根据设备资源情况，对 DolphinDB 内存限制合理规划。
- 合理配置 maxMemSize ： 如果 DolphinDB 许可证限制内存大于等于服务器 80% - 90% 的内存，则 maxMemSize 为服务器内存 80% - 90% 之间 如果 DolphinDB 许可证限制内存小于服务器 80% - 90% 的内存，则 maxMemSize 为 DolphinDB 许可证限制内存大小
- 对自定义插件内存严格管理 自定义插件为 C++ 语言编写，需要插件开发者正确管理函数使用内存等方式确保防止内存泄漏问题。

### 4.2. 合理分区和正确使用 SQL

- 合理均匀分区 ：DolphinDB 以分区为单位加载数据，因此，分区大小对内存影响巨大。合理均匀的分区，不管对内存使用还是对性能而言，都有积极的作用。因此，在创建数据库的时候，根据数据规模，合理规划分区大小。每个分区压缩前的数据量在 100 MB 到 1 GB 之间为宜。具体分区设计，请参考 分区注意事项 。
- 数据查询尽可能使用分区过滤条件 ：DolphinDB 按照分区进行数据检索，如果不加分区过滤条件，则会扫描所有数据，数据量大时，内存很快被耗尽。若存在多个过滤条件，将包含分区列的过滤条件前置。
- 只查询需要的列 ：谨慎使用 select *，select * 会把该分区所有列加载到内存，而实际查询往往只需要几列的数据。因此，为避免内存浪费，尽量明确写出所有查询的列。

### 4.3. 合理配置流数据缓存区

- 合理配置流数据的缓存区 ：一般情况下流数据的容量（capacity）会直接影响发布节点的内存占用。比如，在对流数据表进行持久化时，若 capacity 设置为 1000 万条，那么流数据表在超过 1000 万条时，会将约一半的数据进行存盘并回收，也就是内存中会保留 500 万条左右。因此，应根据发布节点的最大内存，合理设计流表的 capacity。尤其是在多张发布表的情况，更需要谨慎设计。

### 4.4. 及时管理 session 的变量

- 及时释放数据量较大的变量 ：若用户创建数据量较大的变量，例如 v = 1..10000000 ，或者将含有大量数据的查询结果赋值给一个变量 t = select * from t where date = 2010.01.01 ， v 和 t 将会在用户的 session 占用大量的内存。如果不及时释放，执行其他任务时，就有可能因为内存不足而抛出异常。用户的私有变量在其创建的 session 里面保存。session 关闭的时候，会回收这些内存。可通过 undef 函数将其赋值为 NULL，或者关闭 session 来及时释放变量的内存。

### 4.5. 分批次写入数据

- 避免大事务写入 ：通过 append！ 或 tableInsert 批量写入较大数据量时，会长时间占用内存，导致内存无法释放。 可以使用 loadTextEx 接口代替原先 loadText + append！ 的方式写入数据 API 端提供 tableAppender 接口自动切分大数据分批次写入
注意 ：此类接口由于将大数据切分为多块小数据写入，没办法保证一批大数据的写入的原子性。

## 5. 总结

分布式数据库 DolphinDB 的设计十分复杂，发生 OOM 的情况各有不同。若发生节点 OOM，请按本文所述一步步排查：
- 首先，排查宕机原因，是否因为内存耗尽使得 DolphinDB 进程被操作系统杀掉。排查 OOM killer 可查看操作系统日志；
- 其次，检查 OOM 是否由外部限制所导致。如果是因为操作系统限制，可以检查操作系统 ulimit 信息限制 DolphinDB 进程最大内存。如果是 DolphinDB 自身限制，可检查 DolphinDB 许可证文件以及 DolphinDB 配置文件对内存的限制；
- 再次，检查一下是否做到了合理分区，正确使用 SQL，合理配置流数据缓存区，及时管理 session 变量以及合理写入任务；
- 最后，若确定 DolphinDB 系统或者插件有问题，请保存节点日志、操作系统日志以及尽可能复现的脚本，并及时与 DolphinDB 工程师联系。


---

> 来源: `memory_management.html`


# 内存管理

DolphinDB 是一款支持多用户多任务并发操作的高性能分布式时序数据库软件（distributed time-series database）。针对大数据的高效的内存管理是其性能优异的原因之一。

## 1. 内存管理机制及相关配置参数

DolphinDB 使用 TCMalloc 进行内存分配。当用户查询操作或编程环境需要内存时，DolphinDB 会以 512MB 为单位向操作系统申请内存。如果操作系统无法提供大块的连续内存，则会尝试 256MB，128MB 等更小的内存块。系统会每隔 30 秒扫描一次，如果内存块完全空闲，则会整体还给操作系统，如果仍有小部分内存在使用，比如 512MB 的内存块中仍有 10MB 在使用，则不会归还操作系统。
DolphinDB 开放了一些内存管理相关的配置项，方便用户根据系统情况进行合理设置。
- 通过参数 maxMemSize 设定节点的最大内存使用量：该参数指定节点的最大可使用内存。如果设置太小，会严重限制集群的性能；如果设置值超过物理内存，则内存使用量过大时可能会触发操作系统强制关闭进程。同一个服务器上各数据节点的最大内存使用量之和，建议设置为机器可用内存的 75%。例如机器内存为 16GB，并且只部署 1 个节点，建议将该参数设置为 12GB 左右。maxMemSize 包含三个部分：小对象内存区（ reservedMemSize ）、紧急内存区（ emergencyMemSize ），以及剩余部分构成的常规内存区。
- 参数 reservedMemSize 和 maxBlockSizeForReservedMemory：当常规内存区用尽时，系统会进入"小对象分配"状态，此时通过 maxBlockSizeForReservedMemory 控制每次分配的最大内存块大小，避免一次性申请过多内存导致溢出。若新申请的内存块大于该值将受限或失败，从而减少系统关键操作失败的概率。当小对象内存区也用满后，系统会停止为常规任务分配内存。
- 参数 emergencyMemSize：定义紧急内存区大小，当常规内存和小对象内存区都用尽时，紧急内存区会为关键任务保留资源，如存储引擎的刷盘操作或分布式写入事务提交时的关键步骤等。当 maxMemSize 完全用完后，DolphinDB 将停止为所有任务分配内存，仅允许管理员取消任务（详见 系统卡死 ）。
- 参数 warningMemSize 设定最多可缓存的数据量：当节点的内存使用总量小于 warningMemSize（以 GB 为单位，默认值为 maxMemSize 的 75%）时，DolphinDB 会尽可能多的缓存数据库分区数据，以便提升用户下次访问该数据块的速度。当内存使用量超过 warningMemSize 时，系统采用 LRU 的内存回收策略，自动清理部分数据库的缓存，以避免出现 OOM 异常。
- 参数 memoryReleaseRate 控制将未使用的内存释放给操作系统的速率：memoryReleaseRate 是 0 到 10 之间的浮点数。memoryReleaseRate=0 表示不会主动释放未使用的内存。设置值越高，DolphinDB 释放内存的速度越快。默认值是 5。
- 参数 maxPartitionNumPerQuery 控制单次查询数据量：系统默认允许单次最多可查找 65536 个分区的数据。若一次查询过多分区，需加载到内存的数据量过大，则可能导致 OOM。可根据需求以及可用内存量，适当调节该参数，控制单次可查询的分区数量。
- 参数 maxTransactionRatio 控制 TSDB 或 OLAP 存储引擎的单次写入事务大小：该参数表示单次写入事务大小上限与存储引擎 cache engine 大小的比值。例如： TSDB Cache Engine=16G，此参数设置为0.2，则表示写入事务不能超过 16 * 0.2 = 3.2G。 若多节点 Cache Engine 大小不同（如 Node1=4G，Node2=2G），则按平均值（3G）计算。 关于参数配置的详情，参见用户手册，内存配置参数（单实例配置）

## 2. 高效使用内存

在企业的生产环境中，DolphinDB 往往作为流数据处理中心以及历史数据仓库，为业务人员提供数据查询和计算。当用户较多时，不当的使用容易造成 Server 端内存耗尽，抛出 "Out of Memory" 异常。可遵循以下建议，尽量避免内存的不合理使用。
- 合理均匀分区：DolphinDB 以分区为单位加载数据，因此，分区大小对内存影响巨大。合理均匀的分区，不管对内存使用还是对性能而言，都有积极的作用。因此，在创建数据库的时候，根据数据规模，合理规划分区大小。每个分区压缩前的数据量在 100M B 到 1GB 之间为宜。具体分区设计，请参考 分区注意事项 。
- 及时释放数据量较大的变量：若用户创建数据量较大的变量，例如 v = 1..10000000，或者将含有大量数据的查询结果赋值给一个变量 t = select * from t where date = 2010.01.01，v 和 t 将会在用户的 session 占用大量的内存。如果不及时释放，执行其他任务时，就有可能因为内存不足而抛出异常。用户的私有变量在其创建的 session 里面保存。session 关闭的时候，会回收这些内存。可通过 undef 函数将其赋值为 NULL，或者关闭 session 来及时释放变量的内存。
- 只查询需要的列：避免使用 select *，select * 会把该分区所有列加载到内存，而实际查询往往只需要几列的数据。因此，为避免内存浪费，尽量明确写出所有查询的列。
- 数据查询尽可能使用分区过滤条件：DolphinDB 按照分区进行数据检索，如果不加分区过滤条件，则会全部扫描所有数据，数据量大时，内存很快被耗尽。若存在多个过滤条件，将包含分区列的过滤条件前置。
- 合理配置流数据的缓存区：一般情况下流数据的容量 (capacity) 会直接影响发布节点的内存占用。比如，在对流数据表进行持久化时，若 capacity 设置为 1000 万条，那么流数据表在超过 1000 万条时，会将约一半的数据进行存盘并回收，也就是内存中会保留 500 万条左右。因此，应根据发布节点的最大内存，合理设计流数据表的 capacity。尤其是在多张发布表的情况，更需要谨慎设计。

## 3. 内存监控及常见问题


### 3.1. 内存监控


#### 3.1.1. controller 上监控集群中节点内存占用

在 controller 上提供函数 getClusterPerf() 函数，显示集群中各个节点的内存占用情况。包括： memoryAlloc ：节点上分配的总内存，近似于向操作系统申请的内存总和。 memoryUsed ：节点已经使用的内存。该内存包括变量、分布式表缓存以及各种缓存队列等。 maxMemSize ：节点可使用的最大内存限制。

#### 3.1.2. mem() 函数监控某个节点内存占用

mem() 函数可以显示整个节点的内存分配和占用情况。 allocatedBytes 为已分配内存； freeBytes 是可用内存。两者之差为已占用内存。通过 mem().allocatedBytes - mem().freeBytes 得到节点所使用的总的内存大小。（注：如使用 1.20.0 及更早版本，通过 sum(mem().allocatedBytes - mem().freeBytes) 进行计算。）

#### 3.1.3. objs() 函数监控某个会话内各变量的内存占用

通过函数 objs 来查看会话内所有变量的内存占用。该函数返回一个表，其中列 bytes 表示变量占用的内存块大小。 objs(true) 除返回当前会话中变量的内存占用外，还返回共享变量的内存占用（包括其他会话共享的变量）。

#### 3.1.4. 查看某个对象占用的内存大小

通过函数 memSize 来查看某个对象占用内存的具体大小，单位为字节。比如

```
v=1..1000000
memSize(v)
```

输出：4000000。

#### 3.1.5. 监控节点上不同 session 的内存占用

可通过函数 getSessionMemoryStat() 查看节点上每个 session 占用的内存量，输出结果只包含 session 内定义的变量。由于共享表和分布式表不属于某个用户或会话，因此返回结果中不包含它们占用的内存信息。由于输出结果亦包含用户名，也可查看每个用户的内存使用情况。

#### 3.1.6. 查看后台正在运行的任务

以上方法无法获取后台运行任务所占用的内存。可以通过函数 getRecentJobs 来查看正在运行的后台任务。目前，DolphinDB 无法通过函数查看后台任务的内存占用情况，需用户根据业务逻辑进行估算。若内存比较紧张，可以取消一些暂无必要执行的后台任务，以释放内存空间。

### 3.2. 常见问题


#### 3.2.1. 监控显示节点内存占用太高

若节点内存接近 warningMemSize，通过函数 clearAllCache() 来手动释放节点的缓存数据。如果内存占用仍然未显著降低，请依次排查下述问题：
- 通过函数 getSessionMemoryStat() 结合 objs() 和 objs(true) 排查是否某个用户忘记 undef 了变量 ;
- 通过函数 getRecentJobs() 和 getConsoleJobs() 查看是否还有超过预期运行时长的后台或交互任务。运行中的任务的内存占用不反映在 getSessionMemoryStat() ;
- getStreamingStat() 查看流数据内存占用，详见 流数据消息缓存队列 。

#### 3.2.2. Out of Memory (OOM)

该异常往往是由于 query 所需的内存大于系统可提供的内存导致的。请先执行下列脚本查看所有数据节点内存使用情况，其中 maxMemSize 为配置的节点最大可用内存，memoryUsed 为节点已使用内存，memoryAlloc 为节点已分配内存。

```
select site, maxMemSize, memoryUsed, memoryAlloc from rpc(getControllerAlias(),getClusterPerf)
```

OOM 一般可能由以下原因导致:
- 查询没有加分区过滤条件或者条件太宽，导致单个 query 涉及的数据量太大。 查询涉及多少分区可用 sqlDS 函数来判断，示例脚本如下： ds=sqlDS(<select * from loadTable("dfs://demo","sensor")>) ds.size() 为避免这个问题，参见本文档的 DolphinDB 按照分区进行数据检索 ，更详细的说明请参考教程 SQL 案例分区剪枝
- 写入缓存占用了大量内存。 getCacheEngineMemSize() //查看 OLAP 引擎的 Cache Engine 占用内存 getTSDBCacheEngineMemSize() //查看 TSDB 引擎的 Cache Engine 占用内存 为避免这个问题，请参考本文档 为分布式数据库提供写入缓存
- 分区不均匀。可能某个分区过大，该分区的数据超过节点配置的最大内存。 查询每个分区的大小可用 getTabletsMeta 函数，例如： getTabletsMeta("/demo/%", `sensor, true); 为避免这个问题，参见 分区设计注意事项 。
- 某个 session 持有大的变量，导致节点可用的内存很小。 可以通过函数 getSessionMemoryStat() 查看各个 session 占用的内存大小。
- 某个 session 持有大的变量，导致节点可用的内存很小。 通过函数 getSessionMemoryStat() 查看各个 session 占用的内存大小及各 session 对应的用户。 通过以下脚本查看每个数据节点上定义的变量和占用的内存： pnodeRun(objs) //查询每个节点定义变量占用内存（非共享变量） pnodeRun(objs{true}) //查询每个节点定义变量占用内存（包含共享变量） 为避免这个问题，请参考本文档 高效使用内存 下的“及时释放数据量较大的变量”
- 流数据计算引擎占用了大量内存。 用 getStreamingStat() 来查看发布、订阅各个队列的深度。用 getStreamEngineStat() 查看流计算引擎占用内存。 为避免这个问题，请参考本文档 流数据消息缓存队列 。

#### 3.2.3. 查询时，DolphinDB 进程退出，没有 coredump 产生

这种情况往往是由于给节点分配的内存超过系统物理内存的限制，操作系统把 DolphinDB 强制关闭。Linux 内核有个机制叫 OOM killer(Out of Memory killer)，该机制会监控那些占用内存过大，尤其是瞬间占用很大内存的进程，为防止内存耗尽而自动把该进程杀掉。排查 OOM killer 可用 dmesg 命令，示例如下：

```
dmesg -T|grep dolphindb
```

若打印结果中出现了“Out of memory: Kill process”，说明操作系统杀死了 DolphinDB 进程（详情请见 Linux Kernel 相关文档 ）。解决这种问题的办法是：通过参数 maxMemSize（单节点模式修改 dolphindb.cfg，集群模式修改 cluster.cfg）设定节点的最大内存使用量。

#### 3.2.4. 执行 mem(true) 或 clearAllCache() 后，操作系统实际内存占用未降低（如 Linux 中 RSS 占用）

这是由于未使用的内存没有及时释放给操作系统。通过 memoryReleaseRate 控制将未使用的内存释放给操作系统的速率，memoryReleaseRate=10 表示以最快的速度释放内存，默认值是 5。该参数等价于设置了 TCMalloc 的 tcmalloc_release_rate。
注：执行 mem() 以显示本地节点内存使用情况。如设置其参数 freeUnusedBlocks=true，系统将会释放未使用的内存块。

#### 3.2.5. 执行 loadText 大批量加载到内存时，时间过久或卡死

可检查 log 中是否打印 Cache Engine 相关的信息。若存在这样的信息，说明使用 loadText 加载到内存的事务超过了 chunkCacheEngineMemSize 或 TSDBCacheEngineSize 的大小，导致事务卡死。
调大配置中的 chunkCacheEngineMemSize 或 TSDBCacheEngineSize 值。
2.00.4 及以上版本，请使用 loadTextEx 函数加载数据。这是因为 loadTextEx 函数可以通过参数 atomic，设置将大文件加载过程拆分为多个事务进行。

#### 3.2.6. getClusterPerf() 返回的节点分配内存（memoryAlloc）与操作系统实际显示值有差异

DolphinDB 是 C++ 程序，本身需要一些基础的数据结构和内存开销，memoryAlloc 显示内存不包括这些内存。在执行 clearAllCache 后，如果 memoryAlloc 的值占操作系统实际显示值的 80% 以上，属于正常现象。

## 4. 变量的内存管理

DolphinDB 为用户提供与回收编程环境所需内存，本节介绍为变量分配内存，及释放变量内存的方法。

### 4.1. 创建变量

示例 1. 创建一个 vector，含有 1 亿个 INT 类型元素，约 400MB。

```
v = 1..100000000
mem().allocatedBytes - mem().freeBytes //输出内存占用结果
```

结果为：402,865,056，内存占用 400MB 左右，符合预期。
示例 2. 创建一个 table，1000 万行，5 列，每列 4 字节，约 200MB。

```
n = 10000000
t = table(n:n,["tag1","tag2","tag3","tag4","tag5"],[INT,INT,INT,INT,INT])
mem().allocatedBytes - mem().freeBytes
```

结果为：612,530,448，约 600MB，符合预期。

### 4.2. 释放变量

可通过 undef 函数或者赋值为 NULL，释放变量的内存。
示例 3. 使用 undef 函数

```
undef(`v)
```


```
v = NULL
```

释放共享变量占用内存示例如下：

```
undef("sharedTable", SHARED)
```

除了手动释放变量，当 session 关闭时，比如关闭 GUI 和其他 API 连接，都会触发对该 session 的所有内存进行回收。当通过 web notebook 连接时，10 分钟内无操作，系统会关闭 session，自动回收内存。

## 5. 分布式数据库读缓存管理

DolphinDB 在不同存储引擎的读缓存管理上存在区别。本节分别介绍 OLAP 引擎和 TSDB 引擎读缓存的分配和管理。OLAP 引擎自动将查询过的历史数据加载到读缓存，后续相关数据的查询将先从缓存中读取。多个会话共享分区表读缓存数据，以提高内存使用率。TSDB 引擎通过索引可快速定位分布式表的数据位置，所以 TSDB 引擎的读缓存只缓存查询数据相关的索引文件，不缓存查询的数据。

### 5.1. OLAP 引擎分布式表的缓存管理

DolphinDB 对分布式表是以分区中的列为单位管理的。系统自动将查询的分区数据加载到读缓存，后续相关数据的查询将首先从缓存中读取。DolphinDB 记录所有数据的版本号，通过版本机制系统可以快速确定是否需要为一个查询更新缓存。读缓存无需用户指定，系统自动进行管理，当内存不足时，系统自动释放一部分缓存。
历史数据一般以分布式表的形式存在数据库中，用户平时查询操作也往往直接查询分布式表。分布式表的内存管理有如下特点：
- 内存以分区的一列为单位进行管理。
- 数据只加载到所在的节点，不会在节点间转移。
- 多个用户访问相同分区时，使用同一份缓存。
- 内存使用不超过 warningMemSize 情况下，尽量多缓存数据。
- 缓存数据达到 warningMemSize 时，系统开始自动回收。
以下多个示例是基于以下集群：部署于 2 个数据节点，采用单副本模式。按天分 30 个区，每个分区 1000 万行，11 列（1 列 DATE 类型，1 列 INT 类型，9 列 LONG 类型），所以每个分区的每列 (LONG 类型）数据量为 1000 万行 * 8 字节/列 = 80M，每个分区共 1000 万行 * 80 字节/行 = 800M，整个表共 3 亿行，大小为 24GB。
函数 clearAllCache() 可清空已经缓存的数据。下面的每次测试前，先用 pnodeRun(clearAllCache) 清空节点上的所有缓存。

```
login(`admin,`123456)
if(existsDatabase("dfs://mem")){
	dropDatabase("dfs://mem")
}
db = database("dfs://mem",VALUE,2022.01.01..2022.01.30)
m = "tag" + string(1..9)
schema = table(1:0,`id`day join m, [INT,DATE] join take(LONG,9) )
db.createPartitionedTable(schema,"pt1",`day)
//写入模拟数据
for (i in 0..29){
	t=table(1..10000000 as tagid,take(2022.01.01+i,10000000) as date,1..10000000 as tag1,1..10000000 as tag2,1..10000000 as tag3,1..10000000 as tag4,1..10000000 as tag5,1..10000000 as tag6,1..10000000 as tag7,1..10000000 as tag8,1..10000000 as tag9 )
	dfsTable=loadTable("dfs://mem","pt1")
	dfsTable.append!(t)
}

//查询15个分区的数据
t=select top 15 * from rpc(getControllerAlias(),getClusterChunksStatus) where file like "/mem%" and replicas like "node1%" order by file
pnodeRun(clearAllCache)
days=datetimeParse(t.file.substr(5,8),"yyyyMMdd")
for(d in days){
    select * from loadTable("dfs://mem","pt1") where  day= d
    print mem().allocatedBytes - mem().freeBytes
}
```


#### 5.1.1. 内存以分区列为单位进行管理

DolphinDB 采用列式存储，当用户对分布式表的数据进行查询时，加载数据的原则是，只把用户所要求的分区和列加载到内存中。
示例 4. 计算分区 2022.01.01 最大的 tag1 的值。我们只查询 1 个分区的一列数据，仅把该列数据全部加载到内存，其他的列不加载。
该分区储存在 node1 上（可以在 controller 上通过函数 getClusterChunksStatus() 查看分区分布情况，而且由上面可知，每列约 80MB）。在 node1 上 执行如下代码，并查看内存占用。

```
select max(tag1) from loadTable(dbName,tableName) where day = 2022.01.01
mem().allocatedBytes - mem().freeBytes)
```

输出结果为 84,267,136。
示例 5. 在 node1 上查询 2022.01.01 的前 100 条数据，并观察内存占用。

```
select top 100 * from loadTable(dbName,tableName) where day = 2022.01.01
mem().allocatedBytes - mem().freeBytes
```

输出结果为 839,255,392。虽然我们只取 100 条数据，但是 DolphinDB 加载数据的最小单位是分区列，所以需要加载每个列的全部数据，也就是整个分区的全部数据，约 800MB。
注意： 合理分区以避免 "out of memory"：DolphinDB 以分区为单位管理内存，因此内存的使用量跟分区关系密切。假如分区不均匀，导致某个分区数据量超大，甚至机器的全部内存都不足以容纳整个分区，那么当涉及到该分区的查询计算时，系统会抛出 "out of memory" 的异常。一般原则，数据表的每个分区的数据量在 100MB-1GB 之间为宜。如果表有 10 列，每个字段 8 字节，则每个分区约 100-200 万行。

#### 5.1.2. 数据只加载到所在的节点

在数据量大的情况下，节点间转移数据是非常耗时的操作。DolphinDB 的数据是分布式存储的，当执行任务时，把任务发送到数据所在的节点，然后把结果汇总到发送任务的节点。系统会尽最大努力避免将数据跨节点转移。
示例 6. 在 node1 上计算两个分区中 tag1 的最大值。其中分区 2022.01.02 的数据存储在 node1 上，分区 2022.01.03 的数据存储在 node2 上。

```
select max(tag1) from loadTable(dbName,tableName) where day in [2022.01.02,2022.01.03]
mem().allocatedBytes - mem().freeBytes
```

输出结果为 84,284,096。在 node2 上查看内存占用为 84,250,624 字节。每个节点存储的数据都为 80MB 左右，也就是 node1 上存储了分区 2022.01.02 的数据，node2 上仅存储了 2022.01.03 的数据。
示例 7. 在 node1 上查询分区 2022.01.02 和 2022.01.03 的所有数据，我们预期 node1 加载 2022.01.02 数据，node2 加载 2022.01.03 的数据，都是 800MB 左右，执行如下代码并观察内存。

```
select top 100 * from loadTable(dbName,tableName) where day in [2022.01.02,2022.01.03]
mem().allocatedBytes - mem().freeBytes
```

node1 上输出结果为 839,279,968 字节。node2 上输出结果为 839,246,496 字节。结果符合预期。
注意： 请谨慎使用没有过滤条件的 "select *"，因为这会将所有数据载入内存。特别在列数很多的时候，建议仅加载需要的列。

#### 5.1.3. 多个用户访问相同分区时，使用同一份缓存

DolphinDB 支持海量数据的并发查询。为了高效利用内存，对相同分区的数据，内存中只保留同一份副本。
示例 8. 打开两个 GUI，分别连接 node1 和 node2，查询分区 2022.01.01 的数据，该分区的数据存储在 node1 上。

```
select * from loadTable(dbName,tableName) where date = 2022.01.01
mem().allocatedBytes - mem().freeBytes
```

node1 上内存显示 839,101,024，而 node2 上无内存占用。系统将分区数据载入 node1 内存，然后将数据传输到 node2 内存，最后下载到客户端。之后，系统会将此次操作在 node2 所用内存释放。

#### 5.1.4. 节点内存使用不超过 warningMemSize 情况下，尽量多缓存数据

通常情况下，最近访问的数据往往更容易再次被访问，因此 DolphinDB 在内存允许的情况下（内存占用不超过用户设置的 warningMemSize），尽量多缓存数据，来提升后续查询效率。
示例 9. 数据节点设置的 maxMemSize=10,warningMemSize=8。连续加载 9 个分区，每个分区约 800M，总内存占用约 7.2GB，观察内存的变化趋势。

```
dbPath =  right(dbName,strlen(dbName)-5)
p = select top 9 * from rpc(getControllerAlias(),getClusterChunksStatus) 
    where file like dbPath +"%" and replicas like "node1%" //这里节点1的别名为node1
    order by file
days = datetimeParse(t.file.substr(strlen(dbPath)+1,8),"yyyyMMdd")
for(d in days){
    select * from loadTable(dbName,tableName) where  date= d
    print mem().allocatedBytes - mem().freeBytes
}
```

内存随着加载分区数的增加变化规律如下图所示：
当遍历每个分区数据时，在内存使用量不超过 warningMemSize 的情况下，分区数据会全部缓存到内存中，以便用户下次访问时，直接从内存中读取数据，而不需要再次从磁盘加载。

#### 5.1.5. 节点内存使用达到 warningMemSize 时，系统自动回收

当总的内存使用达到 warningMemSize 时，DolphinDB 会采用 LRU 的内存回收策略，回收一部分内存。
示例 10. 上面用例只加载了 9 天的数据，此时我们继续共遍历 15 天数据，查看缓存达到 warningMemSize 时，内存的占用情况。如下图所示：
如上图所示，当缓存的数据超过 warningMemSize 时，系统自动回收内存，总的内存使用量仍然小于用户设置的 warningMemSize。
示例 11. 当缓存数据接近用户设置的 warningMemSize 时，继续申请 Session 变量的内存空间，查看系统内存占用。
此时先查看系统的内存使用：

```
mem().allocatedBytes - mem().freeBytes
```

输出结果为 7,550,138,448。内存占用超过 7GB，而用户设置的最大内存使用量为 10GB，此时我们继续申请 4GB 空间。

```
v = 1..1000000000
mem().allocatedBytes - mem().freeBytes
```

输出结果为 8,196,073,856。约为 8GB，也就是如果用户定义变量时，可使用的内存不足，系统会触发缓存数据的内存回收，以保证有足够的内存提供给用户使用。

### 5.2. TSDB 引擎分布式表的读缓存管理

与 OLAP 引擎存储数据的最小文件为列文件不同，TSDB 引擎存储数据的最小文件是 level file。每个 level file 都记录了元信息，数据块和数据块对应的索引。TSDB 查询分布式表数据时，会将分区下 level file 的索引部分加载到内存，然后将索引定位的数据部分加载到内存。正是因为通过索引就可以快速定位数据并加载，TSDB 只需要缓存索引数据，无需缓存查询的分区数据。TSDB 索引缓存的容量由配置参数 TSDBLevelFileIndexCacheSize 指定，单位为 GB，默认值为 maxMemSize 的 5%，最小值为 0.1GB。若索引缓存空间不够时，会按照 LRU 算法，释放最近最少使用的 5% 的缓存空间。可通过函数 getLevelFileIndexCacheStatus 获取 TSDB 索引数据占用的内存空间大小。
示例 12.TSDB 引擎下已经创建了如下分布式表

```
n=10000
ID=rand(100, n)
dates=2021.08.07..2021.08.11
date=rand(dates, n)
vol=rand(1..10 join int(), n)
t=table(ID, date, vol)
if(existsDatabase("dfs://TSDB_db1")){
dropDatabase("dfs://TSDB_db1")
}
db=database(directory="dfs://TSDB_db1", partitionType=VALUE, partitionScheme=2021.08.07..2021.08.11, engine="TSDB")
pt1=db.createPartitionedTable(table=t, tableName=`pt1, partitionColumns=`date, sortColumns=`ID)
```

重启 server，查看索引缓存

```
getLevelFileIndexCacheStatus().usage
//输出结果为：0
```

加载分区 2021.08.07 下的数据，查询索引缓存和数据缓存

```
select * from loadTable("dfs://TSDB_db1",`pt1) where date=2021.08.07
getLevelFileIndexCacheStatus().usage
//输出结果为：39128
mem().allocatedBytes - mem().freeBytes
//输出结果为：28537352
```

继续加载分区 2021.08.08 下的数据，查询索引缓存和数据缓存。索引缓存增加，数据缓存无变化（因为 TSDB 引擎查询结束会将释放内存中的数据，不会占用缓存）

```
select * from loadTable("dfs://TSDB_db1",`pt1) where date=2021.08.08
getLevelFileIndexCacheStatus().usage  
//输出结果为：78256
mem().allocatedBytes - mem().freeBytes
//输出结果为：28561912
```


## 6. 为分布式数据库提供写入缓存

Cache Engine 是 DolphinDB 中的一种数据写入缓存机制，用于提升海量数据写入性能。DolphinDB 采用先写入 Redo log（预写式日志）和 Cache Engine（写入缓存）的通用做法，等数据积累到一定数量时，批量写入。DolphinDB 2.00.0 及以上版本支持多模存储（OLAP 与 TSDB），不同的存储引擎拥有独立的 Cache Engine。
Cache Engine 需根据系统配置和实际场景合理设置。若设置过小，可能导致 Cache Engine 频繁刷盘，影响系统性能；若设置过大，Cache Engine 可能会缓存大量未刷盘的数据，此时若发生了机器断电或关机，重启后就需要回放大量事务，导致系统启动过慢。

### 6.1. OLAP 引擎的写入缓存

默认情况下，OLAP 存储引擎是不开启 Redo Log 的，即写入事务不会进行缓存，直接进行刷盘。若需要缓存事务进行批量刷盘，则需要通过 chunkCacheEngineMemSize 为 OLAP 指定 Cache Engine 的容量，且指定 dataSync=1 启用 Redo Log。下图示意了开启 Cache Engine 和 Redo Log 后事务的写入流程：事务先写入 Redo Log 和 Cache Engine。达到 cache engine 的刷盘条件后，三个事务的数据将被一次性写入到 DFS 的数据库上。
Cache Engine 空间一般推荐设置为 maxMemSize 的 1/8 到 1/4。chunkCacheEngineMemSize 不是一个刚性的限制，其实际内存占用可能会高于设置的值 ; 根据经验，cacheEngine 不建议设置超过 32G 以上的值，否则会造成 Redo Log 回收慢和内存占用过高的问题; 更多设置请参阅 Cache Engine 与数据库日志 。
写入过程中，可用 getCacheEngineMemSize() 查看 OLAP 引擎的 Cache Engine 占用内存情况。

### 6.2. TSDB 引擎的写入缓存

与 OLAP 不同的是，TSDB 引擎必须开启 Cache Engine（由配置参数 TSDBCacheEngineSize 指定）和 Redo Log。TSDB 引擎的 Cache Engine 内部会对缓存的数据进行排序，当 Cache Engine 写满或经过十分钟后刷入磁盘。刷盘过程中，Cache Engine 为只读状态，不能继续写入数据，此时，如果又有数据写入到 Cache Engine 里，系统会为 TSDB 引擎重新分配一个内存空间作为 TSDB 的 Cache Engine。因此，极端情况下，TSDB 的 Cache Engine 内存占用最多可达到两倍的 TSDBCacheEngineSize。
通过配置参数 TSDBCacheEngineCompression 设置 TSDB 引擎对 Cache Engine 里的数据进行压缩，来缓存更多（约 5 倍）的数据在缓存中，降低查询数据的时延。
写入过程中，可用 getTSDBCacheEngineMemSize() 查看 TSDB 引擎的 Cache Engine 占用内存情况。

## 7. 流数据消息缓存队列

DolphinDB 为流数据发送节点提供持久化队列缓存和发送队列缓存，为订阅节点提供接收数据队列缓存。
当数据进入流数据系统时，首先写入流数据表，然后写入持久化队列和发送队列。假设用户设置为异步持久化，则持久化队列异步写入磁盘，发送队列发送到订阅端。 当订阅端收到数据后，先放入接受队列，然后用户定义的 handler 从接收队列中取数据并处理。如果 handler 处理缓慢，会导致接收队列有数据堆积，占用内存。如下图所示：
流数据内存相关的主要配置选项：
- maxPersistenceQueueDepth: 发布节点流表持久化队列的最大队列深度（消息数上限）。该选项默认设置为 10000000。在磁盘写入成为瓶颈时，队列会堆积数据。
- maxPubQueueDepthPerSite: 发布节点发送队列的最大队列深度（消息数上限）。默认值为 1000 万，当网络出现拥塞时，该发送队列会堆积数据。
- maxSubQueueDepth: 订阅节点上每个订阅线程接收队列的最大队列深度（消息数上限）。订阅的消息，会先放入订阅消息队列。默认设置为 1000 万，当 handler 处理速度较慢，不能及时处理订阅到的消息时，该队列会有数据堆积。
- 流表的 capacity：由函数 enableTablePersistence() 中第四个参数指定，该值表示流表中保存在内存中的最大行数，达到该值时，从内存中删除一半数据。当流数据节点中，流表比较多时，要整体合理设置该值，防止内存不足。
可以通过函数 objs(true) 或 objs() 来查看流表占用内存占用大小，用 getStreamingStat() 来查看各个队列的深度。流计算引擎可能会占用较多内存，可使用 getStreamEngineStat() 以查看流计算引擎的内存使用量。例如， getStreamEngineStat().TimeSeriesEngine.memoryUsed 以查看时间序列引擎的内存使用量。


---

> 来源: `node_startup_exception.html`


# 节点启动异常


## 1. 启动问题定位思路

DolphinDB 节点启动失败时，最明显的现象是启动节点后，对应的 web 界面无法访问，如果是集群在 web 管理界面会看到节点状态为红色。启动问题主要可以分为三类：
- 启动异常关闭
- 启动异常卡住
首先需要先确认是哪类问题，在启动节点后，通过如下命令查看节点进程是否存在：

```
ps -ef | grep dolphindb // 如果修改了可执行文件名，需要修改 dolphindb 为相应可执行文件名
```

若进程不存在则是启动异常关闭；若进程存在，通过如下命令搜索日志确定节点是否启动完成：

```
grep "Job scheduler initialization completed." dolphindb.log
```

若有执行启动后的时间的日志输出则节点已正常启动，可以去 web 界面刷新确认节点状态是否已为绿色；否则是启动异常卡住或启动慢。通过如下命令搜索节点运行日志中的 ERROR：

```
grep "ERROR" dolphindb.log
```

若在执行启动后的时间一直重复刷某段 ERROR 日志，且节点进程一直在，则为启动异常卡在某个阶段；否则为正常启动，只是比较慢还没启动成功，需要继续等待观察启动结果。
具体问题需要分析 节点运行日志 ，节点运行日志存储位置由命令行参数 logFile 决定。另外集群环境下可以通过 logFile 配置项指定节点运行日志存储位置。
注意 ：若使用 startSingle.sh 启动单节点，节点运行日志默认存储在安装目录的 dolphindb.log 。若使用 clusterDemo 文件夹下的 startController.sh 和 startAgent.sh 来启动集群，节点运行日志默认存储在 clusterDemo/log ，文件名为 .log 。

### 1.1 启动异常关闭

首先需要区分节点是启动成功后的运行过程中异常宕机，还是启动过程中异常关闭。需要参照 节点整体启动流程 一节查看是否有节点启动完成的日志 Job scheduler initialization completed. ，有则是节点启动成功后运行过程中异常宕机，需要根据 排查节点宕机的原因 来定位节点宕机问题；否则是启动过程中异常关闭问题。
需要查看节点最新运行日志中启动阶段的 ERROR 日志。注意要查看启动阶段的 ERROR 日志而不是启动失败后关机阶段的 ERROR 日志。DolphinDB 在关机时打印如下 ERROR 日志是预期的：

```
...
<ERROR> : The socket server ended.
...
<ERROR> : AsynchronousSubscriberImp::run Shut down the subscription daemon.
...
```

如果存在关机日志，需要继续往上搜索 ERROR 查看启动阶段的日志，结合前文的启动流程分析和后文的常见启动问题来分析失败原因。另外需要注意是否在启动过程中宕机，若宕机节点进程会直接被杀死，而不会走关机流程，此时需要查看 coredump 里的堆栈信息：

```
cd /path/to/dolphindb
gdb dolphindb /path/to/corefile
bt
```

堆栈信息需要发给 DolphinDB 技术支持来分析定位。

### 1.2 启动异常卡住

需要查看节点最新运行日志中的 ERROR 日志，结合前文的启动流程分析、后文的常见启动问题来确定当前节点正在启动什么模块、执行什么动作失败。一般会重复打印某段 ERROR 日志以尝试启动，可以使用如下命令实时查看节点刷的日志：

```
tail -f dolphindb.log
```

另外可以使用 pstack 命令来查看启动时节点内部各个线程的堆栈信息，以确定线程具体执行的动作：

```
pstack dolphindb_pid > /tmp/pstack.log // 替换 dolphindb_pid 为 DolphinDB 进程号
```

或使用 gdb 命令获取线程的堆栈信息。

```
gdb -p dolphindb_pid -batch -ex 'thread apply all bt' -ex 'quit' > /tmp/gdb_stacks.log 
// 替换 dolphindb_pid 为 DolphinDB 进程号
```

堆栈信息需要发给 DolphinDB 技术支持来分析定位。

### 1.3 启动慢

需要查看节点最新运行日志，结合前文的启动流程分析、后文的常见启动问题来确定当前节点正在启动什么模块、执行什么动作。启动慢时，一般不会有 ERROR 日志。常见的启动慢原因是回滚事务或回放 redo log，详见 节点启动流程简析与常见问题 。
另外可以使用 pstack 命令来查看启动时节点内部各个线程的堆栈信息，以确定线程具体执行的动作：

```
pstack dolphindb_pid > /tmp/pstack.log // 替换 dolphindb_pid 为 dolphindb 进程号
```

或使用 gdb 命令获取线程的堆栈信息。
堆栈信息需要发给 DolphinDB 技术支持来分析定位。
列出 DolphinDB 常见的启动问题和解决方案。若问题现象不属于常见问题，请联系 DolphinDB 技术支持定位处理。

## 2. 常见问题

本章节将详细列举并解释节点启动异常的常见问题及其相应的解决方案，以帮助快速定位并解决问题。

### 2.1 启动异常关闭


#### 2.1.1 license 过期

DolphinDB 会在 license 过期前 15 天在 web 或 gui 提示 license 即将过期，而过期后如果节点还在线则能够继续使用 15 天，到第 15 天时会自动关机。License 过期后启动 DolphinDB 会失败，节点运行日志中会有如下 WARNING 和 ERROR 日志：

```
2023-10-13 09:52:30.007743 <WARNING> :
    The license has expired. Please renew the license and restart the server.
2023-10-13 09:52:30.163238 <ERROR> : 
    The license has expired.
```

需要联系销售获取更新 license。

#### 2.1.2 端口冲突

DolphinDB 启动时会绑定一个端口用来做网络传输，由配置文件的 localSite 配置项指定。若配置的端口被其他程序占用，或上一次关闭节点还没有完全关闭，则会导致节点启动时绑定端口失败而启动失败。查看节点运行日志有如下报错：

```
2023-10-26 09:01:31.349118 <ERROR> :Failed to bind the socket on port 8848 with error code 98
2023-10-26 09:01:31.349273 <ERROR> :Failed to bind the socket on port 8848. Shutting down the server. 
    Please try again in a couple of
 minutes.
```

运行如下命令查看占用指定端口的程序：

```
netstat -nlp | grep 端口号
```

解决方案是停止占用端口的程序后再启动。若为上一次关闭的节点还没有完全关闭，需要等待节点关闭或使用 kill -9 强行停止节点再启动，强行停止节点可能会导致关机前未完成的写入的数据丢失。

#### 2.1.3 redo log 文件损坏

数据节点启动时会回放 redo log，如果上次运行时出现磁盘满、宕机或 bug，可能导致 redo log 文件损坏，可能导致节点启动时回放 redo log 失败抛出异常而启动失败。
redo log 的存储路径按存储引擎不同，可分别由以下参数进行配置，详见 功能配置 ：
| 配置参数 | 存储引擎 | 默认值 |
| --- | --- | --- |
| redoLogDir=/redoLog | OLAP 引擎 | /log/redoLog |
| TSDBRedoLogDir=/TSDBRedo | TSDB 引擎 | /log/TSDBRedo |
| PKEYRedoLogDir=/PKEYRedo | PKEY 引擎 | <ALIAS>/log/PKEYRedo |
例如查看节点运行日志有如下报错：

```
2023-12-11 15:18:58.888865 <INFO> :applyTidRedoLog : 
    2853,c686664b-d020-429a-1746-287d670099e9,
        /hdd/hdd7/hanyang/server/clusterDemo/data/P1-datanode/storage/CHUNKS/multiValueTypeDb1/20231107/Key0/g
z,pt_2,32054400,1046013,0
2023-12-11 15:18:58.895064 <ERROR> :VectorUnmarshall::start Invalid data form 0 type 0
2023-12-11 15:18:58.895233 <ERROR> :The redo log for transaction [2853] comes across error: 
    Failed to unmarshall data.. Invalid message format
2023-12-11 15:18:58.895476 <ERROR> :The ChunkNode failed to initialize with exception 
    [Failed to unmarshall data.. Invalid message format].
2023-12-11 15:18:58.895555 <ERROR> :ChunkNode service comes up with the error message: 
    Failed to unmarshall data.. Invalid message format
```

日志含义为回放 redo log 时发现 tid 为 2853 的 redo log 文件格式错误导致回放失败。此时需要通过如下步骤跳过 redo log 回放：
- mv 移走 redoLogDir 和 TSDBRedoLogDir 文件夹下的 head.log 文件，cp 备份报错对应的 2853.log 文件；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。
如果不是磁盘满导致，需要将 head.log 和报错对应的 2853.log 发给 DolphinDB 技术支持定位问题。

#### 2.1.4 函数视图或定时任务包含不存在的方法

节点启动时会反序列化函数视图和定时任务文件，若反序列化的方法定义中包含不存在于内存的方法，将会导致相关函数视图和定时任务反序列化失败。常见情况如下：
- 使用了未配置自动加载的插件和模块的方法
- 更新插件或模块后相关方法名变更
例如，定义定时任务 myTest 调用 rabbitmq 插件的方法：

```
loadPlugin("plugins/rabbitmq/PluginRabbitMQ.txt")

def myTest() {
	HOST="192.168.0.53"
    PORT=5672
    USERNAME="guest"
    PASSWORD="guest"

    conn = rabbitmq::connection(HOST, PORT, USERNAME, PASSWORD);
}

scheduleJob("myTest", "myTest", myTest, 15:50m, startDate=today(), endDate=today()+3, frequency='D')
```

如果未配置 preloadModules=plugins::rabbitmq ，则节点启动时不会加载 rabbitmq 插件的函数定义到内存，节点启动反序列化定时任务会失败，运行日志会有如下报错：

```
2023-10-13 09:55:30.166268 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: 
    Can't recognize function: rabbitmq::connection
2023-10-13 09:55:30.166338 <ERROR> :Failed to unmarshall the job [myTest]. Can't recognize function: 
    rabbitmq::connection. Invalid message format
```

解决方案是添加报错方法对应的插件或模块到 preloadModules 配置项，即配置 preloadModules=plugins::rabbitmq ，然后再启动节点。
若为更新插件或模块后相关方法名变更，需要回退插件或模块再启动，删除对应的视图或定时任务后，再升级插件或模块。

#### 2.1.5 函数视图或定时任务包含不存在的共享表

注意 ：该问题已在 1.30.23.1/2.00.11.1 或以上版本修复。
节点启动时会反序列化函数视图和定时任务文件，若反序列化的方法定义中包含不存在于内存的共享表，将会导致相关函数视图和定时任务反序列化失败。
问题 1 ： 定时任务反序列化失败
例如，定时任务 myTest 的定义如下：

```
share table(1 2 3 as id, 1 2 3 as val) as t

def myTest() {
	update t set val = val + 1
}

scheduleJob("myTest", "myTest", myTest, minute(now())+5, today(), today(), 'D')
```

其中第 4 行的 update 语句用于更新共享表 t。如果在启动脚本 startup.dos 中未创建共享表 t，则节点启动时反序列化定时任务会失败，运行日志会有如下 WARNING 和 ERROR 日志：

```
2023-10-23 09:38:27.746184 <WARNING> :Failed to recognize shared variable t
2023-10-23 09:38:27.746343 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: 
    Failed to deserialize update statement
2023-10-23 09:38:27.746404 <ERROR> :Failed to deserialize update statement. 
    Invalid message format
```

原因：定时任务的定义中使用了未创建的表。
- 在报错的定时任务 myTest 中检查是否使用了未创建的表，例如前述的共享表 t。
- 在定时任务所在节点的启动脚本 startup.dos 中添加相应的建表语句。
- 启动节点。
问题 2：函数视图反序列化失败
如果将前述问题 1 中的定时任务 myTest 添加到函数视图，例如： addFunctionView(myTest) ，也会导致节点启动时反序列化函数视图失败，运行日志中会出现相同的报错。
原因：函数视图反序列化先于节点启动，因此在控制节点的 startup.dos 中定义共享表 t 也不会有效。
- 对于普通集群： 移除 server/clusterDemo/data/dnode1/sysmgmt 下的 aclEditlog.meta 、 aclCheckPoint.meta 、 aclCheckPoint.tmp 。 重新启动节点。 启动后重新添加所有权限和函数视图定义。
- 对于高可用集群，如果未重启，或存在半数以上控制节点存活： 删除相关函数视图： dropFunctionView("myTest") 生成权限与函数视图的 checkpoint 文件以避免启动时回放 RAFT 日志执行之前的函数视图定义。 rpc(getControllerAlias(), aclCheckPoint, true) // 参数 force = true 表示强制生成 checkpoint。
- 对于高可用集群，如果已重启： 移除所有控制节点的 <HOME_DIR>/<NodeAlias>/raft 目录下的 raftHardstate 、 raftWAL 、 raftSnapshot 、 raftWAL.old 、 raftSnapshot.tmp 。 注意：这会导致集群元数据全部失效。 重新启动节点。
必然存在 aclEditlog.meta ，可能存在 aclCheckPoint.meta ， aclCheckPoint.tmp 。
必然存在 raftHardstate ， raftWAL ，可能存在 raftSnapshot ， raftWAL.old ， raftSnapshot.tmp 。
后续版本会优化添加函数视图的功能以避免需要删除元数据来解决该问题。

#### 2.1.6 函数视图方法名与预加载的模块或插件的方法名冲突

节点启动时会反序列化函数视图文件，若反序列化的函数视图方法名与已通过 preloadModule 加载的方法名冲突，将会导致函数视图反序列化失败。例如，直接添加 ops 模块的 cancelJobEx 方法到函数视图：

```
use ops
addFunctionView(ops::cancelJobEx)
```

若同时配置 preloadModules=ops ，启动时运行日志会有如下报错：

```
2023-10-20 08:46:15.733365 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: 
    Not allowed to overwrite existing functions/procedures [ops::cancelJobEx] by system users.
2023-10-20 08:46:15.733422 <ERROR> :Not allowed to overwrite existing functions/procedures
    [ops::cancelJobEx] by system users.. Invalid message format
```

解决方案是去掉 preloadModules=ops 配置项，然后再启动节点。不建议先定义模块再将模块内的方法添加到函数视图，而应该直接定义方法再添加到函数视图。

#### 2.1.7 定时任务文件损坏

DolphinDB 序列化定时任务到文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致定时任务文件损坏，可能导致节点启动时反序列化失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:57:30.456789 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: 
    Failed to deserialize update statement
2023-10-13 09:57:30.456789 <ERROR> :Failed to unmarshall the job [myTest].
    Failed to deserialize update statement. Invalid message format
```

此时如果希望尽快启动节点，可以移走 server/clusterDemo/data/dnode1/sysmgmt 下的 jobEditlog.meta 、 jobCheckPoint.meta 、 jobCheckPoint.tmp 文件以跳过启动时的定时任务反序列化，再启动节点。
注意 ：必然存在 jobEditlog.meta ，可能存在 jobCheckPoint.meta 、 jobCheckPoint.tmp 。
请打包定时任务文件、日志里报错的定时任务脚本和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，需要重新提交所有定时任务。

#### 2.1.8 权限与函数视图文件损坏

DolphinDB 序列化权限操作和函数视图定义到文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致权限和函数视图文件损坏，可能导致节点启动时反序列化失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:59:35.786438 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception:
    Failed to deserialize sql query object
2023-10-13 09:59:35.786438 <ERROR> :Failed to unmarshall the job [myTest1].
    Failed to deserialize sql query object. Invalid message format
```

此时如果希望尽快启动节点，可以参考 函数视图或定时任务包含不存在的共享表 中的 问题 2：函数视图反序列化失败 小节的解决方法移走权限与函数视图文件以跳过反序列化。
请打包权限和函数视图文件、日志里报错的函数视图脚本和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，需要重新添加所有权限和函数视图定义。

#### 2.1.9 RAFT 文件损坏

注意 ：下面介绍的操作需要确保已经有另一个节点成为 RAFT 集群 leader 才可以进行。DolphinDB 写 RAFT 元数据和日志时，如果上次运行时出现磁盘满、宕机或 bug，可能导致 RAFT 元数据或日志文件损坏，可能导致控制节点启动时恢复 RAFT 数据失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:59:35.786438 <WARNING> :[Raft] incomplete hardstate file 
    [/data/server/data/controllerl/raft/raftHardstatel]
2023-10-13 09:59:35.786438 <INFO> :[Raft] Group DFSMaster RaftWAL::reconstruct: 
    read new file with 83213 entries 
2023-10-13 09:59:35.786438 <ERROR> :[Raft] Group DFSMaster RawNode::init: 
    failed to initialize with exception [basic_string::_S_create].
2023-10-13 09:59:35.786438 <ERROR> :Failed to start DFSMaster with the error message:
    basic_string::_S_create
```

此时如果希望尽快启动节点，可以移走问题节点的 dfsMetaDir 配置项 文件夹和 <HOME_DIR>/<NodeAlias>/raft 文件夹以跳过启动时的 DFS 和 RAFT 元数据初始化，再启动节点。
请打包 dfsMetaDir 配置项 文件夹、 <HOME_DIR>/<NodeAlias>/raft 文件夹和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，会自动同步 leader 节点的元数据。

### 2.2 启动异常卡住


#### 2.2.1 集群间网络不通

在多台机器上部署 DolphinDB 时，需要确保集群间各个节点的 IP:端口号互通，否则会导致节点启动失败。常见原因和现象是调整了机器网络配置后重启高可用集群，进入控制节点 web 管理界面白屏，查看控制节点日志中有如下报错：

```
2023-11-01 16:00:34.992500 <INFO> :New connection from ip = 192.168.0.44 port = 35416
2023-11-01 16:00:35.459220 <INFO> :DomainSiteSocketPool::closeConnection: 
    Close connection to ctl1 #44 with error: epoll-thread: Read/Write failed.
    Connection timed out. siteIndex 0, withinSiteIndex 44
```

或在 web 启动数据节点时报错 IO error type 1 （含义为 Socket is disconnected/closed or file is closed ，即网络连接断开）。
需要联系运维调通网络。

#### 2.2.2 RSA 密钥校验文件损坏

DolphinDB 生成 RSA 密钥校验文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致 RSA 密钥校验文件损坏，使节点无法正常通过 RSA 密钥来通信，进而导致启动流程卡住。查看节点运行日志有如下报错：

```
2023-10-25 11:55:04.987161 <ERROR> :Failed to decrypt the message by RSA public key.
```

解决方案是删除所有控制节点的 <HOME_DIR>/<NodeAlias>/keys 目录并重启集群，以触发 RSA 密钥校验文件重新生成。 注意删除 keys 文件夹后需要重新提交集群的所有定时任务。

### 2.3 启动慢


#### 2.3.1 正在回滚事务，如何查看进度，如何删除事务 log 以跳过

DolphinDB 启动时，如果日志里有 Will process pending transactions. 而没有 ChunkMgmt initialization completed. 说明正在回滚事务。如果节点宕机时的写入事务涉及的分区过多或数据量过大，可能导致事务回滚时间较长。
查看回滚进度的方法如下，其中， chunkMetaDir 是 OLAP 引擎节点的元数据文件的存储路径，具体可查看 功能配置 ：
- 查看 <chunkMetaDir>/LOG 目录下以 tid 命名的文件夹数目是否减少，这里只能根据文件夹减少速度估计回滚速度；
- 若关机前有删除事务，查看 <Alias>/<volumes>/LOG 目录下以 tid 命名的文件夹数目是否减少，文件夹数目为 0 时回滚完毕。
注意 ：统计文件数目可以使用 Linux 命令： ll -hrt <chunkMetaDir>/LOG | wc -l
强烈建议等待事务回滚完成 ，跳过事务回滚会使原本应该回滚的事务不回滚，导致事务相关的数据或元数据错误。若客户希望尽快启动节点且 不需要保证节点宕机前正在写入的数据完整性 ，可以通过如下步骤跳过事务回滚：
- 使用 kill -15 pid 安全关闭节点，若正在启动时无法关闭则使用 kill -9 强制关闭节点，因为正在启动所以不会有新的写入；
- mv 移走 <chunkMetaDir>/LOG 和 <Alias>/<volumes>/LOG 文件夹；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。

#### 2.3.2 正在回放 redo log，如何查看进度，如何删除 redo log 以跳过

DolphinDB 启动时，如果日志里有 Start recovering from redo log. This may take a few minutes. 而没有 Completed CacheEngine GC and RedoLog GC after applying all redo logs and engine is <engineType> ，说明正在回放 redo log。期间会刷带“RedoLog”的日志例如：

```
"applyTidRedoLog : 20716,f7dbaef9-05bc-10b6-f042-a14bc0e9c897,
    /home/DolphinDB/server/clusterDemo/data/node2/storage/CHUNKS/snapshotDB
    /20220803/Key17/5o7,shfe_5,166259,107,0"
```

注意：如果是 DolphinDB 2.0 版本，会有两次 redo log 回放，对应 OLAP 和 TSDB 存储引擎，故对应的日志也有两份；如果是 DolphinDB 1.0 版本，只会有一次 OLAP 存储引擎的 redo log 回放。
- 统计 redoLogDir 和 TSDBRedoLogDir 下的 tid.log 文件数目，为 0 时回放完毕；
- 统计 redoLogDir 和 TSDBRedoLogDir 文件夹大小，除以硬盘的读速率，可估计最快的回放完成时间。
注意 ：统计文件夹大小可以使用 Linux 命令： du -sh <redoLogDir>
强烈建议等待 redo log 回放完成 ，跳过 redo log 回放会使原本应该回放的事务不回放，会导致事务相关的数据或元数据错误。若客户希望尽快启动节点且 不需要保证节点宕机前正在写入的数据完整性 ，可以通过如下步骤跳过 redo log 回放：
- 使用 kill -15 pid 安全关闭节点，若正在启动时无法关闭则使用 kill -9 强制关闭节点，因为正在启动所以不会有新的写入；
- mv 移走 redoLogDir 和 TSDBRedoLogDir 文件夹下的 head.log 文件；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。

### 2.4 其他问题


#### 2.4.1 启动脚本运行慢或失败

启动脚本 startup.dos 运行慢会导致启动流程走不到初始化定时任务一步，但由于节点在 redo log 回放完成后即在 web 集群管理界面转变状态为绿色，已经可以访问，故会导致定时任务相关功能在 startup.dos 执行完前无法使用。
启动脚本 startup.dos 或 postStart.dos 运行失败会在节点运行日志里打印错误日志，然后跳过启动脚本报错行后的执行，不会导致节点启动失败，也不会回滚执行失败的动作，需要客户自行考虑启动脚本运行失败的情况。注意在 集群模式下，执行启动脚本时无法保证分布式数据库已初始化完毕 ，在启动脚本访问分布式库表可能会报错。
故不建议在启动脚本执行太慢的操作或涉及分布式库表的操作，而只做一些比较简单的操作如建共享表、加载插件等。启动脚本详细介绍见 启动脚本 。
可以参考如下脚本在 startup.dos 或 postStart.dos 等待分布式模块准备完毕：

```
def isClusterOk() {
    do {
        try {
            meta = rpc(getControllerAlias(), getClusterChunksStatus)
            configReplicaCount = 2 // 需要修改为 dfsReplicationFactor 配置项值

            cnt1 = exec count(*) from meta where state != "COMPLETE"
            cnt2 = exec count(*) from meta where replicaCount != configReplicaCount
  
            if (cnt1 == 0 and cnt2 == 0) {
                break
            } else {
                writeLog("startup isClusterOk: state != 'COMPLETE' cnt: " + string(cnt1) + ",
                " + "replicaCount != " + string(configReplicaCount) + " cnt: " + string(cnt2))
            }
        } catch (err) {
            writeLog("startup isClusterOk: " + err)
        }

        sleep(3*1000)
    } while(1)

    return true
}

res = isClusterOk()
writeLog("startup isClusterOk: ", string(res))
```



---

> 来源: `how_to_handle_crash.html`


# 节点宕机

在使用 DolphinDB 时，有时客户端会抛出异常信息：Connection refused。此时，Linux 操作系统上使用 ps 命令查看，会发现 DolphinDB 进程消失。本教程针对出现这种情况的各种原因进行定位分析，并给出相应解决方案。

## 1. 查看节点日志排查原因

DolphinDB 每个节点的运行情况会记录在相应的日志文件中。通过分析日志，能有效地掌握 Dolphindb 运行状况，从中发现和定位一些错误原因。当节点宕机时，非 DolphinDB 系统运行原因导致节点关闭的情形通常有以下三种：
- Web 集群管理界面上手动关闭节点或调用 stopDataNode 函数停止节点
- 操作系统 kill 命令杀死节点进程
- license 有效期到期关机
通过查看节点在退出前是否打印了日志：MainServer shutdown 进行盘查。下面假设宕机节点为 datanode1 ，操作命令示例如下：

```
less datanode1.log|grep "MainServer shutdown"
```

若命令执行结果显示如下图，推算系统退出的时间点也合理，可初步认定为系统主动退出。
下面分情况讨论进一步的排查步骤。
第一种：查看控制节点日志 controller.log ，判断是否为 Web 集群管理界面上手动关闭节点或调用 stopDataNode 函数停止节点，操作命令示例如下：

```
less controller.log|grep "has gone offline"
```

若日志产生的信息如上图，可判断为通过 Web 界面手动关闭节点或调用 stopDataNode 函数停止节点。
第二种：查看对应宕机节点运行日志是否含有 Received signal 信息，判断是否为操作系统 kill 命令杀死节点进程。操作命令示例如下：

```
less datanode1.log|grep "Received signal"
```

若相应时间点上有上图所示信息，则可判定为进程被 kill 命令杀掉。
第三种：查看节点运行日志是否打印了如下信息：The license has expired，判断是否为 license 到期而关机：
若出现上图信息，可以判断是 license 到期退出。用户需要关注 license 有效期日期，及时更新 liense 以避免出现因 license 到期而无法使用的问题。从1.30.11，1.20.20开始，DolphinDB 支持不重启系统，在线更新 license。但是，DolphinDB 1.30.11及1.20.20之前的版本因为不支持在线更新 license，更新 license 后，需要重启数据节点、控制节点和代理节点，否则会导致数据节点退出。
注意：节点日志存放路径为：单节点模式默认在安装目录 server 下，文件名为 dolphindb.log，集群默认在 server/log 目录下。若需要修改日志文件路径，只能在命令行中通过 logFile 配置项指定。如果集群内有节点在运行，可通过如下 ps 命令查看每一个节点对应的 log 生成位置，其中 logFile 参数表示日志文件的路径和名称：
如果未查询到上述的日志信息，需进一步查看相应操作系统日志。

## 2. 查看操作系统日志排查 OOM killer

Linux 内核提供了一种名为 OOM killer（Out Of Memory killer）的机制，用于监控占用内存过大的进程，尤其是瞬间占用大量内存的进程，以防止内存耗尽而自动把该进程杀掉。排查 OOM killer 可用 dmesg 命令，示例如下：

```
dmesg -T|grep dolphindb
```

如上图，若出现了“Out of memory: Kill process”，说明 DolphinDB 使用的内存超过了操作系统所剩余的空闲内存，导致操作系统杀死了 DolphinDB 进程。解决这种问题的办法是：通过参数 maxMemSize （单节点模式修改 dolphindb.cfg，集群模式修改 cluster.cfg）设定节点的最大内存使用量。需要合理设置该参数，设置太小会严重限制集群的性能；设置太大可能触发操作系统杀掉进程。若机器内存为16GB，并且只部署1个节点，建议将该参数设置为12GB左右。
使用上述 dmesg 命令显示日志信息时，若看到如下图所示的“segfault”，就是发生了段错误，即应用程序访问的内存超出了系统所给的内存空间：
可能导致段错误的原因有：
- DolphinDB 访问系统数据区，最常见的就是操作0x00地址的指针
- 内存越界（数组越界，变量类型不一致等）：访问到不属于 DolphinDB 的内存区域
- 栈溢出（Linux 一般默认栈空间大小为8192kb，可使用 ulimit -s 命令查看）
此时若正确开启了 core 功能，会生成相应的 core 文件，就可以使用 gdb 对 core 文件进行调试。

## 3. 查看 core 文件

在程序运行过程中，若发生异常导致程序终止，操作系统会自动记录程序终止时的内存状态，并将其保存至一个特定的文件中。这一过程被称为core dump，也可译为“核心转储”。尽管 core dump 常被视作一种“内存快照”，但它所包含的信息远不止内存数据。core dump 还会同时记录其他关键的程序运行状态，如寄存器信息（含程序指针、栈指针等）以及其他处理器和操作系统的状态信息。对于编程人员来说，诊断和调试程序时，core dump 是一个极其有用的工具，因为某些程序错误，例如指针异常，可能在现实环境中难以复现，但 core dump 文件能够提供程序出错时的详细上下文，从而帮助开发人员准确定位问题。core 文件默认生成位置与可执行程序位于同一目录下，文件名为 core.***，其中***是一串数字。

### 3.1 core 文件设置


#### 3.1.1 控制 core 文件生成

首先在终端中输入 ulimit -c 查看是否开启 core 文件。若结果为 0，则表示关闭了此功能，不会生成 core 文件。结果为数字或者" unlimited "，表示开启了 core 文件。因为 core 文件比较大，建议设置为" unlimited "，表示 core 文件的大小不受限制。可用下列命令设置：

```
ulimit -c unlimited
```

以上配置只对当前会话起作用，重新登录时，需要重新配置。以下两种方式可使配置永久生效：
- 在 /etc/profile 中增加一行 ulimit -S -c unlimited >/dev/null 2>&1 后保存退出，重启 server；或者不重启 server，使用 source /etc/profile 使配置马上生效。 /etc/profile 对所有用户有效，若想只针对某一用户有效，则修改此用户的 ~/.bashrc 或者 ~/.bash_profile 文件。
- 在 /etc/security/limits.conf 最后增加如下两行记录：为所有用户开启 core dump。
注意： 由于 DolphinDB 数据节点是通过代理节点进行启动的，在不重启代理节点的情况下，无法使开启 core 功能生效。因此，开启 core 功能后需要重启代理节点，再重启数据节点。

#### 3.1.2 修改 core 文件保存路径

core 文件默认的文件名称是 core.pid ，其中 pid 是指产生段错误程序的进程号。core 文件默认路径是该程序的当前目录。
/proc/sys/kernel/core_uses_pid 可控制产生的 core 文件名中是否添加 pid 作为扩展。1表示添加，0表示不添加。通过以下命令进行修改：

```
echo "1" > /proc/sys/kernel/core_uses_pid
```

/proc/sys/kernel/core_pattern 可设置 core 文件保存的位置和文件名。但保存路径可能在机器重启后失效，若想使设置永久生效，需要在 /etc/sysctl.conf 文件中添加 kernel.core_pattern=/data/core/core-%e-%p-%t.core。通过以下命令将 core 文件存放到 /corefile 目录下，产生的文件名为：core-命令名-pid-时间戳：

```
echo /corefile/core-%e-%p-%t > /proc/sys/kernel/core_pattern
```

上述命令参数列表如下:
- %p - insert pid into filename 添加pid
- %u - insert current uid into filename 添加当前uid
- %g - insert current gid into filename 添加当前gid
- %s - insert signal that caused the coredump into the filename 添加导致产生core的信号
- %t - insert UNIX time that the coredump occurred into filename 添加core文件生成时的unix时间
- %h - insert hostname where the coredump happened into filename 添加主机名
- %e - insert coredumping executable name into filename 添加命令名
一般情况下，无需修改参数，按照默认的方式即可。可根据实际环境修改 core 文件路径及名称。
- 设置 core 文件保存路径时首先要创建这个保存路径，并确保运行进程的用户对此目录有读写权限，否则 core 文件无法生成，建议将此目录的权限设置为 777。 chmod 777 /path2/corefile
- 要注意所在磁盘的剩余空间大小，不要影响 DolphinDB 元数据、分区数据的存储。

#### 3.1.3 测试core文件是否能够生成

使用下列命令对某进程（用 ps 查到进程号为24758）使用 SIGSEGV 信号，可以 kill 掉这个进程：

```
kill -s SIGSEGV 24758
```

若在 core 文件夹下生成了 core 文件，表示设置有效。

### 3.2 gdb 调试 core 文件

若 gdb 没有安装，需要先安装。以 Centos 系统为例，安装 gdb 用命令如下：

```
yum install gdb
```

gdb 调试命令格式： gdb [exec file] [core file] ，然后执行 bt 看堆栈信息：

### 3.3 core 文件不生成的情况

Linux中信号是一种异步事件处理的机制，每种信号对应有默认的操作，可以在 这里 查看 Linux 系统提供的信号以及默认处理。默认操作主要包括忽略该信号（Ingore）、暂停进程（Stop）、终止进程（Terminate）、终止并发生 core dump 等。如果信号均是采用默认操作，那么，以下几种信号发生时会产生 core dump:
| Signal | Action | Comment |
| --- | --- | --- |
| SIGQUIT | core | Quit from keyboard |
| SIGILL | core | Illegal Instruction |
| SIGABRT | core | Abort signal fromabort |
| SIGSEGV | core | Invalid memory reference |
| SIGTRAP | core | Trace/breakpoint trap |
当然，Linux 提供的信号不仅限于上面的几种。还有一些信号是无法产生 core dump 的，比如：
(1) 使用 Ctrl+z 来挂起一个进程或者 Ctrl+C 结束一个进程均不会产生 core dump，因为前者会向进程发出 SIGTSTP 信号，该信号的默认操作为暂停进程(Stop Process)；后者会向进程发出 SIGINT 信号，该信号默认操作为终止进程(Terminate Process)。
(2) kill -9 命令会发出 SIGKILL 命令，该命令默认为终止进程，同样不会产生 core 文件。
(3) 查看 DolphinDB 对应日志，假设日志信息出现“Received signal 15”，如下图，即用 SIGTERM （kill命令默认信号）杀死 DolphinDB 进程，不会生成 core 文件。
以下情况也不会产生 core 文件：
(a) 进程是设置-用户-ID，而且当前用户并非程序文件的所有者；
(b) 进程是设置-组-ID，而且当前用户并非该程序文件的组所有者；
(c) 用户没有写当前工作目录的许可权；
(d) core 文件太大。

## 4. 避免节点宕机

在企业生产环境下，DolphinDB 往往作为流数据中心以及历史数据仓库，为业务人员提供数据查询和计算。当用户较多时，不当的使用容易造成 server 端宕机。遵循以下建议，可尽量避免不当使用。

### 4.1 避免无限递归

用户在自定义递归函数时，如果不设置终止条件，就会导致无限递归。每次递归调用都会消耗堆栈空间，当堆栈空间耗尽时，就会引发堆栈溢出错误，最终导致进程终止。
例如调用以下递归函数会导致节点宕机，应避免类似定义。

```
def danger(x){
  return danger(x)+1
}

danger(1)
```


### 4.2 监控内存使用，避免内存高位运行

内存高位运行容易发生OOM，如何监控和高效使用内存详见 内存管理 中为分布式数据库提供写入缓存小节与流数据消息缓存队列小节

### 4.3 避免多线程并发访问某些内存表

DolphinDB内置编程语言，在操作访问表时有一些规则，如 内存表详解 并发性小节就说明了有些内存表不能并发写入。若不遵守规则，容易出现 server 崩溃。下例创建了一个以 id 字段进行 RANGE 分区的常规内存表：

```
t=table(1:0,`id`val,[INT,INT])
db=database("",RANGE,1 101 201 301)
pt=db.createPartitionedTable(t,`pt,`id)
```

下面的代码启动了两个写入任务，并且写入的分区相同，运行后即导致 server 系统崩溃。

```
def writeData(mutable t,id,batchSize,n){
	for(i in 1..n){
		idv=take(id,batchSize)
		valv=rand(100,batchSize)
		tmp=table(idv,valv)
		t.append!(tmp)
	}
}

job1=submitJob("write1","",writeData,pt,1..300,1000,1000)
job2=submitJob("write2","",writeData,pt,1..300,1000,1000)
```

产生的 core 文件如下图：

### 4.4 自定义开发插件俘获异常

DolphinDB 支持用户自定义开发插件以扩展系统功能。插件与 DolphinDB server 在同一个进程中运行。若插件崩溃，整个系统（server）就会崩溃。因此，在开发插件时要注意完善错误检测机制。除了插件函数所在线程可以抛出异常（server 在调用插件函数时俘获了异常），其他线程都必须自己俘获异常，不得抛出异常。详情请参阅 DolphinDB Plugin 。

## 5. 总结

分布式数据库 DolphinDB 的设计十分复杂，发生宕机的情况各有不同。若发生节点宕机，请按本文所述一步步排查：
- 排查是否为系统主动退出，如是否通过 Web 集群管理界面手动关闭节点或调用 stopDataNode 函数停止节点，是否用 kill 命令杀死节点进程，是否为 license 有效期到期退出等。当这些原因导致数据库宕机时，系统运行日志会记录相关的信息，请检查对应时间点是否有相应的操作；
- 排查是否因为内存耗尽而被操作系统杀掉。排查 OOM killer 可查看操作系统日志；
- 检查是否有使用不当的脚本，如多线程并发访问了没有共享的分区表、插件没有俘获一些异常等；
- 请查看 core 文件，用 gdb 调试获取堆栈信息。若确定系统有问题，请保存节点日志、操作系统日志和 core 文件，并及时与 DolphinDB 工程师联系。


---

> 来源: `node_startup_process_and_questions.html`


# 节点启动流程简析与常见问题

DolphinDB 的重启是运维工作的重要部分，在启动节点时可能会遇到一些问题，例如启动太慢、启动失败等。本教程以 DolphinDB v2.00.11 版本为例，结合运行日志简析 DolphinDB 整体的启动流程和重要模块的启动流程，并分析启动时常见问题的现象、原因和解决方案。

## 1. 节点整体启动流程

DolphinDB 节点整体的启动流程可分为 7 个阶段：
- 初始化内部基础模块；
- 解析和校验参数、配置文件、license 文件，并初始化和启动一些基本功能模块和线程；
- 根据加载的配置文件内容初始化 server，执行 dolphindb.dos ，加载 preloadModules 配置的插件和模块；
- 初始化和启动 server 的各个功能模块和线程，绑定端口。包括用户权限与函数视图初始化、元数据初始化、事务回滚、redo log 回放、RAFT 初始化等；
- 执行 startup 配置项 指定的 startup.dos 脚本；
- 初始化定时任务；
- 执行 postStart 配置项 指定的 postStart.dos 脚本。
其中第 4 步，不同节点会根据自己的职能初始化和启动相应功能的模块和线程。例如单节点和控制节点会启动 DFS 模块来管理分布式文件元数据，而数据节点不会启动；单节点和数据节点会启动 ChunkNode 模块来存储和管理分区数据，而控制节点不会启动。
重要启动流程开始和成功日志如下表：
| 流程 | 开始日志 | 成功日志 | 备注 |
| --- | --- | --- | --- |
| 用户权限与函数视图初始化 | Initializing AclManager with Raft <raft mode> | Initialization of AclManager is completed with Raft <raft mode> | RAFT mode 为 enabled 或 disabled |
| 控制节点元数据初始化 |  | 非 RAFT 模式：Controller initialization completed.RAFT 模式：DFSRaftReplayWorker started |  |
| 数据节点分区元数据初始化 |  | ChunkMgmt initialization completed. |  |
| 数据节点恢复事务的重做日志回放 | dfsRecoverLogDir: <dfsRedoLogDir> | RecoverRedoLogManager finished chunk num is <finishPacksSize>, total chunks recover num is <lsnChunk |  |
| 数据节点 TSDB 元数据初始化 |  | Restore TSDB meta successfuly. |  |
| redo log 回放 | 若存在 redo log：Start recovering from redo log. This may take a few minutes. | 若存在 redo log：Completed CacheEngine GC and RedoLog GC after applying all redo logs and engine is <eng | engine type 为 OLAP 或 TSDB，故如果为 DolphinDB 2.0 版本，相应日志会有两条 |
| RAFT 初始化 | DFSMaster ElectionTick is set to <electionTick> | DFSRaftReplayWorker started |  |
| 执行startup.dos | Executing the startup script: <scriptFile> | The startup script: <scriptFile> execution completed. |  |
| 定时任务初始化 | Job scheduler start to initialize. | Job scheduler initialization completed. |  |
| 执行postStart.dos | Executing the post start script: <scriptFile> | The post start script: <scriptFile> execution completed. |  |
Redo log 回放完成后，数据节点会向控制节点汇报本地分区信息，然后启动心跳进程，此时节点在 web 集群管理界面的状态会转变为绿色，但还没有到“执行 startup.dos ”一步。由于执行 postStart.dos 失败并不会导致节点启动失败，故 节点启动完成的标志日志 为 Job scheduler initialization completed.

## 2. 重要启动流程简析


### 2.1. license 校验

DolphinDB 启动时会先校验 license 与集群配置信息是否合规。如果启用了 license server，则连接 license server 进行校验，否则读取安装目录下 dolphindb.lic 文件进行校验。校验内容包括：
- 过期时间
- 每个节点绑定的核数
- 每个节点的最大内存
- 集群内最大节点数
- 最高支持的 server 版本
如果校验失败，会打印相应的错误日志然后节点关闭。例如 license 过期时会打印错误日志 <ERROR> The license has expired.

### 2.2. 用户权限与函数视图初始化

DolphinDB 的 权限管理 与 函数视图 定义持久化保存在单节点或控制节点的数据目录，根据是否启用 RAFT 高可用存储在不同的位置。非 RAFT 模式（即单节点或普通集群）时，存储在控制节点的 <HOME_DIR>/<NodeAlias>/sysmgmt 路径下，相关文件说明如下表：
| 文件名 | 说明 |
| --- | --- |
| aclEditlog.meta | 权限与函数视图的编辑日志文件 |
| aclCheckPoint.meta | 权限与函数视图的 checkpoint 文件，当编辑日志过大时会做一次 checkpoint 保存最新的权限与函数视图的状态 |
| aclCheckPoint.tmp | 权限与函数视图的 checkpoint 临时文件，做 checkpoint 时临时生成 |
注意 ：HOME_DIR 指节点的主目录，即 getHomeDir() 方法返回结果；NodeAlias 指节点别名，即 getNodeAlias() 方法返回结果。
RAFT 模式（即高可用集群）时，存储在 leader 控制节点的 <HOME_DIR>/<NodeAlias>/raft 路径下，详见 RAFT 元数据初始化一节。
非 RAFT 模式初始化流程如下：
- 开始，打印日志： Initializing AclManager with Raft <raft mode> ；
- 如果不是高可用集群： a. 读取节点数据目录 /sysmgmt ，如果不存在则创建； b. 如果不存在 aclEditlog.meta 但存在有效的 aclCheckPoint.tmp ，重命名 aclCheckPoint.tmp 为 aclEditlog.meta ； c. 如果存在 aclCheckPoint.meta ，检查校验； d. 读取 aclCheckPoint.meta 文件恢复用户权限和函数视图； e. 读取和回放 aclEditlog.meta 文件恢复用户权限和函数视图； f. 裁剪 aclEditlog.meta 中的无效部分；
- 如果不存在 RSA 校验密钥（ aclPublic.key 和 aclPrivate.key ，存储路径为 <HOME_DIR>/<NodeAlias>/keys ，用于集群间加密通信），如果不是高可用集群，生成 RSA 校验密钥；如果是高可用集群，则以控制节点名字母表顺序从其他控制节点拿密钥；
- 如果不存在 admin 用户，创建 admin 用户；
- 结束，打印日志： Initialization of AclManager is completed with Raft <raft mode> 。

### 2.3. 控制节点元数据初始化

DolphinDB 的分布式文件系统（DFS）管理集群的所有分区数据的元数据，元数据根据是否启用 RAFT 高可用存储在不同的位置。非 RAFT 模式（即单节点或普通集群）时，存储在单节点或控制节点的 dfsMetaDir 配置项 路径下，相关文件说明如下表：
| 文件名 | 说明 |
| --- | --- |
| DFSMetaLog.cid | DFS 元数据的编辑日志文件。其中 cid 指事务提交 ID |
| DFSMasterMetaCheckpoint.cid | DFS 元数据的 checkpoint 文件，当编辑日志过大时会做一次 checkpoint 保存最新的元数据的状态 |
| DFSMasterMetaCheckpoint.cid.tmp | DFS 元数据的 checkpoint 临时文件，做 checkpoint 时临时生成 |
RAFT 模式（即高可用集群）时，存储在 leader 控制节点的 <HOME_DIR>/<NodeAlias>/raft 路径下，详见 RAFT 初始化一节。
非 RAFT 模式初始化流程如下：
- 读取节点 dfsMetaDir 配置项目录，如果不存在则创建；
- 如果存在临时 checkpoint 文件，尝试解析，如果文件无效，只保存文件名的 cid 信息然后删除；否则重命名为 checkpoint 文件；
- 如果存在 checkpoint 文件，尝试解析，如果解析失败则程序退出；否则读取 checkpoint 信息。相关运行日志： checkpoint file <checkpointFilepath> metalog file <metalogFilepath>
- 读取和回放 checkpoint 的 cid 开始的所有编辑日志。相关运行日志： ---------start replaying edit log, <metaFileNum> log files to replay---------- done reading editlog file <filepath>, processsed <num> records ---------done replaying edit log, <num> record(s) processed----------- Global cid <currentGlobalCID> Global tid <currentGlobalTID>, reader snapshot id <currentGlobalTID - 1>
- 清理过期无效的 DFSMetaLog 。
- 结束，打印日志 Controller initialization completed. 。

### 2.4. 数据节点元数据初始化

DolphinDB 单节点或数据节点存储和管理本地分区数据及元数据，启动时会先恢复本地分区的元数据，然后根据事务 log 回滚关机时未完成的事务，上报已提交的事务到控制节点。相关文件存储在 chunkMetaDir 配置项 路径和 volumes 配置项 路径下，说明如下：
| 文件名 | 存储路径 | 说明 |
| --- | --- | --- |
| editlog.cid | <chunkMetaDir>/CHUNK_METADATA | 本地元数据的编辑日志文件。其中 cid 指事务提交 ID |
| checkpoint.cid | <chunkMetaDir>/CHUNK_METADATA | 本地元数据的 checkpoint 文件，当编辑日志过大时会做一次 checkpoint 保存最新的元数据的状态 |
| checkpoint.tmp.cid | <chunkMetaDir>/CHUNK_METADATA | 本地元数据的 checkpoint 临时文件，做 checkpoint 时临时生成 |
| 以 tid 为名的文件夹 | <chunkMetaDir>/LOG | 事务 log。其中 tid 为事务 ID |
| 以 tid 为名的文件夹 | <volumes>/LOG | 删除事务涉及的数据文件临时存放的目录 |
- 如果存在 checkpoint 临时文件，校验是否有效，若有效，重命名为 checkpoint 文件；
- 找到 tid 最大的 checkpoint 文件并读取元数据状态；
- 读取 tid 大于等于最大的 checkpoint tid 的 editlog 编辑日志并回放，相关运行日志： Opened editlog file <editLogFile>.
- 回滚未提交的事务，相关运行日志： Will process pending transactions. Processing transactions took <time> seconds. Will process uncommited and committed but not completed transactions.
- 删除已回滚的事务 LOG 文件夹，相关运行日志： As transaction <tid> is rollbacked, transaction log directory <logDir> is deleted.
- 完成，打印日志 ChunkMgmt initialization completed.。

### 2.5. 数据节点恢复事务的重做日志回放

DolphinDB 集群支持 节点间数据恢复 功能，若配置了 enableDfsRecoverRedo = true ，在节点间数据恢复的过程中，会将恢复事务相关的数据先写入 recover redo log 中，然后在启动时回放恢复事务的重做日志。相关文件存储在 recoverLogDir 配置项 目录下，说明如下：
| 文件名 | 说明 |
| --- | --- |
| recover.log | 恢复事务重做日志 |
| recover.log.tmp | 生成恢复事务重做日志时的临时文件 |
- 如果同时存在恢复事务重做日志和对应临时文件，删除临时文件；
- 从恢复事务重做日志中解析需要回放的恢复事务，相关运行日志： RecoverRedoLogManager will recover chunk num=<chunkLsnSize>, total recover pack num=<allPacksSize>, skip garbage pack num=<garbageLsnsSize> will redo pack num=<size>
- 重做恢复事务，相关运行日志： RecoverRedoLogManager finished chunk num is <finishPacksSize>, total chunks recover num is <lsnChunksSize>, compare checksum succ pack num is <recoverChecksumPackNum>, test replay num is <recoverTestPackNum>, actual recover redo pack num is <recoverReplayPackNum>, redo chunk num is <recoverReplayNum>
- 清理无用的恢复日志。

### 2.6. 数据节点 TSDB 元数据初始化

DolphinDB 2.0 版本支持 TSDB 存储引擎，启动时会恢复 TSDB 的 level file 相关元数据。相关文件存储在 TSDBRedoLogDir 配置项 的同级目录的 TSDBMeta 目录下，说明如下：
| 文件名 | 说明 |
| --- | --- |
| iotEditLog.meta | TSDB 元数据的编辑日志文件。其中 cid 指事务提交 ID |
| iotCheckPointFile.meta | TSDB 元数据的 checkpoint 文件，当编辑日志过大时会做一次 checkpoint 保存最新的元数据的状态 |
| iotCheckPointerFile.tmp | TSDB 元数据的 checkpoint 临时文件，做 checkpoint 时临时生成 |
- 如果存在 checkpoint 临时文件，校验是否有效，若有效，重命名为 checkpoint 文件并删除编辑日志；若无效，则删除 checkpoint 临时文件，相关运行日志： ------RESTORE <tmpCheckPointFile> size <tmpLength>, <checkPointFile> size <length>, <editLogFile> size <editLength>; ------RESTORE rename tmp file<tmpCheckPointFile> to <checkPointFile>
- 读取 checkpoint 文件恢复元数据状态，相关运行日志： ------RESTORE checkpoint file to validPos <validPos>
- 读取和回放编辑日志文件恢复元数据状态，相关运行日志： ------RESTORE editLogFile file to validPos <validPos>
- 完成，打印日志 Restore TSDB meta successfuly. 。

### 2.7. 数据节点 redo log 回放

DolphinDB 通过 redo log 来实现意外重启时对已提交但未完成事务的回放。DolphinDB v2.00.10.3 版本对 append, tableInsert, insert into 等新增数据的写入操作支持 redo log。OLAP 存储引擎的 redo log 存储路径在 redoLogDir 配置项 目录下，TSDB 存储引擎的 redo log 存储路径在 TSDBRedoLogDir 配置项 目录下，相关文件：
| 文件名 | 说明 |
| --- | --- |
| head.log | redo log 元数据信息 |
| head.log.tmp | 生成 redo log 元数据信息时的临时文件 |
| lsn.log | redo log 全局序列号 |
| tid.log | 未完成的事务数据信息 |
- 如果不存在 redo log 文件夹或 redo log 文件夹为空，跳过 redo log 回放，打印日志： No available redo log files were found. Redo log replay was skipped. ；
- 开始回放，打印日志： Start recovering from redo log. This may take a few minutes. ；
- 如果存在 head.log.tmp ，删除；
- 解析 head.log 得到 redo log 的元数据信息，包含可能需要回放的事务（已提交的）和不需要回放的事务（未提交或已完成的）；
- 遍历可能需要回放的事务的 redo log，结合多方面信息具体判断是否需要回放，对于状态完成的事务立即回放，状态不确定的事务保留相关信息以备决议，相关运行日志： applyTidRedoLog : <tid>,<chunkId>,<chunkPath>,<tablePhysicalName>,<newTableSize>,<lsn>,<newColumns>
- 结束回放，打印日志： Completed CacheEngine GC and RedoLog GC after applying all redo logs and engine is <engineType> 。
注意 ：redo log 与 cache engine 的具体功能介绍见 redo log 和 cache engine 。有后台线程定期自动清理不再需要的 tid.log 文件。

### 2.8. 控制节点 RAFT 元数据初始化

DolphinDB 的高可用集群通过 RAFT 管理 DFS 元数据、权限和函数视图数据等控制节点的元数据。Raft 日志存储在控制节点的 <HOME_DIR>/<NodeAlias>/raft 目录下，相关文件说明：
| 文件 | 说明 |
| --- | --- |
| raftHardstate[group] | RAFT 的任期（term）和投票相关元数据信息。其中 group 为 RAFT 组号 |
| raftWAL[group] | RAFT 业务数据日志，含其他使用 RAFT 模块的数据，如 DFS 元数据、权限和函数视图数据等。 |
| raftSnapshot[group] | RAFT 业务数据日志的快照 |
| raftWAL[group].old | 旧的业务数据日志，生成快照时的临时文件 |
| raftSnapshot[group].tmp | 生成快照时的临时文件 |
- 开始，打印日志： DFSMaster ElectionTick is set to [electionTick] ；
- 从 controller.cfg 读取 RAFT 集群信息；
- 从 raftHardstate 读取 RAFT 成员相关的信息；
- 从 raftSnapshot 读取业务日志的快照（可以理解为 checkpoint），清理快照临时文件；
- 从 raftWAL 读取和重做业务日志（包含 dfsMeta, acl log, streamingHA log 等），相关运行日志： <Raft> Group <groupAlias> RaftWAL::reconstruct: read new file with <size> entries <Raft> Group <groupAlias> RaftWAL::reconstruct: read old file with <size> entries
- 初始化节点成为 follower，加入 RAFT 集群，相关运行日志： <Raft> Group <groupAlias> <NodeAlias> became follower at term <currentTerm>, leader is <leader> <Raft> Group <groupAlias> <NodeAlias> begin to clear all old notifiers
- 完成，打印日志： <Raft> Group <groupAlias> initialized successfully.

### 2.9. 定时任务初始化

DolphinDB 的 定时任务 会持久化保存到硬盘，单节点或控制节点保存在 <HOME_DIR>/<NodeAlias>/sysmgmt 目录下，数据节点或计算节点保存在 <HOME_DIR>/sysmgmt 目录下，相关文件说明：
| 文件 | 说明 |
| --- | --- |
| jobEditlog.meta | 定时任务的编辑日志文件 |
| jobCheckPoint.meta | 定时任务的 checkpoint 文件，当编辑日志过大时会做一次 checkpoint 保存最新的元数据的状态 |
| jobCheckPoint.tmp | 定时任务的 checkpoint 临时文件，做 checkpoint 时临时生成 |
- 开始，打印日志： Job scheduler start to initialize. ；
- 如果不存在编辑日志文件或为空，校验 checkpoint 临时文件，如果有效则重命名为 checkpoint 文件；
- 如果存在 checkpoint 文件，校验，如果失效则报错退出；
- 读取 checkpoint 文件恢复定时任务；
- 读取和回放编辑日志文件恢复定时任务；
- 裁剪无效的编辑日志；
- 结束，打印日志： Job scheduler initialization completed.

## 3. 启动问题定位思路

DolphinDB 节点启动失败时，最明显的现象是启动节点后，对应的 web 界面无法访问，如果是集群在 web 管理界面会看到节点状态为红色。启动问题主要可以分为三类：
- 启动异常关闭
- 启动异常卡住
首先需要先确认是哪类问题，在启动节点后，通过如下命令查看节点进程是否存在：

```
ps -ef | grep dolphindb // 如果修改了可执行文件名，需要修改 dolphindb 为相应可执行文件名
```

若进程不存在则是启动异常关闭；若进程存在，通过如下命令搜索日志确定节点是否启动完成：

```
grep "Job scheduler initialization completed." dolphindb.log
```

若有执行启动后的时间的日志输出则节点已正常启动，可以去 web 界面刷新确认节点状态是否已为绿色；否则是启动异常卡住或启动慢。通过如下命令搜索节点运行日志中的 ERROR：

```
grep "ERROR" dolphindb.log
```

若在执行启动后的时间一直重复刷某段 ERROR 日志，且节点进程一直在，则为启动异常卡在某个阶段；否则为正常启动，只是比较慢还没启动成功，需要继续等待观察启动结果。
具体问题需要分析 节点运行日志 ，节点运行日志存储位置由命令行参数 logFile 决定。另外集群环境下可以通过 logFile 配置项指定节点运行日志存储位置。
注意 ：若使用 startSingle.sh 启动单节点，节点运行日志默认存储在安装目录的 dolphindb.log 。若使用 clusterDemo 文件夹下的 startController.sh 和 startAgent.sh 来启动集群，节点运行日志默认存储在 clusterDemo/log ，文件名为 <NodeAlias>.log 。

### 3.1. 启动异常关闭

首先需要区分节点是启动成功后的运行过程中异常宕机，还是启动过程中异常关闭。需要参照 “节点整体启动流程” 一节查看是否有节点启动完成的日志 Job scheduler initialization completed. ，有则是节点启动成功后运行过程中异常宕机，需要根据 排查节点宕机的原因 来定位节点宕机问题；否则是启动过程中异常关闭问题。
需要查看节点最新运行日志中启动阶段的 ERROR 日志。注意要查看启动阶段的 ERROR 日志而不是启动失败后关机阶段的 ERROR 日志。DolphinDB 在关机时打印如下 ERROR 日志是预期的：

```
...
<ERROR> : The socket server ended.
...
<ERROR> : AsynchronousSubscriberImp::run Shut down the subscription daemon.
...
```

如果存在关机日志，需要继续往上搜索 ERROR 查看启动阶段的日志，结合前文的启动流程分析和后文的常见启动问题来分析失败原因。另外需要注意是否在启动过程中宕机，若宕机节点进程会直接被杀死，而不会走关机流程，此时需要查看 coredump 里的堆栈信息：

```
cd /path/to/dolphindb
gdb dolphindb /path/to/corefile
bt
```

堆栈信息需要发给 DolphinDB 技术支持来分析定位。

### 3.2. 启动异常卡住

需要查看节点最新运行日志中的 ERROR 日志，结合前文的启动流程分析、后文的常见启动问题来确定当前节点正在启动什么模块、执行什么动作失败。一般会重复打印某段 ERROR 日志以尝试启动，可以使用如下命令实时查看节点刷的日志：

```
tail -f dolphindb.log
```

另外可以使用 pstack 命令来查看启动时节点内部各个线程的堆栈信息，以确定线程具体执行的动作：

```
pstack dolphindb_pid > /tmp/pstack.log # 替换 dolphindb_pid 为 dolphindb 进程号
```

堆栈信息需要发给 DolphinDB 技术支持来分析定位。

### 3.3. 启动慢

需要查看节点最新运行日志，结合前文的启动流程分析、后文的常见启动问题来确定当前节点正在启动什么模块、执行什么动作。启动慢时，一般不会有 ERROR 日志。常见的启动慢原因是回滚事务或回放 redo log，详见 “启动慢” 一节。
另外可以使用 pstack 命令来查看启动时节点内部各个线程的堆栈信息，以确定线程具体执行的动作：
堆栈信息需要发给 DolphinDB 技术支持来分析定位。

## 4. 常见问题

列出 DolphinDB 常见的启动问题和解决方案。若问题现象不属于常见问题，请联系 DolphinDB 技术支持定位处理。

### 4.1. 启动异常关闭


#### 4.1.1. license 过期

DolphinDB 会在 license 过期前 15 天在 web 或 gui 提示 license 即将过期，而过期后如果节点还在线则能够继续使用 15 天，到第 15 天时会自动关机。License 过期后启动 DolphinDB 会失败，节点运行日志中会有如下 WARNING 和 ERROR 日志：

```
2023-10-13 09:52:30.007743 <WARNING> :The license has expired. Please renew the license and restart the server.
2023-10-13 09:52:30.163238 <ERROR> : The license has expired.
```

需要联系销售获取更新 license。

#### 4.1.2. 端口冲突

DolphinDB 启动时会绑定一个端口用来做网络传输，由配置文件的 localSite 配置项指定。若配置的端口被其他程序占用，或上一次关闭节点还没有完全关闭，则会导致节点启动时绑定端口失败而启动失败。查看节点运行日志有如下报错：

```
2023-10-26 09:01:31.349118 <ERROR> :Failed to bind the socket on port 8848 with error code 98
2023-10-26 09:01:31.349273 <ERROR> :Failed to bind the socket on port 8848. Shutting down the server. Please try again in a couple of
 minutes.
```

运行如下命令查看占用指定端口的程序：

```
netstat -nlp | grep 端口号
```

解决方案是停止占用端口的程序后再启动。若为上一次关闭的节点还没有完全关闭，需要等待节点关闭或使用 kill -9 强行停止节点再启动，强行停止节点可能会导致关机前未完成的写入的数据丢失。

#### 4.1.3. redo log 文件损坏

数据节点启动时会回放 redo log，如果上次运行时出现磁盘满、宕机或 bug，可能导致 redo log 文件损坏，可能导致节点启动时回放 redo log 失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-12-11 15:18:58.888865 <INFO> :applyTidRedoLog : 2853,c686664b-d020-429a-1746-287d670099e9,/hdd/hdd7/hanyang/server/clusterDemo/data/P1-datanode/storage/CHUNKS/multiValueTypeDb1/20231107/Key0/g
z,pt_2,32054400,1046013,0
2023-12-11 15:18:58.895064 <ERROR> :VectorUnmarshall::start Invalid data form 0 type 0
2023-12-11 15:18:58.895233 <ERROR> :The redo log for transaction [2853] comes across error: Failed to unmarshall data.. Invalid message format
2023-12-11 15:18:58.895476 <ERROR> :The ChunkNode failed to initialize with exception [Failed to unmarshall data.. Invalid message format].
2023-12-11 15:18:58.895555 <ERROR> :ChunkNode service comes up with the error message: Failed to unmarshall data.. Invalid message format
```

日志含义为回放 redo log 时发现 tid 为 2853 的 redo log 文件格式错误导致回放失败。此时需要通过如下步骤跳过 redo log 回放：
- mv 移走 redoLogDir 和 TSDBRedoLogDir 文件夹下的 head.log 文件，cp 备份报错对应的 2853.log 文件；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。
如果不是磁盘满导致，需要将 head.log 和报错对应的 2853.log 发给 DolphinDB 技术支持定位问题。

#### 4.1.4. 函数视图或定时任务包含不存在的方法

节点启动时会反序列化函数视图和定时任务文件，若反序列化的方法定义中包含不存在于内存的方法，将会导致相关函数视图和定时任务反序列化失败。常见情况如下：
- 使用了未配置自动加载的插件和模块的方法
- 更新插件或模块后相关方法名变更
例如，定义定时任务 myTest 调用 rabbitmq 插件的方法：

```
loadPlugin("plugins/rabbitmq/PluginRabbitMQ.txt")

def myTest() {
	HOST="192.168.0.53"
    PORT=5672
    USERNAME="guest"
    PASSWORD="guest"

    conn = rabbitmq::connection(HOST, PORT, USERNAME, PASSWORD);
}

scheduleJob("myTest", "myTest", myTest, 15:50m, startDate=today(), endDate=today()+3, frequency='D')
```

如果未配置 preloadModules=plugins::rabbitmq ，则节点启动时不会加载 rabbitmq 插件的函数定义到内存，节点启动反序列化定时任务会失败，运行日志会有如下报错：

```
2023-10-13 09:55:30.166268 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: Can't recognize function: rabbitmq::connection
2023-10-13 09:55:30.166338 <ERROR> :Failed to unmarshall the job [myTest]. Can't recognize function: rabbitmq::connection. Invalid message format
```

解决方案是添加报错方法对应的插件或模块到 preloadModules 配置项，即配置 preloadModules=plugins::rabbitmq ，然后再启动节点。
若为更新插件或模块后相关方法名变更，需要回退插件或模块再启动，删除对应的视图或定时任务后，再升级插件或模块。

#### 4.1.5. 函数视图或定时任务包含不存在的共享表

注意 ：该问题已在 1.30.23.1/2.00.11.1 或以上版本修复。
节点启动时会反序列化函数视图和定时任务文件，若反序列化的方法定义中包含不存在于内存的共享表，将会导致相关函数视图和定时任务反序列化失败。
问题 1 ： 定时任务反序列化失败
例如，定时任务 myTest 的定义如下：

```
share table(1 2 3 as id, 1 2 3 as val) as t

def myTest() {
	update t set val = val + 1
}

scheduleJob("myTest", "myTest", myTest, minute(now())+5, today(), today(), 'D')
```

其中第 4 行的 update 语句用于更新共享表 t。如果在启动脚本 startup.dos 中未创建共享表 t，则节点启动时反序列化定时任务会失败，运行日志会有如下 WARNING 和 ERROR 日志：

```
2023-10-23 09:38:27.746184 <WARNING> :Failed to recognize shared variable t
2023-10-23 09:38:27.746343 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: Failed to deserialize update statement
2023-10-23 09:38:27.746404 <ERROR> :Failed to deserialize update statement. Invalid message format
```

原因：定时任务的定义中使用了未创建的表。
- 在报错的定时任务 myTest 中检查是否使用了未创建的表，例如前述的共享表 t。
- 在定时任务所在节点的启动脚本 startup.dos 中添加相应的建表语句。
- 启动节点。
问题 2：函数视图反序列化失败
如果将前述问题 1 中的定时任务 myTest 添加到函数视图，例如： addFunctionView(myTest) ，也会导致节点启动时反序列化函数视图失败，运行日志中会出现相同的报错。
原因：函数视图反序列化先于节点启动，因此在控制节点的 startup.dos 中定义共享表 t 也不会有效。
- 对于普通集群： 移除 /sysmgmt 下的 aclEditlog.meta 、 aclCheckPoint.meta 、 aclCheckPoint.tmp 。 重新启动节点。 启动后重新添加所有权限和函数视图定义。
- 对于高可用集群，如果未重启，或存在半数以上控制节点存活： 删除相关函数视图： dropFunctionView("myTest") 生成权限与函数视图的 checkpoint 文件以避免启动时回放 RAFT 日志执行之前的函数视图定义。 rpc(getControllerAlias(), aclCheckPoint, true) // 参数 force = true 表示强制生成 checkpoint。
- 对于高可用集群，如果已重启： 移除所有控制节点的 <HOME_DIR>/<NodeAlias>/raft 目录下的 raftHardstate[group] 、 raftWAL[group] 、 raftSnapshot[group] 、 raftWAL[group].old 、 raftSnapshot[group].tmp 。 注意：这会导致集群元数据全部失效。 重新启动节点。
必然存在 aclEditlog.meta ，可能存在 aclCheckPoint.meta ， aclCheckPoint.tmp 。
必然存在 raftHardstate[group] ， raftWAL[group] ，可能存在 raftSnapshot[group] ， raftWAL[group].old ， raftSnapshot[group].tmp 。
后续版本会优化添加函数视图的功能以避免需要删除元数据来解决该问题。

#### 4.1.6. 函数视图方法名与预加载的模块或插件的方法名冲突

节点启动时会反序列化函数视图文件，若反序列化的函数视图方法名与已通过 preloadModule 加载的方法名冲突，将会导致函数视图反序列化失败。例如，直接添加 ops 模块的 cancelJobEx 方法到函数视图：

```
use ops
addFunctionView(ops::cancelJobEx)
```

若同时配置 preloadModules=ops ，启动时运行日志会有如下报错：

```
2023-10-20 08:46:15.733365 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: Not allowed to overwrite existing functions/procedures [ops::cancelJobEx] by system users.
2023-10-20 08:46:15.733422 <ERROR> :Not allowed to overwrite existing functions/procedures [ops::cancelJobEx] by system users.. Invalid message format
```

解决方案是去掉 preloadModules=ops 配置项，然后再启动节点。不建议先定义模块再将模块内的方法添加到函数视图，而应该直接定义方法再添加到函数视图。

#### 4.1.7. 定时任务文件损坏

DolphinDB 序列化定时任务到文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致定时任务文件损坏，可能导致节点启动时反序列化失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:57:30.456789 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: Failed to deserialize update statement
2023-10-13 09:57:30.456789 <ERROR> :Failed to unmarshall the job [myTest]. Failed to deserialize update statement. Invalid message format
```

此时如果希望尽快启动节点，可以移走 jobEditlog.meta 、 jobCheckPoint.meta 、 jobCheckPoint.tmp 文件以跳过启动时的定时任务反序列化，再启动节点。
注意 ：必然存在 jobEditlog.meta ，可能存在 jobCheckPoint.meta 、 jobCheckPoint.tmp 。
请打包定时任务文件、日志里报错的定时任务脚本和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，需要重新提交所有定时任务。

#### 4.1.8. 权限与函数视图文件损坏

DolphinDB 序列化权限操作和函数视图定义到文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致权限和函数视图文件损坏，可能导致节点启动时反序列化失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:59:35.786438 <ERROR> :CodeUnmarshall::start readObjectAndDependency exception: Failed to deserialize sql query object
2023-10-13 09:59:35.786438 <ERROR> :Failed to unmarshall the job [myTest1]. Failed to deserialize sql query object. Invalid message format
```

此时如果希望尽快启动节点，可以参考 “函数视图或定时任务包含不存在的共享表” 中的 问题 2：函数视图反序列化失败 小节的解决方法移走权限与函数视图文件以跳过反序列化。
请打包权限和函数视图文件、日志里报错的函数视图脚本和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，需要重新添加所有权限和函数视图定义。

#### 4.1.9. RAFT 文件损坏

注意 ：下面介绍的操作需要确保已经有另一个节点成为 RAFT 集群 leader 才可以进行。DolphinDB 写 RAFT 元数据和日志时，如果上次运行时出现磁盘满、宕机或 bug，可能导致 RAFT 元数据或日志文件损坏，可能导致控制节点启动时恢复 RAFT 数据失败抛出异常而启动失败。例如查看节点运行日志有如下报错：

```
2023-10-13 09:59:35.786438 <WARNING> :[Raft] incomplete hardstate file [/data/server/data/controllerl/raft/raftHardstatel]
2023-10-13 09:59:35.786438 <INFO> :[Raft] Group DFSMaster RaftWAL::reconstruct: read new file with 83213 entries 
2023-10-13 09:59:35.786438 <ERROR> :[Raft] Group DFSMaster RawNode::init: failed to initialize with exception [basic_string::_S_create].
2023-10-13 09:59:35.786438 <ERROR> :Failed to start DFSMaster with the error message: basic_string::_S_create
```

此时如果希望尽快启动节点，可以移走问题节点的 dfsMetaDir 配置项 文件夹和 <HOME_DIR>/<NodeAlias>/raft 文件夹以跳过启动时的 DFS 和 RAFT 元数据初始化，再启动节点。
请打包 dfsMetaDir 配置项 文件夹、 <HOME_DIR>/<NodeAlias>/raft 文件夹和节点运行日志，联系 DolphinDB 技术支持排查问题。
节点启动后，会自动同步 leader 节点的元数据。

### 4.2. 启动异常卡住


#### 4.2.1. 集群间网络不通

在多台机器上部署 DolphinDB 时，需要确保集群间各个节点的 IP:端口号互通，否则会导致节点启动失败。常见原因和现象是调整了机器网络配置后重启高可用集群，进入控制节点 web 管理界面白屏，查看控制节点日志中有如下报错：

```
2023-11-01 16:00:34.992500 <INFO> :New connection from ip = 192.168.0.44 port = 35416
2023-11-01 16:00:35.459220 <INFO> :DomainSiteSocketPool::closeConnection: Close connection to ctl1 #44 with error: epoll-thread: Read/Write failed. Connection timed out. siteIndex 0, withinSiteIndex 44
```

或在 web 启动数据节点时报错 IO error type 1 （含义为 Socket is disconnected/closed or file is closed ，即网络连接断开）。
需要联系运维调通网络。

#### 4.2.2. RSA 密钥校验文件损坏

DolphinDB 生成 RSA 密钥校验文件时，如果上次运行时出现磁盘满、宕机或 bug，可能导致 RSA 密钥校验文件损坏，使节点无法正常通过 RSA 密钥来通信，进而导致启动流程卡住。查看节点运行日志有如下报错：

```
2023-10-25 11:55:04.987161 <ERROR> :Failed to decrypt the message by RSA public key.
```

解决方案是删除所有控制节点的 <HOME_DIR>/<NodeAlias>/keys 目录并重启集群，以触发 RSA 密钥校验文件重新生成。 注意删除 keys 文件夹后需要重新提交集群的所有定时任务。

### 4.3. 启动慢


#### 4.3.1. 正在回滚事务，如何查看进度，如何删除事务 log 以跳过

DolphinDB 启动时，如果日志里有 Will process pending transactions. 而没有 ChunkMgmt initialization completed. 说明正在回滚事务。如果节点宕机时的写入事务涉及的分区过多或数据量过大，可能导致事务回滚时间较长。
- 查看 <chunkMetaDir>/LOG 目录下以 tid 命名的文件夹数目是否减少，这里只能根据文件夹减少速度估计回滚速度；
- 若关机前有删除事务，查看 <volumes>/LOG 目录下以 tid 命名的文件夹数目是否减少，文件夹数目为 0 时回滚完毕。
注意 ：统计文件数目可以使用 Linux 命令： ll -hrt <chunkMetaDir>/LOG | wc -l
强烈建议等待事务回滚完成 ，跳过事务回滚会使原本应该回滚的事务不回滚，导致事务相关的数据或元数据错误。若客户希望尽快启动节点且 不需要保证节点宕机前正在写入的数据完整性 ，可以通过如下步骤跳过事务回滚：
- 使用 kill -15 pid 安全关闭节点，若正在启动时无法关闭则使用 kill -9 强制关闭节点，因为正在启动所以不会有新的写入；
- mv 移走 <chunkMetaDir>/LOG 和 <volumes>/LOG 文件夹；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。

#### 4.3.2. 正在回放 redo log，如何查看进度，如何删除 redo log 以跳过

DolphinDB 启动时，如果日志里有 Start recovering from redo log. This may take a few minutes. 而没有 Completed CacheEngine GC and RedoLog GC after applying all redo logs and engine is <engineType> ，说明正在回放 redo log。期间会刷带“RedoLog”的日志例如：

```
"applyTidRedoLog : 20716,f7dbaef9-05bc-10b6-f042-a14bc0e9c897,/home/DolphinDB/server/clusterDemo/data/node2/storage/CHUNKS/snapshotDB/20220803/Key17/5o7,shfe_5,166259,107,0"
```

注意：如果是 DolphinDB 2.0 版本，会有两次 redo log 回放，对应 OLAP 和 TSDB 存储引擎，故对应的日志也有两份；如果是 DolphinDB 1.0 版本，只会有一次 OLAP 存储引擎的 redo log 回放。
- 统计 redoLogDir 和 TSDBRedoLogDir 下的 tid.log 文件数目，为 0 时回放完毕；
- 统计 redoLogDir 和 TSDBRedoLogDir 文件夹大小，除以硬盘的读速率，可估计最快的回放完成时间。
注意 ：统计文件夹大小可以使用 Linux 命令： du -sh <redoLogDir>
强烈建议等待 redo log 回放完成 ，跳过 redo log 回放会使原本应该回放的事务不回放，会导致事务相关的数据或元数据错误。若客户希望尽快启动节点且 不需要保证节点宕机前正在写入的数据完整性 ，可以通过如下步骤跳过 redo log 回放：
- 使用 kill -15 pid 安全关闭节点，若正在启动时无法关闭则使用 kill -9 强制关闭节点，因为正在启动所以不会有新的写入；
- mv 移走 redoLogDir 和 TSDBRedoLogDir 文件夹下的 head.log 文件；
- 启动节点，观察是否正常启动，启动后检查重启前正在写入的数据完整性，是否要补数据等。

### 4.4. 其他问题


#### 4.4.1. 启动脚本运行慢或失败

启动脚本 startup.dos 运行慢会导致启动流程走不到初始化定时任务一步，但由于节点在 redo log 回放完成后即在 web 集群管理界面转变状态为绿色，已经可以访问，故会导致定时任务相关功能在 startup.dos 执行完前无法使用。
启动脚本 startup.dos 或 postStart.dos 运行失败会在节点运行日志里打印错误日志，然后跳过启动脚本报错行后的执行，不会导致节点启动失败，也不会回滚执行失败的动作，需要客户自行考虑启动脚本运行失败的情况。注意在 集群模式下，执行启动脚本时无法保证分布式数据库已初始化完毕 ，在启动脚本访问分布式库表可能会报错。
故不建议在启动脚本执行太慢的操作或涉及分布式库表的操作，而只做一些比较简单的操作如建共享表、加载插件等。启动脚本详细介绍见 启动脚本 。
可以参考如下脚本在 startup.dos 或 postStart.dos 等待分布式模块准备完毕：

```
def isClusterOk() {
    do {
        try {
            meta = rpc(getControllerAlias(), getClusterChunksStatus)
            configReplicaCount = 2 // 需要修改为 dfsReplicationFactor 配置项值

            cnt1 = exec count(*) from meta where state != "COMPLETE"
            cnt2 = exec count(*) from meta where replicaCount != configReplicaCount
  
            if (cnt1 == 0 and cnt2 == 0) {
                break
            } else {
                writeLog("startup isClusterOk: state != 'COMPLETE' cnt: " + string(cnt1) + ", " + "replicaCount != " + string(configReplicaCount) + " cnt: " + string(cnt2))
            }
        } catch (err) {
            writeLog("startup isClusterOk: " + err)
        }

        sleep(3*1000)
    } while(1)

    return true
}

res = isClusterOk()
writeLog("startup isClusterOk: ", string(res))
```


## 5. 新旧版本日志对照表

| 2.00.11 及以后 | 新版备注 | 2.00.11 以前 | 旧版备注 |
| --- | --- | --- | --- |
| Initializing AclManager with Raft <raft mode> | RAFT mode 为 enabled 或 disabled | AclManager start to initialize with raftmode is <raft mode> | RAFT mode 为 0 或 1 |
| Initialization of AclManager is completed with Raft <raft mode> | RAFT mode 为 enabled 或 disabled | AclManager::init successfully with raftmode is <raft mode> | RAFT mode 为 0 或 1 |
| Controller initialization completed. |  | DFS master is ready. |  |
| ChunkMgmt initialization completed. |  | ChunkMgmt initiated successfully. |  |
| Restore TSDB meta successfuly. |  | Restore iot meta successfuly. |  |
| No available redo log files were found. Redo log replay was skipped. |  | No need to recover from redo log. |  |
| Job scheduler start to initialize. |  | Job schedule start to initialize. |  |
| Job scheduler initialization completed. |  | Job schedule initialized successfully. |  |

## 6. 总结

本文介绍了 DolphinDB 整体的启动流程和重要模块的启动流程，并分析了启动时常见问题的现象、原因和解决方案。了解 DolphinDB 的启动流程与常见问题，有助于维护 DolphinDB 的稳定运行。遇到启动问题时，DolphinDB 运维人员可以参照本文处理一些常见问题，使 DolphinDB 尽快正常运行。
