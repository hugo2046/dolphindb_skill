# DolphinDB 参考：数据库建库与最佳实践



---

> 来源: `tsdb_explained.html`


# TSDB 存储引擎详解


## 1. 阅读指南

- 如果需要进行引擎间的迁移或希望了解 OLAP 引擎和 TSDB 引擎的区别，推荐阅读第 1 章。
- 如果需要了解底层的架构和工作原理推荐阅读第 2 章。
- 如果希望直接上手搭建 TSDB 数据库，推荐从第 3 章开始阅读。
- 第 4 章给出了 TSDB 查询优化案例。
- 更多 TSDB 的实践案例，请参考第 5 章的场景指南。
- 第 8 章附录列出了 TSDB 相关的配置项、运维函数以及常见问题一览表。
时序数据是一种按照时间顺序排列的数据集合，例如传感器数据、金融市场数据、网络日志等。由于时序数据具有时间上的连续性和大量的时间戳，传统的关系型数据库或通用数据库在处理大规模时序数据时可能效率较低。因此，针对时序数据进行优化的专用数据库——TSDB 变得越来越重要。
为了更好地支持时序数据的分析和存储，DolphinDB 于 2.0 版本推出了”时序数据存储引擎“（Time Series Database Engine，以下简称 TSDB）。
2.0 版本以前 DolphinDB 仅支持分析型数据库存储引擎（Online Analytical Processing，以下简称 OLAP）。OLAP 引擎采用列式存储，压缩率高、查询和计算列的性能也十分优越。随着 DolphinDB 2.0 版本 TSDB 引擎的推出，用户在多模引擎的选择和使用上可能会遇到困难。本文的目标是通过原理解析、使用指导、案例分析的介绍，帮助用户更好地理解 TSDB 引擎，并能够在实际场景中配置更优的存储策略。

## 2. TSDB vs OLAP

关于 TSDB 和 OLAP 适用场景的描述性说明，请参照教程： TSDB 存储引擎简介
为了便于用户由浅入深地进行学习，本文开头提供了一个 TSDB 和 OLAP 引擎的区别表，以帮助用户对两者的区别有一个概括性的认识。除了列出区别点外，表格中标注了一些使用上的注意点；表格中的术语请参照表尾的术语对照表。
| 指标 | TSDB | OLAP |
| --- | --- | --- |
| 存储结构 | 行列混存（Partition Attributes Across，简称 PAX）写入时，每 32 M 数据存储为一个 Level File 文件（压缩后可能小于 10 M），文件内部进一步划分数据为多 | 列式存储每个分区下每列都存储为一个文件。注：若列文件过多时，在 IO 时会造成性能瓶颈，因此 OLAP 不适合存储列数过多的宽表。建议 OLAP 字段不超过 100。 |
| 数据形式和类型（区别点） | Array Vector、BLOB 类型 | 不支持左述两类数据 |
| 数据压缩 | 以 block 为压缩单元 | 以列文件为压缩单元。注：一般情况下，OLAP 压缩率会高于 TSDB。 |
| 数据加载（最小单元） | 每个分区的 Level File 文件中的 block | 每个分区的子表的列文件 |
| 写入方式 | 顺序写入注：TSDB 引擎每次刷盘都会直接存储为新的 Level File 文件，无需像 OLAP 引擎一样追加数据到对应列文件，磁盘几乎没有寻道压力。 | 顺序写入注：OLAP 引擎列文件过多时，写入时会增加磁盘（HDD）的寻道时间。 |
| 查找方式 | 1. 分区剪枝 查 Cache Engine 缓存（分为两个部分）： a. unsorted buffer：遍历 b. sorted buffer：二分查找 3. 查找磁盘：根据索引快速定位到 blo | 1. 分区剪枝查2. Cache Engine 缓存3. 查找磁盘：读取相关列文件后遍历查找注：OLAP 会维护用户态缓存。第一次查询后，会将上次查询的列文件缓存在内存中。该部分缓存会保存在内存中，直 |
| 是否支持索引 | 支持，以 sortKey 字段的组合值作为索引。 | 不支持 |
| 优化查询方法 | 命中分区剪枝；通过索引加速查询。 | 命中分区剪枝；读取整列数据较为高效，因此查询整个分区或整列数据时具有查询优势。 |
| 是否支持去重 | 支持，基于 sortColumns 去重 | 不支持 |
| 读写数据是否有序 | 每批刷盘的数据会按照 sortColumns 排序。注：查询时，只保证单个 Level File 内的数据是有序的，Level File 之间的数据顺序不能保证。 | 按写入顺序写入磁盘。注：单个分区内，查询结果的顺序和写入顺序一致；跨分区查询时，单分区数据和写入顺序一致，系统会按固定的分区顺序合并各分区的查询结果返回给用户端，因此重复查询仍维持结果的一致性。 |
| 增删改 | 增：每次写入的数据可能存储在不同的 Level File。Level File 是不可修改和追加的。删：keepDuplicates=LAST 时，若 softDelete=true，则会先打上删除标 | 增：直接追加到列尾删：读取相关分区下的所有列文件到内存，删除后再写回改：读列文件到内存，更新完再写回 |
| DDL & DML 操作（仅列出两者存在区别的点） | 不支持 rename!（修改字段名），replaceColumn!（修改字段的值或类型），dropColumns! （删除列字段）等操作 | 支持左述操作 |
| 资源开销（仅列出两者存在区别的点） | 内存：排序开销，索引缓存；磁盘：合并 Level File 的 IO 开销。 | 不存在左述开销 |

### 2.1. 术语对照

- block：参见 确定索引键值 部分的说明。
- Level File：参见 Level File 部分的说明。
- unsorted buffer / sorted buffer：参见 写入流程 部分的说明。
- sortColumns： sortColumns 的介绍。
- sortKey：参见 确定索引键值 部分的说明。

## 3. TSDB 存储引擎架构

本章将存储架构、读写流程，参数功能，以及底层文件组织层面详细解析 TSDB 引擎的工作原理，以帮助用户从架构层面理解 TSDB 引擎的机制，从而更好的调试和配置 TSDB 的功能参数，实现更优的解决方案。

### 3.1. LSM-Tree

TSDB 引擎是 DolphinDB 基于 Log Structured Merge Tree (以下简称 LSM-Tree) 自研的存储引擎。
LSM-Tree 是一种分层、有序、面向磁盘的数据结构，主要用于处理写入密集型的工作负载，如日志系统、实时分析系统和数据库等。其实现的核心思想是利用磁盘批量顺序写入大幅提高写入性能。其工作流程图如下所示：
- LSM-Tree 的数据查询流程： 首先在内存表进行检索，由于内存表是有序的数据结构，因此可以利用算法快速检索并返回结果。 如果在内存表中找不到数据，则会按层级遍历磁盘上的 SSTable 进行检索，由于 SSTable 文件数据是按键值排序的，因此可以在单个 SSTable 内快速进行范围查询。为了提高 SSTable 遍历的效率，可以给 SSTable 增加 Bloom Filter，以减少不必要的磁盘扫描。 查询时，SSTable 可能会包含多个不同历史版本的数据，此时需要根据查询条件，找到符合版本的数据（例如最新一条数据）。 注： 最坏的情况下，需要将所有层级的 SSTable 文件都遍历一遍，如果 SSTable 文件过多，查询是非常低效的，因此可以理解为什么 LSM-Tree 要引入合并机制，减小 SSTable 的文件数。
相较于传统的 B+ 树而言，LSM-Tree 的优势在于高吞吐写入与较低的随机写延迟，这非常契合 DolphinDB 海量数据高性能写入的设计诉求。虽然没有 B+ 树那样高效的随机读和范围查询性能，LSM-Tree 依靠键值排序使得单个 SSTable 文件的范围查询也十分高效。
参考资料： Principle of LSM Tree
DolphinDB 的 TSDB 引擎是基于 LSM-Tree 架构进行开发的，因此大部分内部的流程机制也和 LSM-Tree 类似，但在某些步骤进行了优化：
- 写入： 与 LSM-Tree 直接写入一个有序的数据结构不同，DolphinDB TSDB 引擎在写入数据到内存时，会先按写入顺序存储在一个写缓冲区域（unsorted write Buffer），当数据量累积到一定程度，再进行排序转化为一个 sorted buffer。
- 查询： DolphinDB TSDB 引擎会预先遍历 Level File（对应 LSM-Tree 的 SSTable 文件），读取查询涉及的分区下所有 Level File 尾部的索引信息到内存的索引区域（一次性读，常驻内存）。后续查询时，系统会先查询内存中的索引，若命中，则可以快速定位到对应 Level File 的数据块，无需再遍历磁盘上的文件。

### 3.2. TSDB 引擎 CRUD 流程

本节将具体介绍 DolphinDB TSDB 引擎的增删改查（CRUD）流程。

#### 3.2.1. 写入流程

TSDB 引擎写入整体上和 OLAP 一致，都是通过两阶段协议进行提交。写入时，先写 Redo Log（每个写事务都会产生一个 Redo Log），并写入 Cache Engine 缓存，最后 由后台线程异步批量写入磁盘。需要注意的是，TSDB 引擎和 OLAP 引擎各自单独维护 Redo 以及 Cache Engine，用户需要通过不同的配置项去设置两个引擎的 Cache Engine 大小，分别为 OLAPCacheEngineSize 和 TSDBCacheEngineSize。 功能配置
- 写 Redo： 先将数据写入 TSDB Redo Log。
- 写 Cache Engine： 写 Redo Log 的同时，将数据写入 TSDB Cache Engine 的 CacheTable，并在 CacheTable 内部完成数据的排序过程。 CacheTable 分为两个部分：首先是 write buffer，数据刚写入时会追加到 write buffer 的尾部，该 buffer 的数据是未排序的。当 write buffer 超过 TSDBCacheTableBufferThreshold 的配置值（默认 16384 行），则按照 sortColumns 指定的列排序，转成一个 sorted buffer (该内存是 read only 的)，同时清空 write buffer。

#### 3.2.2. 查询流程

相较于 OLAP 引擎，TSDB 引擎增加了索引的机制，因此更适用于点查场景，因此在配置 TSDB 引擎的 sortKey 字段（参照 确定索引键值 的说明）时，可以优先考虑从频繁查询的字段中进行选取（如需了解更多 sortColumns 的设置原则，请参见 创建数据库 的合理设置排序字段部分）。
此外和 OLAP 引擎在用户态缓存数据不同（第一次执行查询语句会加载分区数据，这部分数据会常驻内存，后续的查询可以提速，但增大了内存开销），TSDB 引擎只有操作系统的缓存，没有用户态的缓存。
- 分区剪枝： 根据查询语句进行分区剪枝，缩窄查询范围。
- 加载索引： 遍历涉及到的分区下的所有 Level File，将其尾部的索引信息加载到内存中（索引信息采用惰性缓存策略，即不会在节点启动时被立即加载进内存，而是在第一次查询命中该分区时才被加载进内存）。查询命中的分区的索引信息一旦被加载到内存后，会一直缓存在内存中（除非因内存不够被置换），后续查询若涉及该分区则不会重复此步骤，而是直接从内存中读取索引信息。 注： 内存中存放索引的区域大小由配置项 TSDBLevelFileIndexCacheSize 决定，用户可以通过函数 getLevelFileIndexCacheStatus 在线查询内存中的索引占用。若加载的索引大小超过了该值，内部会通过一些缓存淘汰算法进行置换，用户可配置 TSDBLevelFileIndexCacheInvalidPercent 来调整缓存淘汰算法的阈值。
- 查找内存中的数据： 先搜索 TSDB Cache Engine 中的数据。若数据在 write buffer 中，则采用顺序扫描的方式查找；若在 sorted buffer 中，则利用其有序性，采用二分查找。
- 查找磁盘上的数据： 根据索引查找磁盘 Level File 中各查询字段的数据块，解压到内存。若查询的过滤条件包含 sortKey 字段，即可根据索引加速查询。
- 返回查询结果： 合并上述两步的结果并返回。
内存索引包含两个部分：sortKey 维护的 block 的偏移量信息（对应 Level File indexes 部分），zonemap 信息（对应 Level File zonemap 部分）。
- 查询 indexes 定位 sortKey： indexes 中记录了每个 Level File 中每个 sortKey 值下每个字段的 block 的地址偏移量。若查询条件包含 sortKey 字段，则可以根据索引剪枝，缩窄查询范围。命中 sortKey 索引后，即可获取到对应 sortKey 下所有 block 的地址偏移量信息。 例如，上图 sortColumns=`deviceId`location`time，查询条件 deviceId = 0，则可以快速定位到所有 Level File 中 deviceId= 0 的 sortKey 及其对应所有字段的 block 数据。
- 根据 zonemap 定位 block： 由 1 定位到 sortKey 后，系统会查询对应 sortKey 在 zonemap 里的最值信息（min，max）。如果查询条件提供了 sortKey 以外的字段范围，则可以进一步过滤掉一些不在查询范围内的 block 数据。 例如，上图 sortColumns=`deviceId`location`time，查询条件 time between 13:30 and 15:00，则可以根据 time 列的 zonemap 信息，快速定位到查询数据所在的 block 为“block2”。系统根据该信息再去 indexes 找到 block2 的地址偏移量信息，然后根据索引到的所有 Level File 的 block2 的地址，把 block 数据从磁盘中读取出来，再根据去重策略过滤结果后返回给用户端。

#### 3.2.3. 更新流程

TSDB 引擎的更新效率取决于 keepDuplicates 参数配置的去重机制。
- keepDuplicates=ALL/FIRST 时的更新流程： 分区剪枝： 根据查询语句进行分区剪枝，缩窄查询范围。 查到内存更新： 取出对应分区所有数据到内存后，更新数据。 写回更新后的分区数据到新目录： 将更新后的数据重新写入数据库，系统会使用一个新的版本目录（默认是“物理表名_cid”，见下图，其中 tick_2 是物理表名）来保存更新后的分区数据，旧版本的分区数据文件将被定时回收（默认 30 min）。
- keepDuplicates=LAST 时的更新流程： 分区剪枝： 根据查询语句进行分区剪枝，缩窄查询范围。 查到内存更新： 根据查询条件，查询出需要更新的数据。（查询步骤见 查询流程 ） 直接以写数据的方式追加写入： 更新后，直接追加写入数据库。
更新后的数据和旧的数据可能同时存储在磁盘上，但查询时，由于会按照 LAST 机制进行去重，因此可以保证不会查询出旧的数据。旧数据会在 Level File 合并操作时进行删除。
若需要深入了解更新方法、更新流程以及不同策略的更新性能，请参考教程： 分布式表数据更新原理和性能介绍

#### 3.2.4. 删除流程

TSDB 引擎的删除流程和更新流程一致，即：
- 指定 keepDuplicates=ALL/FIRST 时，按分区全量取数，根据条件删除后写入一个新版本目录；
- 指定 keepDuplicates=LAST 时，若同时指定 softDelete=true，则取出待删除的数据打上删除标记（软删除），再以追加的方式写回数据库；若指定 softDelete=false，则同 keepDuplicates=ALL/FIRST 时的删除流程。
- 分区剪枝： 根据查询语句进行分区剪枝，缩窄查询范围。
- 查到内存删除： 取出对应分区所有数据到内存后，根据条件删除数据。
- 写回删除后的分区数据到新目录： 将删除后的数据重新写入数据库，系统会使用一个新的 CHUNK 目录（默认是“物理表名_cid”）来保存写入的数据，旧的文件将被定时回收（默认 30 min）。
- 在配置 keepDuplicates=ALL / FIRST 场景下，更新操作（keepDuplicates=ALL / FIRST）和删除操作是按分区全量修改，因此需要确保每次更新删除操作涉及分区总大小不会超过系统的可用内存大小，否则会造成内存溢出。
- 在配置 keepDuplicates=LAST 场景下，更新操作和删除操作是按照直接追加的方式增量修改，效率更高，若业务场景需要高频更新和删除，可以配置此策略。

### 3.3. 排序列 sortColumns

sortColumns 参数在 TSDB 引擎中起到三个作用：确定索引键值、数据排序、数据去重。

#### 3.3.1. 确定索引键值

根据查询流程可以知道，TSDB 的索引机制可以提升查询性能，那么索引又是如何确定的？
写入流程提到，在 Cache Engine 中，数据会根据 sortColumns 指定的列进行排序，TSDB 引擎的索引列就是基于排序列建立的。
sortColumns 由两部分组成：sortKey（可以是多列，其组合值作为数据的索引键），最后一列。
假设 sortColumns 指定了 n 个字段，则系统取前 n-1 个字段的组合值，作为索引键 sortKey，每个 sortKey 值对应的数据按列存储在一起（如下图，假设 sortColumns 为 deviceId 和 timestamp）。
注： 若 sortColumns 只有一列，则该列将作为 sortKey。
每个 sortKey 内部的数据仍然是按列存储的，其中每个列的数据又按记录数划分为多个 block（按固定行数划分，参见 Level File 层级示意图）。block 是内部最小的查询单元，也是数据压缩的单元，其内部数据按照时间列的顺序排序。
查询时，若查询条件包含 sortColumns 指定的字段，系统会先定位到对应的 sortKey 的数据所在的位置，然后根据 block 内部 数据的有序性 以及 列之间 block 的对齐性 ，通过时间列快速定位到对应的 block，将相关列的 block 读取到内存中。
- block 数据的有序性 ：block 内部数据是按照时间列排好序的，此外每个 sortKey 的元数据都记录了对应每个 block 的第一条数据，因此根据和每个 block 的第一条数据比较，可以快速过滤掉一些不必要的 block 的查询，从而提升查询性能。
- block 的对齐性 ：由于 block 的数据量都是固定的，因此根据时间列快速定位到时间列所在的 block，就能根据该 block 的 offset 快速定位到其他列的 block。
sortKey 需要合理设置，每个分区的 sortKey 值不宜过多。因为同等数据量下，sortKey 值越多，意味着每个 sortKey 对应的数据量越少，不仅在命中索引的过程会增大查询开销，而且会增大读取每个 sortKey 元数据的开销，反而会降低查询效率。

#### 3.3.2. 数据排序

写入流程提到，在 TSDB Cache Engine 中，每批刷盘的数据会根据 sortColumns 进行排序，然后再写入磁盘，可以推断：
- 每个写入事务的数据一定是有序的。
- 单个 Level File 文件内的数据一定是有序的。
- Level File 之间数据的顺序无法保证。
- 每个分区的数据的有序性无法保证。
结合 确定索引键值 节的说明和上述推断，可以了解到 sortColumns 的数据排序功能，并不保证数据整体的有序性，而只是保证 Level File 内数据按 sortKey 有序排列，以及每个 sortKey 中 block 内数据的有序性， 这有助于：
- 查询条件包含 sortKey 字段的范围查询或第一个 sortKey 字段的等值查询时，加速内存查找索引的效率。
- 命中 sortKey 索引时，可以根据 sortKey 的元数据信息，加速 block 的定位。

#### 3.3.3. 数据去重

TSDB 的去重机制主要用于 同一个时间点产生多条数据，需要去重的场景 。
去重是基于 sortColumns 进行的，发生在写入时数据排序阶段以及 Level File 的合并阶段。其对应的配置参数为 keepDuplicates（在建表是设置），有三个可选项：ALL（保留所有数据，为默认值），LAST（仅保留最新数据），FIRST（仅保留第一条数据）。
不同去重机制可能会对更新操作产生影响（具体参见 数据更新流程 ）：若 keepDuplicates=ALL/FIRST，那么每次更新，都需要将分区数据读取到内存更新后再写回磁盘；若 keepDuplicates=LAST，则更新数据将以追加的方式写入，真正的更新操作将会在 Level File 合并阶段进行。
- 去重策略不能保证磁盘上存储数据不存在冗余，只能保证查询时不会返回冗余结果。查询数据时，会将对应 sortKey 所在各个 Level File 中的数据块读出，然后在内存中进行去重，再返回查询结果。
- DolphinDB 不支持约束。现实场景下，很多用户会利用 sortColumns 的去重机制，将数据中的主键或唯一约束设置为 sortColumns，造成一个 sortKey 键值对应的数据量很少，从而导致 TSDB 数据库 数据膨胀 （详见案例 sortkey 数量对查询性能的影响 ）。

### 3.4. Level File 文件

TSDB 引擎存储的数据文件在 DolphinDB 中被称为 Level File（对应 LSM-Tree 的 SSTable 文件），即 LSMTree 架构各 Level 层级的文件。Level File 主要保存了数据和索引信息，其中索引信息保存在 Level File 的尾部。为了更好地理解 Level File 在整个系统中发挥的作用，本节将对 Level File 进行一个简单介绍。

#### 3.4.1. Level File 的组成

Level File 文件存储的内容可以划分为：
- header：记录了一些保留字段、表结构以及事务相关的信息。
- sorted col data：按 sortKey 顺序排列，每个 sortKey 依次存储了每列的 block 数据块。
- zonemap：存储了数据的预聚合信息（每列每个 sortKey 对应数据的 min，max，sum，notnullcount）。
- indexes：sorted col data 的索引信息，记录了 sortKey 个数，每个 sortKey 的记录数、每列 block 数据块在文件中的偏移量信息，checksum 等。
- footer：存储 zonemap 的起始位置，用来定位预聚合和索引区域。
Level File 的 zonemap 和 indexes 部分在查询时会加载到内存中，用于索引。

#### 3.4.2. Level File 层级组织

Level File 层级间的组织形式如下图所示：
磁盘的 Level File 共分为 4 个层级，即 Level 0, 1, 2, 3 层。层级越高，Level File 文件大小越大，每个 Level File 中数据划分的 block 大小也越大。

#### 3.4.3. Level File 合并

多次写入后，多个相同的 sortKey 的数据可能分散在不同的 Level File 里。查询时，系统会将所有满足条件的数据读到内存，然后再根据去重机制进行去重。为此，TSDB 设计了文件合并的机制，通过合并操作（compaction）大大减少文件数，有效的清除无效数据，减少碎片化查找，提高磁盘空间利用率（压缩率提升）以及提升查询性能。
文件合并机制： 若一层的 Level File 超过 10 个，或者单层文件总大小超过下一层单个文件的大小，则做一次合并操作，合并时会根据参数 keepDuplicates 指定的去重策略进行去重。文件数过多时，用户也可以在线调用函数 triggerTSDBCompaction 手动触发合并，以减少文件数， 提升查询性能 。合并状态可以通过调用函数 getTSDBCompactionTaskStatus 进行查询。
但需要注意： 合并操作非常消耗 CPU 和磁盘 IO，会造成一定程度的写放大。且对于每个 volume 而言，合并是单线程进行的，当合并任务很多时，耗时将会非常久。如果在写入高峰期，发生合并操作，会降低整个系统的吞吐量。因此， 建议用户使用 scheduleJob 定时（在业务不密集的时间段）触发手动合并任务。

## 4. 合理使用 TSDB 引擎

分区、索引等因素都会影响引擎的写入和查询性能，因此在建库建表时合理地配置 TSDB 引擎至关重要。本章将从功能参数配置，以及使用时的注意事项方面，给出一些指导性的建议，方便用户快速上手。

### 4.1. 数据库部署

在使用 TSDB 引擎前，可以按需调整系统的配置项（TSDB 引擎相关的配置项一览表见 TSDB 引擎配置项汇总 ），以充分发挥系统的性能。本节主要介绍其中几个重点参数：
- TSDBRedoLogDir：为了提高写入效率，建议将 TSDB redo log 配置在 SSD 盘。
- TSDBCacheEngineSize：默认是 1G，写入压力较大的场景可以适当调大该值。 若设置过小，可能导致 cache engine 频繁刷盘，影响系统性能； 若设置过大，由于 cache engine 内缓存的数据量很大，但由于未达到 cache engine 的大小的 50% （且未达到十分钟），因此数据尚未刷盘，此时若发生了机器断电或关机，重启后就需要回放大量事务，导致系统启动过慢。
- TSDBLevelFileIndexCacheSize：默认是 5% * maxMemSize，该配置项确定了索引数据（Level File indexes 和 zonemap）的上限，若配置过小，会造成索引频繁置换。在索引部分，较占内存空间的是 zonemap 部分，用户可以根据“分区数量 × sortKey 数量 × （4 × Σ各字段字节数）”进行估算（其中 4 表示 4 种预聚合指标 min，max，sum，notnullcount）。
- TSDBAsyncSortingWorkerNum：非负整数，默认值为 1，用于指定 TSDB cache engine 异步排序的工作线程数。在 CPU 资源充足的情况下，可以适当增大该值，以提高写入性能。
- TSDBCacheFlushWorkNum：TSDB cache engine 刷盘的工作线程数，默认值是 volumes 指定的磁盘卷数。若配置值小于磁盘卷数，则仍取默认值。通常无需修改此配置。

### 4.2. 创建数据库

下述脚本以创建一个组合分区的数据库为例，和 OLAP 引擎创库时的区别仅在于 engine 设置不同：

```
db1 = database(, VALUE, 2020.01.01..2021.01.01)
db2 = database(, HASH, [SYMBOL, 100])
db = database(directory=dbName, partitionType=COMPO, partitionScheme=[db1, db2], engine="TSDB")
```

- 库表对应关系： 设计分布式数据库时，若存储的是分布式表，推荐一库一表，因为对于不同的表按照同一分区方案进行分区，可能造成每个分区的数据量不合理；若存储的是维度表，推荐一库多表，集中管理，因为维度表只有一个分区，且一次加载常驻内存。
- 分区设计： TSDB 引擎单个分区推荐大小 ：400MB - 1GB（压缩前） 分布式查询按照分区加载数据进行并行计算（包括查询、删除、修改等操作），若分区粒度过大，可能会造成内存不足、查询并行度降低、更新删除效率降低等问题；若分区粒度过小，可能会产生大量子任务增加节点负荷、大量小文件独写增加系统负荷、控制节点元数据爆炸等问题。 分区设计步骤： 以推荐大小作为参照，先根据表中的记录数和每个字段的大小 估算数据量 ，再根据分区方案计算的 分区数（如天 + 股票 HASH10 的组合分区，可以按天数 * 10） ，通过数据量/分区数计算得到每个分区的大小。 若分区粒度不合理，调整分区粒度可以参考以下方案： 粒度过小：若采用了值分区可以考虑改成范围分区，例如按天改成按月；若采用了 HASH 分区，可以考虑改小 HASH 分区数。 粒度过大：若采用了范围分区可以考虑改成值分区，例如按年改成按月；若采用了 HASH 分区，可以考虑改大 HASH 分区数；若是一级分区，可以考虑用组合分区，此时新增一级通常是 HASH 分区，例按天单分区，粒度过大，考虑二级按股票代码 HASH 分区。
合理设置分区至关重要，如需了解详细的分区机制和如何设计合理的分区，可参见 数据库分区
- 是否允许并发写入同一分区： 此外，为了支持用户多线程能够并发写数据且不会因写入分区冲突而失败，DolphinDB 在创建数据库时支持了一个特殊的配置参数 atomic（该参数是 OLAP 和 TSDB 引擎共有参数）： 默认为‘TRANS'，即不允许并发写入同一个 CHUNK 分区； 设置为‘CHUNK'，则允许多线程并发写入同一分区。系统内部仍会串行执行写入任务，当一个线程正在写入某个分区时，其他写入该分区的线程检测到冲突会不断尝试重新写入。尝试 250 次（每次尝试间隔时间会随着尝试次数增加而增加，上限是 1s，整个尝试过程约持续 5 分钟）后仍然无法写入，则会写入失败。atomic='CHUNK' 配置可能会破坏事务的原子性，因为若某线程冲突重试达到上限后仍失败，则该部分数据将会丢失，需要谨慎设置。
实际场景下，若设置为 'CHUNK' 发生数据丢失，用户可能难以定位到具体的分区，针对该场景有几个较推荐的方案：
- 使用 tableInsert 写入，该函数会返回写入数据的记录数，根据记录数可以定位到写入失败的线程任务，若线程涉及的分区没有重叠，可以删除相关分区数据后重新写入。
- 若使用 TSDB 引擎，且去重策略设置为了 FIRST 或者 LAST，则直接重复提交写入失败的线程任务，系统会进行去重，查询时不会读出重复数据。不适用于去重策略为 ALL 的场景，若为 ALL，则参照方案 1。

### 4.3. 创建数据表

创建分布式表/维度表时，与 OLAP 引擎不同，TSDB 需要额外设置 sortColumns 这个必选参数，以及 keepDuplicates, sortKeyMappingFunction, softDelete 这几个可选参数。

```
// 函数
createPartitionedTable(dbHandle, table, tableName, [partitionColumns], [compressMethods], [sortColumns], [keepDuplicates=ALL], [sortKeyMappingFunction], [softDelete=false])

// 标准 SQL
create table dbPath.tableName (
    schema[columnDescription]
)
[partitioned by partitionColumns],
[sortColumns],
[keepDuplicates=ALL],
[sortKeyMappingFunction]
```

这里以通过函数创建一个 TSDB 下的分布式表为例：

```
colName = `SecurityID`TradeDate`TradeTime`TradePrice`TradeQty`TradeAmount`BuyNo`SellNo
colType = `SYMBOL`DATE`TIME`DOUBLE`INT`DOUBLE`INT`INT
tbSchema = table(1:0, colName, colType)
db.createPartitionedTable(table=tbSchema, tableName=tbName, partitionColumns=`TradeDate`SecurityID, compressMethods={TradeTime:"delta"}, sortColumns=`SecurityID`TradeDate`TradeTime, keepDuplicates=ALL)
```

- 对字段应用合适的压缩算法（compressMethods）： 对重复较高的字符串，使用 SYMBOL 类型存储。需要注意单个分区下 SYMBOL 字段的唯一值数不能超过 2^21(2097152) 个，否则会抛异常，见 S00003 。 对时序数据或顺序数据（整型）可以使用 delta 算法存储，压缩性能测试可以参考 物联网应用范例 。 其余默认 lz4 压缩算法。
- 索引降维（sortKeyMappingFunction）： 如果单纯的依靠合理配置 sortColumns 仍然不能降低每个分区的 sortKey 数量，则可以通过指定该参数进行降维。 例如： 5000 只股票，按照日期分区，设置股票代码和时间戳为 sortColumns，每个分区 sortKey 组合数约为 5000，不满足每个分区 sortKey 组合数不超过 1000 的原则，则可以通过指定 sortKeyMappingFunction=[hashBucket{, 500}] 进行降维，使每个分区的 sortKey 组合数降为 500。 降维是对每个 sortKey 的字段进行的，因此有几个 sortKey 字段就需要指定几个降维函数。 常用的降维函数是 hashBucket，即进行哈希映射。 降维后可以通过 getTSDBSortKeyEntry 查询每个分区的 sortKey 信息。
- 数据去重（keepDuplicates）： 对 同一个时间点产生多条数据，需要去重的场景， 可以根据业务需要设置 keepDuplicates=FIRST/LAST，分别对应每个 sortColumns 保留第一条还是最后一条数据。 若对去重策略没有要求，则可以根据以下需求进行评估，以设置去重策略： 高频更新：建议指定 keepDuplicates=LAST，因为 LAST 采用追加更新的方式效率更高（见 更新流程 ）。 查询性能：较推荐使用 keepDuplicates=ALL，因为其他去重策略查询时有额外的去重开销。 atomic=CHUNK：推荐使用 keepDuplicates=FIRST/LAST，若并发写入失败，直接重复写入即可，无需删除数据再写。

## 5. TSDB 引擎查询优化案例

- CPU：48 核
- 内存：128 G
- 磁盘：HDD
- 数据库设置：单机集群 2 数据节点

### 5.1. sortKey 数量对查询性能的影响

推荐每个分区的 sortKey 组合数不要超过 1000。
本节以一个极端场景为例，只利用 sortColumns 去重的特性，将其作为主键或唯一约束使用，每个 sortKey 只对应很少的数据。
宽表存储，包含 100 个指标值。
| 字段名 | 字段类型 | 说明 |
| --- | --- | --- |
| tradeDate | DATE |  |
| tradeTime | TIME |  |
| ID | INT | 每个日期下具有唯一性 |
| factor1~factor100 | DOUBLE |  |
写入 5 日数据，每日 50 万条数据（约 400M），总数据量为 1.9 G。
- 分区方案：单分区，按天 VALUE 分区。
- 排序列 sortColumns：ID，tradeDate，TradeTime。其中 ID 和 tradeDate 的组合值将作为 sortKey。
- 去重规则 keepDuplicates：LAST，对于重复数据只保存最新的一条。
| 属性 | 值 | 查询脚本 |
| --- | --- | --- |
| 落盘实际大小 | 17.05 G | use opsgetTableDiskUsage(dbName, tableName) |
| sortKey 总数量 | 2,500,000 | chunkIDs = exec chunkID from pnodeRun(getChunksMeta{"/test_tsdb%"}) where dfsPath not like "%tbl%" a |
| 查询前 10 条数据耗时 | 59.5 s | timer select top 10 * from loadTable(dbName, tableName) |
| 按照 ID 和 tradeDate 点查耗时 | 4.5 s | timer select * from loadTable(dbName, tableName) where ID = 1 and tradeDate = 2023.03.09 |
当每个 sortKey 对应很少的数据量的时候，会产生以下几个问题：
- 由于数据总量不变，导致 sortKey 非常多，索引信息占据大量内存。
- 数据碎片化存储，导致数据读写效率降低。
- 由于数据按照 block 进行压缩，而 block 是每个 sortKey 对应数据按照数据量进行划分的，若每个 block 数据量很少，压缩率也会降低。
若要对上述场景进行优化 ，可通过设置 sortKeyMappingFunction 对 sortKey 字段进行降维处理。该场景下，我们对 ID 字段进行降维，降维函数为 hashBucket{,500}，该函数用于将每个分区的 sortKey 数量降为 500：

```
db.createPartitionedTable(table=tbSchema,tableName=tbName,partitionColumns=`tradeDate,sortColumns=`ID`tradeDate`TradeTime,keepDuplicates=LAST, sortKeyMappingFunction=[hashBucket{,500},])
```

| 属性 | 值 | 查询脚本 |
| --- | --- | --- |
| 落盘实际大小 | 2.88 G | use opsgetTableDiskUsage(dbName, tableName) |
| sortKey 总数量 | 2,500 | chunkIDs = exec chunkID from pnodeRun(getChunksMeta{"/test_tsdb%"}) where dfsPath not like "%tbl%" a |
| 查询前 10 条数据耗时 | 104 ms | timer select top 10 * from loadTable(dbName, tableName) |
| 按照 ID 和 tradeDate 点查耗时 | 7.62 ms | timer select * from loadTable(dbName, tableName) where ID = 1 and tradeDate = 2023.03.09 |
可以看到降维后，实际落盘大小和查询性能都有显著提升。

### 5.2. Level File 数量对查询性能的影响

不同去重策略下，Level File 合并前后对查询耗时的影响。
| 字段名 | 字段类型 |
| --- | --- |
| machineId | INT |
| datetime | TIMESTAMP |
| tag1~tag50 | DOUBLE |
每日写入 100,000,000 条数据（约 3800 M，包含 100000 个设备），通过脚本写入 10 天的数据量。
- 分区方案：组合分区，按 datetime（VALUE）+ machineId（HASH 100）分区。每个分区约 400 M 数据。
- 排序列 sortColumns：machineId，datetime。machineId 将作为 sortKey。
- 去重规则 keepDuplicates：建表时，通过 keepDuplicates 参数配置不同的去重策略，进行测试。
实际场景下，重复数据一般是由设备或通讯协议的机制产生的。一般数据重复率在 5% 左右。本实验模拟物联网场景下的数据产生，数据量也较贴合现实场景的数据重复率。
| keepDuplicates | 数据量 | 合并前占用磁盘（GB） | 合并后占用磁盘（GB） | 合并前 Level File 文件数 | 合并后 Level File 文件数 |
| --- | --- | --- | --- | --- | --- |
| ALL | 1,000,000,000 | 357.43 | 347.50 | Level 0:9,000Level 1:1,000 | Level 0:4,851Level 1:539Level 2:461 |
| FIRST | 951,623,687 | 341.23 | 325.01 | Level 0:9,000Level 1:1,000 | Level 0:2,070Level 1:230Level 2:760 |
| LAST | 951,624,331 | 341.23 | 324.04 | Level 0:9,000Level 1:1,000 | Level 0:1,656Level 1:184Level 2:816 |
- 查询单个分区所有记录 select * from loadTable(dbName, tbName_all) where datetime=2023.07.10 keepDuplicates 数据量 合并前耗时（ms） 合并后耗时（ms） ALL 100,000,000 59,566.7 47,474.3 FIRST 95,161,440 54,275.1 46,069.2 LAST 95,161,940 57,954.2 44,443.1
- 查询全表的总记录数（命中元数据） select count(*) from loadTable(dbName, tbName) keepDuplicates 数据量 合并前耗时（ms） 合并后耗时（ms） ALL 1,000,000,000 218.9 216.8 FIRST 951,623,687 6,324.1 1,180.9 LAST 951,624,331 6,470.2 958.7
（一） 对比 Level File 合并前后，每个查询语句性能的区别：
- 在不去重的场景下（ keepDuplicates =ALL），整体提升不显著，但在遍历整个分区数据时，由于文件数据减少，查询性能提升约 20 %。
- 在去重的场景下（ keepDuplicates =LAST / FIRST）： 命中 sortKey 的查询语句：性能几乎不受 Level File 文件合并的影响。 遍历分区的查询语句：性能在合并后会略所提升，约为 15 % 左右。 命中元数据的查询语句：性能在合并前后提升较为显著，约提升 6 倍。
从内部机制分析，合并 Level File 对性能的影响主要在：
- 由于文件数减少，磁盘的寻道时间减少。但在查询数量大的场景下，磁盘的寻道时间可以忽略。
- 合并后重复数据减少，从而使数据去重的耗时也减少。
- 合并后文件数减少，从而使查询遍历的开销也减少。
Level File 的去重操作是在合并阶段完成的。若磁盘上存在较多的重复数据，查询时，会将这些数据全部读取到内存再进行去重。 冗余的重复数据越多，这一步骤的耗时开销越显著 。
（二） 对比使用三种不同的去重策略时，每个查询语句性能的区别：
- 命中 sortKey 的查询语句：三种去重策略性查询性能相差不大。
- 遍历分区的查询语句：ALL 策略的查询性能会略低于 FIRST 和 LAST 策略，主要是因为 ALL 策略无需去重，FIRST 和 LAST 都有一定的去重开销。
- 命中元数据的查询语句：ALL 的性能明显优于 FIRST 和 LAST（ALL 性能约为 FIRST/LAST 的 30 倍），这是因为 ALL 只需要读元数据即可，FIRST，LAST 需要对数据去重后进行统计，开销较大。
不同的查询策略对性能的影响主要在：
- ALL 策略文件数较 FIRST 和 LAST 策略多，理论上磁盘寻道时间应该更长，但在查询数量大的场景下，磁盘的寻道时间可以忽略。
- ALL 策略在查询时无需去重，而 FIRST 和 LAST 策略在查询时，需要读出所有满足查询条件的数据，然后在内存中进行去重操作，这一步较为耗时。
在 Level File 文件数较多的场景下，若数据存在重复并设置了去重策略，则 Level File 文件合并后，遍历分区和命中元数据的查询语句性能会显著提升，点查性能提升不明显；若数据重复率低，或者未设置去重机制，则文件合并前后，查询性能提升较小。
参考 Level File 合并章节，DolphinDB TSDB 引擎内置了 Level File 的自动合并机制，依靠自动合并，大部分场景下 Level File 数量不会成为查询性能的瓶颈。
若自动合并后，Level File 数量仍比较多，且满足下述场景：
- 数据重复率较高
- 数据已经入库，不再变化（后续不再修改和写入）
用户也可以在线调用函数 triggerTSDBCompaction 手动触发合并，以减少文件数， 提升查询性能 。合并状态可以通过调用函数 getTSDBCompactionTaskStatus 进行查询。

## 6. TSDB 引擎场景实践案例

本章汇总了 DolphinDB 教程中 TSDB 引擎相关的实践案例，供大家快速索引。

### 6.1. 金融场景

| 场景 | 教程链接 | 场景描述 | 存储方案 |
| --- | --- | --- | --- |
| 中高频多因子库存储 | 中高频多因子库存储最佳实践 | 测试 TSDB 宽表存储和 TSDB 窄表存储，在 HDD 和 SDD 这两种不同的硬件配置下的存储性能。 | 推荐采用 TSDB 窄表存储 |
| Level 2 行情数据存储 | 金融 PoC 用户历史数据导入指导手册之股票 level2 逐笔篇处理 Level 2 行情数据实例搭建行情回放服务的最佳实践 | 介绍行情快照、逐笔委托、逐笔成交的存储方案，以及数据导入的处理流程。 | 引擎：TSDB分区方案：交易日按值分区 + 标的哈希 20 分区sortColumns：market，SecurityID，TradeTime |
| 公募基金数据存储 | 公募基金历史数据基础分析教程 | 提供公开市场数据导入 TSDB 引擎的示例脚本。 | 数据量较小，故存储为 TSDB 引擎维度表，sortColumns 为日期列。 |
| 实时波动率预测 | 金融实时实际波动率预测 | 比较数据预处理阶段，OLAP 引擎和 TSDB 引擎的存储效率。 | 使用 TSDB 引擎，将多档数据以Array Vector的形式存储，原 40 列数据合并为 4 列，在数据压缩率、数据查询和计算性能上都会有大幅提升。 |
| ETL 数据清洗优化 | 利用 DolphinDB 高效清洗数据 | 优化 ETL 数据清洗流程的性能。 | 利用 TSDB 的索引机制，可以通过扫描稀疏索引文件，来查询对应的数据块 ID。进而只读取对应数据块，从而避免全表扫描。 |

### 6.2. 物联网场景

| 案例 | 教程链接 | 场景描述 | 存储方案 |
| --- | --- | --- | --- |
| 地震波形数据存储 | 地震波形数据存储解决方案 | 介绍地震波形数据的存储方案，包含分区方案和字段压缩方案。 | 引擎：TSDB 分区方案：设备 ID 值分区 + 按天值分区 sortColumns：设备 ID + 时间戳 |
| 实时数据异常预警：流数据入库存储 | 物联网实时数据异常率预警 | 数据预警场景，订阅数据持久化部分采用 TSDB 引擎存储。 | 消费数据入库：引擎：TSDB分区方案：设备代码值分区 + 按小时值分区sortColumns：设备代码 + 时间戳聚合计算结果入库：引擎：TSDB分区方案：设备代码值分区 + 按天值分区sortCol |

## 7. 总结

TSDB 引擎较 OLAP 引擎而言，内部实现机制更加复杂，对用户开放的配置也更多样。在使用时，需要根据用户的数据特性和查询场景，设置合理的去重机制和 sortColumns，才能够发挥 TSDB 的最佳性能。
TSDB 引擎使用注意点汇总：
- 合理设置 sortColumns 字段(建议不超过 4 列)，一个通用原则是：确保每个分区的 sortKey 组合数不超过 2000，以及每个 sortColumns 对应的数据量不能过少。
- 若查询性能较差，在引擎层面，可以从两个方面进行排查： sortKey 组合数是否过多（可以在创建分布式表时指定 sortKeyMappingFunction 进行降维）。 Level File 文件数是否过多（可以手动调用 triggerTSDBCompaction 触发合并，减少 Level File 文件数）。

## 8. 附录


### 8.1. TSDB 引擎配置项汇总

| 功能 | 配置项 | 描述 |
| --- | --- | --- |
| Redo | TSDBRedoLogDir | TSDB 存储引擎重做日志的目录 |
| Cache Engine | TSDBCacheEngineSize TSDBCacheTableBufferThreshold TSDBCacheFlushWorkNum TSDBAsyncSortingWorkerNum | 设置 TSDB 存储引擎 cache engine 的容量（单位为 GB） TSDB 引擎缓存数据进行批量排序的阈值 TSDB cache engine 刷盘的工作线程数 TSDB cache eng |
| Index | TSDBLevelFileIndexCacheSize TSDBLevelFileIndexCacheInvalidPercent | TSDB 存储引擎 level file 元数据内存占用空间上限 TSDB 引擎 level file 索引缓存淘汰后保留数据的百分比。 |
| SYMBOL 字段的缓存 | TSDBSymbolBaseEvictTime TSDBCachedSymbolBaseCapacity | TSDB 存储引擎 symbolBase 内存占用空间上限及缓存管理策略。 |

### 8.2. TSDB 引擎运维函数汇总

| 功能 | 函数 | 描述 |
| --- | --- | --- |
| Level File 合并 | triggerTSDBCompactiongetTSDBCompactionTaskStatus | 触发合并 获取合并任务状态 |
| 异步排序 | enableTSDBAsyncSortingdisableTSDBAsyncSorting | 开启异步排 关闭异步排序序 |
| Cache Engine | flushTSDBCachesetTSDBCacheEngineSizegetTSDBCacheEngineSize | 将 TSDB Cache Engine 中的数据强制刷盘 设置 TSDB Cache Engine 的内存大小 获取 TSDB Cache Engine 的内存大小 |
| 索引 | getLevelFileIndexCacheStatusinvalidateLevelIndexCache | 获取所有 level file 的索引内存占用的情况 手动删除索引缓存 |
| SYMBOL 字段的缓存 | getTSDBCachedSymbolBaseMemSizeclearAllTSDBSymbolBaseCache | 获取 TSDB 引擎中 SYMBOL 类型的字典编码的缓存大小。 清除SYMBOL 类型的字典编码的缓存 |
| 元数据信息 | getTSDBMetaDatagetTSDBSortKeyEntry | 获取 TSDB 引擎 chunk 的元数据信息 获取 TSDB 引擎已经写入磁盘的 chunk 的 sort key 信息 |
触发集群中所有分区 Level File 合并脚本参考：

```
// dbName 需要替换成自己的数据库名
chunkIDs = exec * from pnodeRun(getChunksMeta{"/dbName/%"}) where type=1
for(chunkID in chunkIDs){
    pnodeRun(triggerTSDBCompaction{chunkID})
}
// 检查是否完成 Level File 合并
select * from pnodeRun(getTSDBCompactionTaskStatus) where endTime is null
```


### 8.3. 常见问题 Q & A

| 问题 | 回答 |
| --- | --- |
| 创建报错 TSDB engine is not enabled. | 1. 检查 server 版本是否为 2.00 版本，1.30 版本不支持 TSDB 引擎。2. 检查配置文件是否设置了 TSDBCacheEngineSize 配置项。 |
| TSDB 更新数据后，查询发现更新的数据出现在表末尾，而不是按时间序排序。 | TSDB 引擎只保证 level file 内部是有序的，但是 level file 间可能是无序的，可以尝试通过调用 triggerTSDBCompaction 函数手动触发 level file  |
| 1.30 版本的 server 和 2.00 版本有什么区别？ | 相较于 1.30 版本，2.00 版本还支持了 TSDB 引擎，array vector 数据形式，DECIMAL 数据类型等功能。 |
| 如果希望表的数据写入后去重，可以使用 TSDB 引擎，然后将 sortColumns 设置为去重键吗？ | - 如果去重键只有一个字段，不建议设置为 sortColumns，因为此时该字段会被作为 sortKey，去重后，一个 sortKey 只对应一条记录，会造成索引膨胀，影响查询效率。- 如果去重键有多 |
| TSDB 引擎查询性能低于预期。 | - sortKey 字段设置是否合理，可以调用 getTSDBSortKeyEntry 函数查询 sortKey 的信息。- LevelFile 的索引缓存是否设置过小。若每次查询设计的数据范围较大， |

### 8.4. 案例脚本

- test_5_._1.dos
- test_5_._2.dos


---

> 来源: `best_practices_for_partitioned_storage.html`


# 存储金融数据的分区方案最佳实践


## 1. 概述

对数据库进行合理的分区可以显著降低系统响应延迟，提高数据吞吐量。合理的分区设计需要综合考虑数据分布的特征、查询和计算的典型场景、数据更新的频率等方方面面。
本文为入门级教程，针对几种常见的金融数据给出了较为通用的分区方案和示例脚本，以帮助初次体验 DolphinDB 的用户快速完成建库建表。
本文全部代码适用于 2.00.10 及以上版本。

## 2. 数据存储方案总览

存储方案包括金融数据存储方案和因子库存储方案两大部分。

### 2.1. 金融数据存储方案

针对不同金融数据，给出以下的分区存储方案：
| 产品大类 | 数据集 | 存储引擎 | 分区方案 | 分区列 | 排序列 |
| --- | --- | --- | --- | --- | --- |
| 股票 | Level-2 快照（沪深分表） | TSDB | 按天分区 + 按股票代码 HASH25 分区 | 交易日期 + 股票代码 | 股票代码 + 交易时间 |
| 股票 | Level-1 快照（沪深分表） | TSDB | 按天分区 + 按股票代码 HASH25 分区 | 交易日期 + 股票代码 | 股票代码 + 交易时间 |
| 股票 | 逐笔委托（沪深分表） | TSDB | 按天分区 + 按股票代码 HASH25 分区 | 交易日期 + 股票代码 | 股票代码 + 交易时间 |
| 股票 | 逐笔成交（沪深分表） | TSDB | 按天分区 + 按股票代码 HASH25 分区 | 交易日期 + 股票代码 | 股票代码 + 交易时间 |
| 股票 | Level-2 快照（沪深合并） | TSDB | 按天分区 + 按股票代码 HASH50 分区 | 交易日期 + 股票代码 | 交易所类型 + 股票代码 + 交易时间 |
| 股票 | Level-1 快照（沪深合并） | TSDB | 按天分区 + 按股票代码 HASH50 分区 | 交易日期 + 股票代码 | 交易所类型 + 股票代码 + 交易时间 |
| 股票 | 逐笔委托（沪深合并） | TSDB | 按天分区 + 按股票代码 HASH50 分区 | 交易日期 + 股票代码 | 交易所类型 + 股票代码 + 交易时间 |
| 股票 | 逐笔成交（沪深合并） | TSDB | 按天分区 + 按股票代码 HASH50 分区 | 交易日期 + 股票代码 | 交易所类型 + 股票代码 + 交易时间 |
| 股票 | 日 K 线 | OLAP | 按年分区 | 交易时间 | 无 |
| 股票 | 分钟 K 线 | OLAP | 按天分区 | 交易时间 | 无 |
| 期权 | 快照 | TSDB | 按天分区 + 按期权代码 HASH20 分区 | 交易日期 + 期权代码 | 期权代码 + 接收时间 |
| 期货 | 快照 | TSDB | 按天分区 + 按期货代码 HASH10 分区 | 交易日期 + 期货代码 | 期货代码 + 接收时间 |
| 期货 | 日 K 线 | OLAP | 按年分区 | 交易时间 | 无 |
| 期货 | 分钟 K 线 | OLAP | 按天分区 | 交易时间 | 无 |
| 银行间债券 | X-Bond 报价 | TSDB | 按天分区 | 创建日期 | 债券代码 + 创建时间 |
| 银行间债券 | X-Bond 成交 | TSDB | 按天分区 | 创建日期 | 债券代码 + 创建时间 |
| 银行间债券 | ESP 报价 | TSDB | 按天分区 | 创建日期 | 债券代码 + 创建时间 |
| 银行间债券 | ESP 成交 | TSDB | 按天分区 | 创建日期 | 债券代码 + 创建时间 |
| 银行间债券 | QB 报价 | TSDB | 按天分区 | 市场时间 | 债券代码 + 市场时间 |
| 银行间债券 | QB 成交 | TSDB | 按天分区 | 市场时间 | 债券代码 + 市场时间 |

### 2.2. 因子库窄表存储方案

因子挖掘是量化交易的必备环节之一。随着量化交易和 AI 模型训练规模的发展，量化投资团队在投研环节势必需要处理大量因子数据。因子的存储也是一个关键问题，本文推荐使用窄表存储因子库。
相较于宽表，窄表存储有以下优势：
- 更加高效地删除过期失效因子 宽表的方案是删除全表的某一列，相比窄表方案效率低下
- 更加高效地添加新因子 宽表的方案是先增加列，然后更新新增列的值，相比窄表方案效率低下
- 更加高效地按照因子名更新指定因子的值 宽表的方案是 update 全表的某一列，相比窄表效率低下
- 单次多因子面板数据的查询效率和宽表近似 DolphinDB 通过对 pivot by 方法的并行优化，保证了窄表存储方案的多因子数据查询效率
针对不同频率的因子数据，给出以下的分区存储方案，均采用 TSDB 存储引擎：
| 产品大类 | 因子频率 | 分区方案 | 分区列 | 排序列 | sortKey 降维 |
| --- | --- | --- | --- | --- | --- |
| 股票（假设共 5000 支标的） | 日频因子 | 按年分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | 30 分钟因子 | 按年分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | 10 分钟因子 | 按月分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | 1 分钟频因子 | 按日分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | 1 秒钟频因子 | 按小时分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | Level-2 快照频率因子（3 秒~5 秒） | 按日分区 + 按因子名分区 | 交易时间 + 因子名 | 股票代码 + 交易时间 | 500 |
| 股票（假设共 5000 支标的） | Level-2 逐笔频因子 | 按小时分区 + 按因子名分区 + 按股票代码 HASH10 分区 | 交易时间 + 因子名 + 股票代码 | 股票代码 + 交易时间 | 不降维 |
| 期货（假设共 200 支标的） | 500ms 频因子 | 按日分区 + 按因子名分区 | 交易时间 + 因子名 | 期货代码 + 交易时间 | 500 |

## 3. 金融数据存储方案实践

基于上一章的分区方案，本章将提供对应的 DolphinDB 脚本。

### 3.1. 股票数据


#### 3.1.1. Level-2 行情数据

股票 Level-2 行情数据包含 Level-2 快照数据、逐笔委托数据、逐笔成交数据，在分布式数据库中，如果要连接（join）多个分区的数据表，通常会非常耗时，因为涉及到的分区可能位于不同的节点上，需要在不同节点之间复制数据。为解决这个问题，DolphinDB 推出了共存储位置（co-location）的分区机制，确保同一个分布式数据库里所有表在相同分区的数据存储在相同的节点上。这样的安排，保证了这些表在连接的时候非常高效。因此在本方案中，把 Level-2 快照数据、逐笔委托数据、逐笔成交数据这些分区方案一致的数据表存入同一个数据库中。
上交所和深交所两个交易所的数据在字段结构上略有不同，用户在建立库表时可以考虑是否将两个交易所的数据进行合并，如果选择存为一张表，则表中的字段是两个交易所数据字段的并集，并新增字段 Market 用于标识数据来自哪个交易所。Level-1 的分区方案采用 Level-2 的即可。
如果您希望将通联数据导入 DolphinDB 数据库，可以参考如下教程，使用封装好的模块即可完成建库建表和 Level-2 行情数据导入：
- DolphinDBModules::easyTLDataImport 通联历史数据自动化导入功能模块使用教程
下面将根据是否将沪深交易所分开存储介绍不同建库建表的分区规则。

##### 3.1.1.1. Level-2 快照（沪深分表）


```
create database "dfs://split_SZ_TB"
partitioned by VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 25])
engine='TSDB'

create table "dfs://split_SZ_TB"."split_SZ_snapshotTB"(
    TradeDate DATE[comment="交易日期", compress="delta"]   
    TradeTime TIME[comment="交易时间", compress="delta"]
    MDStreamID SYMBOL
    SecurityID SYMBOL
    SecurityIDSource SYMBOL
    TradingPhaseCode SYMBOL
    PreCloPrice DOUBLE
    NumTrades LONG
    TotalVolumeTrade LONG
    TotalValueTrade DOUBLE
    LastPrice DOUBLE
    OpenPrice DOUBLE
    HighPrice DOUBLE
    LowPrice DOUBLE
    DifPrice1 DOUBLE
    DifPrice2 DOUBLE
    PE1 DOUBLE
    PE2 DOUBLE
    PreCloseIOPV DOUBLE
    IOPV DOUBLE
    TotalBidQty LONG
    WeightedAvgBidPx DOUBLE
    TotalOfferQty LONG
    WeightedAvgOfferPx DOUBLE
    UpLimitPx DOUBLE
    DownLimitPx DOUBLE
    OpenInt INT
    OptPremiumRatio DOUBLE
    OfferPrice DOUBLE[]
    BidPrice DOUBLE[]
    OfferOrderQty LONG[]
    BidOrderQty LONG[]
    BidNumOrders INT[]
    OfferNumOrders INT[]
    LocalTime TIME
    SeqNo INT
    OfferOrders LONG[]
    BidOrders LONG[]
)
partitioned by TradeDate, SecurityID,
sortColumns=[`SecurityID,`TradeTime],
keepDuplicates=ALL
```

Level-2 行情数据使用“ 时间维度按天 + 股票维度 HASH25 ”的分区规则，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。下面对上述脚本进行说明：
- TSDB 引擎只支持创建分布式数据库，库名前必须添加”dfs://”；
- 在此处建立分区表时对某些列使用了指定压缩算法，对 TradeTime 列使用了“delta”压缩算法，其余列则默认使用“lz4”压缩算法；
- 建表时指定 OfferPrice 为 DOUBLE[] 类型，表示将 10 档卖价数据存 储在一个字段中，该字段类型为 DOUBLE 类型的数组向量（arrray vector），更详细的 array vector 介绍见： DolphinDB 中有关 array vector 的最佳实践指南 ；
- sortColumns 参数用于将写入的数据按照指定字段进行排序，系统默认 sortColumns （指定多列时）除最后一列外的列字段作为排序的索引列，称作 sort key。频繁查询的字段适合设置为 sortColumns ，且建议优先把查询频率高的字段作为 sortColumns 中位置靠前的列；
- keepDuplicates 参数表示在每个分区内如何处理所有 sortColumns 之值皆相同的数据，在两个函数中该参数的值均为"ALL"表示保留所有数据。

```
create database "dfs://split_SH_TB"
partitioned by VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 25])
engine='TSDB'

create table "dfs://split_SH_TB"."split_SH_snapshotTB"(
    TradeDate DATE[comment="交易日期", compress="delta"]   
    TradeTime TIME[comment="交易时间", compress="delta"]
    SecurityID SYMBOL
    ImageStatus INT
    PreCloPrice DOUBLE
    OpenPrice DOUBLE
    HighPrice DOUBLE
    LowPrice DOUBLE
    LastPrice DOUBLE
    ClosePrice DOUBLE
    TradingPhaseCode SYMBOL
    NumTrades LONG
    TotalVolumeTrade LONG
    TotalValueTrade DOUBLE
    TotalBidQty LONG
    WeightedAvgBidPx DOUBLE
    AltWAvgBidPri DOUBLE
    TotalOfferQty LONG
    WeightedAvgOfferPx DOUBLE
    AltWAvgAskPri DOUBLE
    ETFBuyNumber INT
    ETFBuyAmount LONG
    ETFBuyMoney DOUBLE
    ETFSellNumber INT
    ETFSellAmount LONG
    ETFSellMoney DOUBLE
    YieldToMatu DOUBLE
    TotWarExNum DOUBLE
    UpLimitPx DOUBLE
    DownLimitPx DOUBLE
    WithdrawBuyNumber INT
    WithdrawBuyAmount LONG
    WithdrawBuyMoney DOUBLE
    WithdrawSellNumber INT
    WithdrawSellAmount LONG
    WithdrawSellMoney DOUBLE
    TotalBidNumber INT
    TotalOfferNumber INT
    MaxBidDur INT
    MaxSellDur INT
    BidNum INT
    SellNum INT
    IOPV DOUBLE
    OfferPrice DOUBLE[]
    BidPrice DOUBLE[]
    OfferOrderQty LONG[]
    BidOrderQty LONG[]
    BidNumOrders INT[]
    OfferNumOrders INT[]
    LocalTime TIME
    SeqNo INT
    OfferOrders LONG[]
    BidOrders LONG[]
)
partitioned by TradeDate, SecurityID,
sortColumns=[`SecurityID,`TradeTime],
keepDuplicates=ALL
```

上交所的数据格式与深交所略有不同，但分区方案一致。

##### 3.1.1.2. 逐笔委托（沪深分表）

用户可以使用上述代码以创建存储深交所逐笔委托数据的数据库表，分区规则采用了“ 时间维度按日 + 股票代码 HASH25 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。
上交所的数据格式与深交所略有不同，但分区方案一致。

##### 3.1.1.3. 逐笔成交（沪深分表）

用户可以使用上述代码以创建存储深交所逐笔委托数据的数据库表，分区规则采用了“ 时间维度按日 + 股票代码 HASH25 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。
上交所的数据格式与深交所略有不同，但分区方案一致。

##### 3.1.1.4. Level-2 快照（沪深合并）


```
create database "dfs://merge_TB"
partitioned by VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 50])
engine='TSDB'

create table "dfs://merge_TB"."merge_snapshotTB"(
	Market SYMBOL
    TradeDate DATE[comment="交易日期", compress="delta"]   
    TradeTime TIME[comment="交易时间", compress="delta"]
    MDStreamID SYMBOL
    SecurityID SYMBOL
    SecurityIDSource SYMBOL
    TradingPhaseCode SYMBOL
    ImageStatus INT
    PreCloPrice DOUBLE
    NumTrades LONG
    TotalVolumeTrade LONG
    TotalValueTrade DOUBLE
    LastPrice DOUBLE
    OpenPrice DOUBLE
    HighPrice DOUBLE
    LowPrice DOUBLE
    ClosePrice DOUBLE
    DifPrice1 DOUBLE
    DifPrice2 DOUBLE
    PE1 DOUBLE
    PE2 DOUBLE
    PreCloseIOPV DOUBLE
    IOPV DOUBLE
    TotalBidQty LONG
    WeightedAvgBidPx DOUBLE
    AltWAvgBidPri DOUBLE
    TotalOfferQty LONG
    WeightedAvgOfferPx DOUBLE
    AltWAvgAskPri DOUBLE
    UpLimitPx DOUBLE
    DownLimitPx DOUBLE
    OpenInt INT
    OptPremiumRatio DOUBLE
    OfferPrice DOUBLE[]
    BidPrice DOUBLE[]
    OfferOrderQty LONG[]
    BidOrderQty LONG[]
    BidNumOrders INT[]
    OfferNumOrders INT[]
    ETFBuyNumber INT
    ETFBuyAmount LONG
    ETFBuyMoney DOUBLE
    ETFSellNumber INT
    ETFSellAmount LONG
    ETFSellMoney DOUBLE
    YieldToMatu DOUBLE
    TotWarExNum DOUBLE
    WithdrawBuyNumber INT
    WithdrawBuyAmount LONG
    WithdrawBuyMoney DOUBLE
    WithdrawSellNumber INT
    WithdrawSellAmount LONG
    WithdrawSellMoney DOUBLE
    TotalBidNumber INT
    TotalOfferNumber INT
    MaxBidDur INT
    MaxSellDur INT
    BidNum INT
    SellNum INT
    LocalTime TIME
    SeqNo INT
    OfferOrders LONG[]
    BidOrders LONG[]
)
partitioned by TradeDate, SecurityID,
sortColumns=[`Market,`SecurityID,`TradeTime],
keepDuplicates=ALL
```

用户可以使用上述代码建立同时存储沪深 Level-2 行情数据的数据库表，使用“ 时间维度按天 + 股票维度 HASH50 ”的分区规则，使用存储引擎 TSDB ，使用“ 交易所类型 + 股票代码 + 交易时间 ”作为排序列。与沪深分开存储的建库建表的分区规则类似，仅在 HASH 数量和排序列方面有所区别。

##### 3.1.1.5. 逐笔委托（沪深合并）

用户可以使用上述代码建立同时存储沪深 Level-2 行情数据的数据库表，使用“ 时间维度按天 + 股票维度 HASH50 ”的分区规则，使用存储引擎 TSDB ，使用“ 交易所类型 + 股票代码 + 交易时间 ”作为排序列。与沪深分开存储的建库建表的分区规则类似，仅在 HASH 数量和排序列方面有所区别。

##### 3.1.1.6. 逐笔成交（沪深合并）

用户可以使用上述代码建立同时存储沪深 Level-2 行情数据的数据库表，使用“ 时间维度按天 + 股票维度 HASH50 ”的分区规则，使用存储引擎 TSDB ，使用“ 交易所类型 + 股票代码 + 交易时间 ”作为排序列。与沪深分开存储的建库建表的分区规则类似，仅在 HASH 数量和排序列方面有所区别。

#### 3.1.2. 股票日 K 线


```
create database "dfs://k_day_level"
partitioned by RANGE(2000.01M + (0..30)*12)
engine='OLAP'

create table "dfs://k_day_level"."k_day"(
	securityid SYMBOL  
	tradetime TIMESTAMP
	open DOUBLE        
	close DOUBLE       
	high DOUBLE        
	low DOUBLE
	vol INT
	val DOUBLE
	vwap DOUBLE
)
partitioned by tradetime
```

用户可以使用上述代码建立同时存储日 K 的数据库表，使用“ 时间维度按年 ”的分区规则，使用存储引擎 OLAP 。每个分区内都包含了这一年所有股票的日 K 数据。

#### 3.1.3. 股票分钟 K 线


```
create database "dfs://k_minute_level"
partitioned by VALUE(2020.01.01..2021.01.01)
engine='OLAP'

create table "dfs://k_minute_level"."k_minute"(
	securityid SYMBOL  
	tradetime TIMESTAMP
	open DOUBLE        
	close DOUBLE       
	high DOUBLE        
	low DOUBLE
	vol INT
	val DOUBLE
	vwap DOUBLE
)
partitioned by tradetime
```

用户可以使用上述代码建立同时存储日 K 的数据库表，使用“ 时间维度按天 ”的分区规则，存储引擎使用 OLAP 。每个分区内都包含了当天所有股票的分钟 K 数据。

### 3.2. 期权数据


#### 3.2.1. 期权快照


```
create database "dfs://ctp_options"
partitioned by VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 20])
engine='TSDB'

create table "dfs://ctp_options"."options"(
    TradingDay DATE[comment="交易日期", compress="delta"]
    ExchangeID SYMBOL
    LastPrice DOUBLE
    PreSettlementPrice DOUBLE
    PreClosePrice DOUBLE
    PreOpenInterest DOUBLE
    OpenPrice DOUBLE
    HighestPrice DOUBLE
    LowestPrice DOUBLE
    Volume INT
    Turnover DOUBLE
    OpenInterest DOUBLE
    ClosePrice DOUBLE
    SettlementPrice DOUBLE
    UpperLimitPrice DOUBLE
    LowerLimitPrice DOUBLE
    PreDelta DOUBLE
    CurrDelta DOUBLE
    UpdateTime SECOND
    UpdateMillisec INT
    BidPrice DOUBLE[]
    BidVolume INT[]
    AskPrice DOUBLE[]
    AskVolume INT[]
    AveragePrice DOUBLE
    ActionDay DATE
    InstrumentID SYMBOL
    ExchangeInstID STRING
    BandingUpperPrice DOUBLE
    BandingLowerPrice DOUBLE
    tradeTime TIME
    receivedTime NANOTIMESTAMP
    perPenetrationTime LONG
)
partitioned by TradingDay, InstrumentID,
sortColumns=[`InstrumentID,`ReceivedTime],
keepDuplicates=ALL
```

用户可以使用上述代码以创建存储期权快照数据的数据库表，分区规则采用了“ 时间维度按天 + 期权代码维度 HASH20 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 期权代码 + 接收时间 ”作为排序列。经测试期权快照数据采用这种方式分区存储，综合性能最佳。

### 3.3. 期货数据


#### 3.3.1. 期货快照


```
create database "dfs://ctp_futures"
partitioned by VALUE(2020.01.01..2021.01.01), HASH([SYMBOL, 10])
engine='TSDB'

create table "dfs://ctp_futures"."futures"(
    TradingDay DATE[comment="交易日期", compress="delta"]
    ExchangeID SYMBOL
    LastPrice DOUBLE
    PreSettlementPrice DOUBLE
    PreClosePrice DOUBLE
    PreOpenInterest DOUBLE
    OpenPrice DOUBLE
    HighestPrice DOUBLE
    LowestPrice DOUBLE
    Volume INT
    Turnover DOUBLE
    OpenInterest DOUBLE
    ClosePrice DOUBLE
    SettlementPrice DOUBLE
    UpperLimitPrice DOUBLE
    LowerLimitPrice DOUBLE
    PreDelta DOUBLE
    CurrDelta DOUBLE
    UpdateTime SECOND
    UpdateMillisec INT
    BidPrice DOUBLE[]
    BidVolume INT[]
    AskPrice DOUBLE[]
    AskVolume INT[]
    AveragePrice DOUBLE
    ActionDay DATE
    InstrumentID SYMBOL
    ExchangeInstID STRING
    BandingUpperPrice DOUBLE
    BandingLowerPrice DOUBLE
    tradeTime TIME
    receivedTime NANOTIMESTAMP
    perPenetrationTime LONG
)
partitioned by TradingDay, InstrumentID,
sortColumns=[`InstrumentID,`ReceivedTime],
keepDuplicates=ALL
```

用户可以使用上述代码以创建存储期货快照数据的数据库表，分区规则采用了“ 时间维度按天 + 期货代码维度 HASH10 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 期货代码 + 接收时间 ”作为排序列。经测试期货快照数据采用这种方式分区存储，综合性能最佳。

#### 3.3.2. 期货日 K 线

由于期货 K 数据与股票 K 数据类似，因此可以使用股票 K 数据的表格结构作为期货 K 数据的表格结构

```
create database "dfs://ctp_k_day_level"
partitioned by RANGE(2000.01M + (0..30)*12)
engine='OLAP'

create table "dfs://ctp_k_day_level"."ctp_k_day"(
	securityid SYMBOL  
	tradetime TIMESTAMP
	open DOUBLE        
	close DOUBLE       
	high DOUBLE        
	low DOUBLE
	vol INT
	val DOUBLE
	vwap DOUBLE
)
partitioned by tradetime
```

用户可以使用上述代码以创建存储期货日 K 数据的数据库表，与股票日 K 数据的分区规则相同，使用“ 时间维度按年 ”的分区规则，存储引擎使用 OLAP 。在一个分区中包含了这一年所有期货的日 K 数据。

#### 3.3.3. 期货分钟 K 线


```
create database "dfs://ctp_k_day_level"
partitioned by VALUE(2020.01.01..2021.01.01)
engine='OLAP'

create table "dfs://ctp_k_day_level"."ctp_k_day"(
	securityid SYMBOL  
	tradetime TIMESTAMP
	open DOUBLE        
	close DOUBLE       
	high DOUBLE        
	low DOUBLE
	vol INT
	val DOUBLE
	vwap DOUBLE
)
partitioned by tradetime
```

用户可以使用上述代码以创建存储期货分钟 K 数据的数据库表，与股票分钟 K 数据的分区规则相同，使用“ 时间维度按天 ”的分区规则，存储引擎使用 OLAP 。每个分区内都包含了当天所有期货的分钟 K 数据。

### 3.4. 银行间债券


#### 3.4.1. X-Bond 报价


```
create database "dfs://XBond"
partitioned by VALUE(2023.01.01..2023.12.31)
engine='TSDB'

create table "dfs://XBond"."XBondDepthTable"(
    bondCodeVal SYMBOL
    createDate DATE[comment="创建日期", compress="delta"]  
    createTime TIME[comment="创建时间", compress="delta"]
    marketDepth LONG
    mdBookType LONG
    messageId LONG
    messageSource STRING
    msgSeqNum LONG
    msgType STRING
    bidmdEntryPrice DOUBLE[]
    offermdEntryPrice DOUBLE[]
    bidmdEntrySize LONG[]
    offermdEntrySize LONG[]
    bidsettlType LONG[]
    offersettlType LONG[]
    bidyield DOUBLE[]
    offeryield DOUBLE[]
    bid1yieldType STRING
    offer1yieldType STRING
    bid2yieldType STRING
    offer2yieldType STRING
    bid3yieldType STRING
    offer3yieldType STRING
    bid4yieldType STRING
    offer4yieldType STRING
    bid5yieldType STRING
    offer5yieldType STRING
    bid6yieldType STRING
    offer6yieldType STRING
    securityID SYMBOL
    senderCompID STRING
    senderSubID STRING
    sendingTime TIMESTAMP
)
partitioned by createDate,
sortColumns=[`securityID, `createTime]
```

用户可以使用上述代码以创建存储 X-Bond 报价数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 创建时间 ”作为排序列。每个分区内都包含了当天所有 X-Bond 报价数据。

#### 3.4.2. X-Bond 成交

用户可以使用上述代码以创建存储 X-Bond 成交数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 创建时间 ”作为排序列。每个分区内都包含了当天所有 X-Bond 成交数据。

#### 3.4.3. ESP 报价


```
create database "dfs://ESP"
partitioned by VALUE(2023.01.01..2023.12.31)
engine='TSDB'

create table "dfs://ESP"."ESPDepthtable"(
    createDate DATE[comment="创建日期", compress="delta"]  
    createTime TIME[comment="创建时间", compress="delta"]
    bondCodeVal SYMBOL
    marketDepth LONG
    marketIndicator LONG
    mdBookType LONG
    mdSubBookType LONG
    messageId LONG
    messageSource STRING
    msgSeqNum LONG
    msgType STRING
    askclearingMethod LONG[]
    bidclearingMethod LONG[]
    askdeliveryType LONG[]
    biddeliveryType LONG[]
    askinitAccountNumSixCode LONG[]
    bidinitAccountNumSixCode LONG[]
    asklastPx DOUBLE[]
    bidlastPx DOUBLE[]
    askmdEntryDate DATE[]
    bidmdEntryDate DATE[]
    askmdEntrySize LONG[]
    bidmdEntrySize LONG[]
    askmdEntryTime TIME[]
    bidmdEntryTime TIME[]
    askmdQuoteType LONG[]
    bidmdQuoteType LONG[]
    askquoteEntryID LONG[]
    bidquoteEntryID LONG[]
    asksettlType LONG[]
    bidsettlType LONG[]
    askyield DOUBLE[]
    bidyield DOUBLE[]
    ask1initPartyTradeCode STRING
    bid1initPartyTradeCode STRING
    ask2initPartyTradeCode STRING
    bid2initPartyTradeCode STRING
    ask3initPartyTradeCode STRING
    bid3initPartyTradeCode STRING
    ask4initPartyTradeCode STRING
    bid4initPartyTradeCode STRING
    ask5initPartyTradeCode STRING
    bid5initPartyTradeCode STRING
    ask6initPartyTradeCode STRING
    bid6initPartyTradeCode STRING
    ask7initPartyTradeCode STRING
    bid7initPartyTradeCode STRING
    ask8initPartyTradeCode STRING
    bid8initPartyTradeCode STRING
    ask9initPartyTradeCode STRING
    bid9initPartyTradeCode STRING
    ask10initPartyTradeCode STRING
    bid10initPartyTradeCode STRING
    securityID SYMBOL
    securityType STRING
    senderCompID STRING
    senderSubID STRING
    sendingTime TIMESTAMP
    symbol STRING
)
partitioned by createDate,
sortColumns=[`securityID, `createTime]
```

用户可以使用上述代码以创建存储 ESP 报价数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 创建时间 ”作为排序列。每个分区内都包含了当天所有 ESP 报价数据。

#### 3.4.4. ESP 成交

用户可以使用上述代码以创建存储 ESP 成交数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 创建时间 ”作为排序列。每个分区内都包含了当天所有 ESP 成交数据。

#### 3.4.5. QB 报价


```
create database "dfs://QB_QUOTE"
partitioned by VALUE(2023.10.01..2023.10.31)
engine='TSDB'

create table "dfs://QB_QUOTE"."qbTable"(
	SENDINGTIME TIMESTAMP
	CONTRIBUTORID SYMBOL
	MARKETDATATIME TIMESTAMP
	SECURITYID SYMBOL
	BONDNAME SYMBOL
	DISPLAYLISTEDMARKET SYMBOL
	BIDQUOTESTATUS INT
	BIDYIELD DOUBLE
	BIDPX DOUBLE
	BIDPRICETYPE INT
	BIDPRICE DOUBLE
	BIDDIRTYPRICE DOUBLE
	BIDVOLUME INT
	BIDPRICEDESC STRING
	ASKQUOTESTATUS INT
	ASKYIELD DOUBLE
	OFFERPX DOUBLE
	ASKPRICETYPE INT
	ASKPRICE DOUBLE
	ASKDIRTYPRICE DOUBLE
	ASKVOLUME INT
	ASKPRICEDESC STRING
)
partitioned by MARKETDATATIME,
sortColumns=[`SECURITYID,`MARKETDATATIME]
```

用户可以使用上述代码以创建存储 QB 报价数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 市场时间 ”作为排序列。每个分区内都包含了当天所有 QB 报价数据。

#### 3.4.6. QB 成交


```
create database "dfs://QB_TRADE"
partitioned by VALUE(2023.10.01..2023.10.31)
engine='TSDB'

create table "dfs://QB_TRADE"."lastTradeTable"(
	SECURITYID SYMBOL
	BONDNAME SYMBOL
	SENDINGTIME TIMESTAMP
	CONTRIBUTORID SYMBOL
	MARKETDATATIME TIMESTAMP
	MODIFYTIME SECOND
	DISPLAYLISTEDMARKET SYMBOL
	EXECID STRING
	DEALSTATUS INT
	TRADEMETHOD INT
	YIELD DOUBLE
	TRADEPX DOUBLE
	PRICETYPE INT
	TRADEPRICE DOUBLE
	DIRTYPRICE DOUBLE
	SETTLSPEED STRING
)
partitioned by MARKETDATATIME,
sortColumns=[`SECURITYID,`MARKETDATATIME]
```

用户可以使用上述代码以创建存储 QB 交易数据的数据库表，使用“ 时间维度按天 ”的分区规则，使用存储引擎 TSDB ，使用“ 债券代码 + 市场时间 ”作为排序列。每个分区内都包含了当天所有 QB 交易数据。

## 4. 因子库窄表存储方案实践


### 4.1. 日频因子库


```
create database "dfs://dayFactorDB" 
partitioned by RANGE(date(datetimeAdd(1980.01M,0..80*12,'M'))), VALUE(`f1`f2), 
engine='TSDB'

create table "dfs://dayFactorDB"."dayFactorTB"(
    tradetime DATE[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储日频因子库的数据库表，分区规则采用了“ 时间维度按年 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。经测试日频因子数据采用这种方式分区存储，综合性能最佳。
对于分区内的分组排序存储来说，DolphinDB 中的 TSDB 存储引擎提供排序键设置，每一个分区的数据写在一个或多个 level file 中。每一个 level file 内部的数据按照指定的列进行排序且创建块索引。在排序列中，除了最后一列之外的其他列通常用作点查中的过滤条件。这些列的唯一值组合被称为 SortKeys（排序键）。为保证性能最优，每个分区的 SortKeys 建议不超过 1000 个。如 SortKeys 较多可通过设置 sortKeyMappingFunction 对 SortKeys 降维。经测试日频因子数据采用“ securityid + tradetime ”的方式进行排序， sortKeyMapping 设置为 500 时，综合性能最佳。

### 4.2. 半小时频因子库


```
create database "dfs://halfhourFactorDB"
partitioned by RANGE(date(datetimeAdd(1980.01M,0..80*12,'M'))), VALUE(`f1`f2),
engine='TSDB'

create table "dfs://halfhourFactorDB"."halfhourFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"],
    securityid SYMBOL,
    value DOUBLE,
    factorname SYMBOL,
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime],
keepDuplicates=ALL,
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储半小时频因子库的数据库表，分区规则采用了“ 时间维度按年 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。经测试半小时频因子数据采用这种方式分区存储，综合性能最佳。

### 4.3. 十分钟频因子库


```
create database "dfs://tenMinutesFactorDB" 
partitioned by VALUE(2023.01M..2023.06M), 
VALUE(`f1`f2), 
engine='TSDB'

create table "dfs://tenMinutesFactorDB"."tenMinutesFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储十分钟频因子库的数据库表，分区规则采用了“ 时间维度按月 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。经测试十分钟频因子数据采用这种方式分区存储，综合性能最佳。

### 4.4. 分钟频因子库


```
create database "dfs://minuteFactorDB" 
partitioned by VALUE(2012.01.01..2021.12.31), VALUE(`f1`f2), 
engine='TSDB'

create table "dfs://minuteFactorDB"."minuteFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储分钟频因子库的数据库表，分区规则采用了“ 时间维度按日 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。经测试分钟频因子数据采用这种方式分区存储，综合性能最佳。

### 4.5. 秒钟频因子库


```
create database "dfs://secondFactorDB" 
partitioned by VALUE(datehour(2022.01.01T00:00:00)..datehour(2022.01.31T00:00:00)), VALUE(`f1`f2), 
engine='TSDB'

create table "dfs://secondFactorDB"."secondFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储秒钟频因子库的数据库表，分区规则采用了“ 时间维度按小时 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。经测试秒钟频因子数据采用这种方式分区存储，综合性能最佳。

### 4.6. Level-2 快照频因子库


```
create database "dfs://level2FactorDB" partitioned by VALUE(2022.01.01..2022.12.31), VALUE(["f1", "f2"]), 
engine='TSDB'

create table "dfs://level2FactorDB"."level2FactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储 Level-2 快照频因子库的数据库表，分区规则采用了“ 时间维度按日 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。根据测试结果，Level-2 快照频因子数据采用这种方式进行分区存储，可以实现最佳的综合性能。

### 4.7. 逐笔频因子库


```
create database "dfs://tickFactorDB" partitioned by VALUE(2022.01.01..2022.12.31), VALUE(["f1", "f2"]), HASH([SYMBOL,10])
engine='TSDB'

create table "dfs://tickFactorDB"."tickFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,securityid,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL
```

用户可以使用上述代码创建存储逐笔频因子库的数据库表，分区规则采用了“ 时间维度按日 + 因子名维度 + 股票维度 HASH10 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 股票代码 + 交易时间 ”作为排序列。根据测试结果，逐笔频因子数据采用了这种分区存储方式，可以获得最佳的综合性能。与之前提到的因子存储方案不同之处在于，这里没有对排序键（sortKey）进行降维处理。原因是每天的数据按照股票代码维度进行了 10 个哈希分区，而每个最小分区内包含 500 个股票，因此无需再进行降维处理。

### 4.8. 期货 500ms 频因子库


```
create database "dfs://futuresFactorDB" 
partitioned by VALUE(2022.01.01..2023.01.01), VALUE(["f1", "f2"]), 
engine='TSDB' 

create table "dfs://futuresFactorDB"."futuresFactorTB"(
    tradetime TIMESTAMP[comment="时间列", compress="delta"], 
    securityid SYMBOL, 
    value DOUBLE, 
    factorname SYMBOL
)
partitioned by tradetime, factorname,
sortColumns=[`securityid, `tradetime], 
keepDuplicates=ALL, 
sortKeyMappingFunction=[hashBucket{, 500}]
```

用户可以使用上述代码以创建存储 L 期货 500ms 频因子库的数据库表，分区规则采用了“ 时间维度按日 + 因子名 ”的组合分区方式存储，使用存储引擎 TSDB ，使用“ 期货代码 + 交易时间 ”作为排序列。根据测试结果，采用这种分区存储方式可以实现期货 500 毫秒频率因子数据的最佳综合性能。

## 5. 结语

针对常见的金融数据，我们通过最佳实践，给出上述建库建表的推荐方案，用户可以根据实际业务场景，进行适当调整和修改。
- DolphinDB 教程：分区数据库
- TSDB 存储引擎详解

## 6. 常见问题与解答（FAQ）


### 6.1. 创建数据库表时报错：已经存在重名库表

执行创建数据库表的代码后，报出如下错误：

```
create database partitioned by VALUE(["f1","f2"]), engine=TSDB => It is not allowed to overwrite an existing database.
```

解决方法 ：建议使用其他名字或者删掉该数据库。
删除指定数据库的代码如下：

```
dropDatabase("dfs://dayFactorDB")
```

检查指定数据库是否存在的代码如下：

```
existsDatabase("dfs://dayFactorDB")
```



---

> 来源: `database.html`


# 分区数据库设计和操作


## 1. 为什么对数据库进行分区

对数据库进行分区可以显著降低系统响应延迟，提高数据吞吐量。具体来说，分区有以下主要好处。
- 分区使得大型表更易于管理。对数据子集的维护操作也更加高效，因为这些操作只针对需要的数据而不是整个表。一个好的分区策略通过只读取查询所需的相关数据来减少要扫描的数据量。如果分区机制设计不合理，对数据库的查询、计算以及其它操作都可能受到磁盘访问 I/O 这个瓶颈的限制。
- 分区使得系统可以充分利用所有资源。选择一个良好的分区方案搭配并行计算，分布式计算可以充分利用所有节点来完成通常要在一个节点上完成的任务。若一个任务可以拆分成几个子任务，每个子任务访问不同的分区，可以显著提升效率。

## 2. DolphinDB 分区和基于 MPP 架构的数据存储的区别

MPP(Massive Parallel Processing) 是目前主流数据仓库普遍采用的一种方案，包括开源软件 Greenplum，云数据库 AWS Redshift 等。MPP 有一个主节点，每个客户端都连接到这个主节点。DolphinDB 在数据库层面不存在主节点，是点对点结构，每个客户端可以连接到任何一个数据节点，不会出现主节点瓶颈问题。
MPP 一般通过哈希规则，将数据分布到各个节点上（水平分割），在各个节点内部再进行分区（垂直分割）。哈希时容易出现各个节点数据分布不均匀的问题。DolphinDB 将各个节点的存储空间交给内置的分布式文件系统（DFS）统一进行管理，分区的规则与分区的存储位置解耦，数据分割不再按水平和垂直两个步骤进行，而是进行全局优化。这样一来，分区的粒度更细更均匀，在计算时能充分的利用集群的所有计算资源。
由于分布式文件系统具有强大的分区管理、容错、复制机制，以及事务管理机制，DolphinDB 的单表能轻松的支持百万级别的分区。若每个分区有 1GB 的数据，就可实现 PB 级数据的存储和快速查询。另外，通过引入 DFS，数据库的存储和数据库节点相分离，使得 DolphinDB 在集群水平扩展（新增节点）上更加方便。

## 3. 分区类型

DolphinDB database 支持多种分区类型：范围分区、哈希分区、值分区、列表分区与复合分区。选择合适的分区类型，有助于用户根据业务特点对数据进行均匀分割。
- 范围分区对每个分区区间创建一个分区。
- 哈希分区利用哈希函数对分区列操作，方便建立指定数量的分区。
- 值分区每个值创建一个分区，例如股票交易日期、股票交易月等。
- 列表分区是根据用户枚举的列表来进行分区，比值分区更加灵活。
- 复合分区适用于数据量特别大而且 SQL where 或 group by 语句经常涉及多列。可使用 2 个或 3 个分区列，每个分区选择都可以采用区间、值、哈希或列表分区。例如按股票交易日期进行值分区，同时按股票代码进行范围分区。
创建一个新的分布式数据库时，需要在 database 函数中指定数据库路径 directory，分区类型 partitionType 以及分区模式 partitionScheme。重新打开已有的分布式数据库时，只需指定数据库路径。不允许用不同的分区类型或分区方案覆盖已有的分布式数据库。
聚合函数在分区表上利用分区列操作时，例如当 group by 列与分区列一致时，运行速度特别快。
为了学习方便，以下分区例子使用 Windows 本地目录，用户可以将数据库创建使用的路径改成 Linux 或 DFS 目录。
调用 database 函数前，用户必须先登录，只有具有 DB_OWNER 或 admin 管理员权限才能创建数据库。默认的 admin 管理员登录脚本为：

```
login(userId=`admin, password=`123456)
```

下文提供的所有创建数据库脚本，默认已经登录。

### 3.1. 范围 (RANGE) 分区

在范围分区中，分区由区间决定，而区间由分区向量的任意两个相邻元素定义。区间包含起始值，但不包含结尾值。
在下面的例子中，数据库 db 有两个分区：[0,5) 和[5,10)。使用 ID 作为分区列，并使用函数 append! 在数据库 db 中保存表 t 为分区表 pt。

```
n=1000000
ID=rand(10, n)
x=rand(1.0, n)
t=table(ID, x)
db=database("dfs://rangedb", RANGE,  0 5 10)

pt = db.createPartitionedTable(t, `pt, `ID)
pt.append!(t)

pt=loadTable(db,`pt)
select count(x) from pt;
```

范围分区创建后，可使用 addRangePartitions 函数来追加分区。细节参见用户手册。

### 3.2. 哈希 (HASH) 分区

哈希分区对分区列使用哈希函数以产生分区。哈希分区是产生指定数量的分区的一个简便方法。但是要注意，哈希分区不能保证分区的大小一致，尤其当分区列的值的分布存在偏态的时候。此外，若要查找分区列中一个连续范围的数据时，哈希分区的效率比范围分区或值分区要低。
在下面的例子中，数据库 db 有两个分区。使用 ID 作为分区列，并使用函数 append! 在数据库 db 中保存表 t 为分区表 pt。

```
n=1000000
ID=rand(10, n)
x=rand(1.0, n)
t=table(ID, x)
db=database("dfs://hashdb", HASH,  [INT, 2])

pt = db.createPartitionedTable(t, `pt, `ID)
pt.append!(t)

pt=loadTable(db,`pt)
select count(x) from pt;
```


### 3.3. 值 (VALUE) 分区

在值域（VALUE）分区中，一个值代表一个分区。

```
n=1000000
month=take(2000.01M..2016.12M, n)
x=rand(1.0, n)
t=table(month, x)

db=database("dfs://valuedb", VALUE, 2000.01M..2016.12M)

pt = db.createPartitionedTable(t, `pt, `month)
pt.append!(t)

pt=loadTable(db,`pt)
select count(x) from pt;
```

上面的例子定义了一个具有 204 个分区的数据库 db。每个分区是 2000 年 1 月到 2016 年 12 月之间的一个月 (如下图）。在数据库 db 中，表 t 被保存为分区表 pt，分区列为 month。
在默认配置（newValuePartitionPolicy=add）情况下，无需手动添加新的分区，当写入数据时，会自动增加对应分区。如本例中，2017年01月的数据写入时，会自动创建201701M这个分区。

### 3.4. 列表 (LIST) 分区

在列表（LIST）分区中，我们用一个包含多个元素的列表代表一个分区。

```
n=1000000
ticker = rand(`MSFT`GOOG`FB`ORCL`IBM,n)
x=rand(1.0, n)
t=table(ticker, x)

db=database("dfs://listdb", LIST, [`IBM`ORCL`MSFT, `GOOG`FB])
pt = db.createPartitionedTable(t, `pt, `ticker)
pt.append!(t)

pt=loadTable(db,`pt)
select count(x) from pt;
```

上面的数据库有 2 个分区。第一个分区包含 3 个股票代号，第二个分区包含 2 个股票代号。

### 3.5. 组合 (COMPO) 分区

组合（COMPO）分区可以定义 2 或 3 个分区列。每列可以独立采用范围 (RANGE)、值 (VALUE)、哈希 (HASH) 或列表 (LIST) 分区。组合分区的多个列在逻辑上是并列的，不存在从属关系或优先级关系。

```
n=1000000
ID=rand(100, n)
dates=2017.08.07..2017.08.11
date=rand(dates, n)
x=rand(10.0, n)
t=table(ID, date, x)

dbDate = database(, VALUE, 2017.08.07..2017.08.11)
dbID=database(, RANGE, 0 50 100)
db = database("dfs://compoDB", COMPO, [dbDate, dbID])

pt = db.createPartitionedTable(t, `pt, `date`ID)
pt.append!(t)

pt=loadTable(db,`pt)
select count(x) from pt;
```

在 20170807 这个分区中，有 2 个区间域 (RANGE) 分区：
若组合分区有一列为值分区，创建后可使用 addValuePartitions 函数来追加分区。细节参见用户手册。

## 4. 分区设计注意事项

分区的总原则是让数据管理更加高效，提高查询和计算的性能，达到低延时和高吞吐量。下面是设计和优化分区表的需要考虑的因素，以供参考。

### 4.1. 选择合适的分区字段

在 DolphinDB 中，可以用于分区的数据类型包括整型 (CHAR, SHORT, INT)，日期类型 (DATE, MONTH, TIME, MINUTE, SECOND, DATETIME, DATEHOUR)，以及 STRING 与 SYMBOL。除此之外，哈希分区还支持 LONG, UUID, IPADDR, INT128 类型。虽然 STRING 可作为分区列，但为了性能考虑，建议将 STRING 转化为 SYMBOL 再用于分区列。
FLOAT 和 DOUBLE 数据类型不可作为分区字段。

```
db=database("dfs://rangedb1", RANGE,  0.0 5.0 10.0)
```

会产生出错信息：DOUBLE 数据类型的字段不能作为分区字段

```
The data type DOUBLE can't be used for a partition column
```

虽然 DolphinDB 支持对 TIME, SECOND, DATETIME 类型字段的分区，但是在实际使用中要尽量避免对这些数据类型采用值分区，以免分区粒度过细，将耗费大量时间创建或查询百万级以上的很小的分区。例如下面这个例子就会产生过多的分区。序列：2012.06.01T09:30:00..2012.06.30T16:00:00 包含 2,529,001 个元素。如果用这个序列进行值分区，将会在磁盘上产生 2,529,001 个分区，即 2,529,001 个文件目录和相关文件，从而使得分区表创建、写入、查询都非常缓慢。
分区字段应当在业务中，特别是数据更新的任务中有重要相关性。譬如在证券交易领域，许多任务都与股票交易日期或股票代码相关，因此以这两个字段来分区比较合理。更新数据库时，DolphinDB 的事务机制（在 5.2 中会提到）不允许多个 writer 的事务在分区上有重叠。鉴于经常需要对某个交易日或某只股票的数据进行更新，若采用其它分区字段（例如交易时刻），有可能造成多个 writer 同时对同一分区进行写入而导致问题。
一个分区字段相当于数据表的一个物理索引。如果查询时用到了该字段做数据过滤，SQL 引擎就能快速定位需要的数据块，而无需对整表进行扫描，从而大幅度提高处理速度。因此，分区字段应当选用查询和计算时经常用到的过滤字段。

### 4.2. 分区粒度不要过大

一个分区内的多个列以文件形式独立存储在磁盘上，通常数据是经过压缩的。使用的时候，系统从磁盘读取所需要的列，解压后加载到内存。若分区粒度过大，可能会造成多个工作线程并行时内存不足，或者导致系统频繁地在磁盘和工作内存之间切换，影响性能。一个经验公式是，若数据节点的可用内存是 S，工作线程（worker）的数量是 W，建议每个分区解压后在内存中的大小不超过 S/(8*W)。假设工作内存上限为 32GB，并有 8 个工作线程，建议单个分区解压后的大小不超过 512MB。
DolphinDB 的子任务以分区为单位。因此分区粒度过大会造成无法有效利用多节点多分区的优势，将本来可以并行计算的任务转化成了顺序计算任务。
DolphinDB 是为 OLAP 的场景优化设计的，支持添加数据，不支持对个别行进行删除或更新。如果要修改数据，需以分区为单位替换全部数据。如果分区过大，会降低效率。DolphinDB 在节点之间复制副本数据时，同样以分区为单位，若分区过大，则不利于数据在节点之间的复制。
综上各种因素，建议一个分区未压缩前的原始数据大小不超过 1GB。当然这个限制可结合实际情况调整。譬如在大数据应用中，经常有宽表设计，一个表有几百个字段，但是单个应用只会使用一部分字段。这种情况下，可以适当放大上限的范围。
降低分区粒度可采用以下几种方法：（1）采用组合分区 (COMPO)；（2）增加分区个数；（3）将范围分区改为值分区。

### 4.3. 分区粒度不要过小

若分区粒度过小，一个查询和计算作业往往会生成大量的子任务，这会增加数据节点和控制节点，以及控制节点之间的通讯和调度成本。分区粒度过小，也会造成很多低效的磁盘访问（小文件读写)，造成系统负荷过重。另外，所有的分区的元数据都会驻留在控制节点的内存中。分区粒度过小，分区数过多，可能会导致控制节点内存不足。我们建议每个分区未压缩前的数据量不要小于 100M。
综合前述，推荐分区大小控制在 100MB 到 1GB 之间。
股票的高频交易数据若按交易日期和股票代码的值做组合分区，会导致许多极小的分区，因为许多交易不活跃的股票的交易数据量太少。如果将股票代码的维度按照范围分区的方法来切分数据，将多个交易不活跃的股票组合在一个分区内，则可以有效解决分区粒度过小的问题，提高系统的性能。

### 4.4. 如何将数据均匀分区

当各个分区的数据量差异很大时，会造成系统负荷不均衡，部分节点任务过重，而其它节点处于闲置等待状态。当一个任务有多个子任务时，只有最后一个子任务完成了，才会将结果返回给用户。由于一个子任务对应一个分区，如果数据分布不均匀，可能会增大作业延时，影响用户体验。
为了方便根据数据的分布进行分区，DolphinDB 提供了函数 cutPoints(X, N, [freq]) 。这里 X 是一个数组，N 指需要产生多少组，而 freq 是 X 的等长数组，其中每个元素对应着 X 中元素出现的频率。函数返回具有 (N + 1) 个元素的数组，代表 N 个组，使得 X 中的数据均匀地分布在这 N 个组中。
下面的例子中，需要对股票的报价数据按日期和股票代码两个维度做数据分区。如果简单的按股票的首字母进行范围分区，极易造成数据分布不均，因为极少量的股票代码以 U, V, X，Y，Z 等字母开头。我们这里使用 cutPoints 函数将 2020 年 10 月 01 日到 2020 年 10 月 29 日的数据根据股票代码划为 5 个分区，

```
dates=2020.10.01..2020.10.29;
syms="A"+string(1..13);
syms.append!(string('B'..'Z'));
buckets=cutPoints(syms,5);//cutpoints
t1=table(take(syms,10000) as stock, rand(dates,10000) as date, rand(10.0,10000) as x);
dateDomain = database("", VALUE, dates);
symDomain = database("", RANGE, buckets);
stockDB = database("dfs://stockDBTest", COMPO, [dateDomain, symDomain]);
pt = stockDB.createPartitionedTable(t1, `pt, `date`stock).append!(t1);
```

除了使用范围分区的方法，列表分区也是解决数据分布不均匀的有效方法。

### 4.5. 时序类型分区

时间是实际数据中最常见的一个维度。DolphinDB 提供了丰富时间类型以满足用户的需求。当我们以时间类型字段作为分区字段时，在时间取值上可以预留分区以容纳未来的数据。下面的例子，我们创建一个数据库，以天为单位，将 2000.01.01 到 2030.01.01 的日期分区。注意，只有当实际数据写入数据库时，数据库才会真正创建需要的分区。

```
dateDB = database("dfs://testDate", VALUE, 2000.01.01 .. 2030.01.01)
```

DolphinDB 使用时间类型作为分区字段时，还有一个特殊的优点。数据库定义的分区字段类型和数据表实际采用的时间类型可以不一致，只要保证定义的分区字段数据类型精度小于等于实际数据类型即可。比如说，如果数据库是按月（month）分区，数据表的字段可以是 month, date, datetime, timestamp 和 nanotimestamp。系统自动会作数据类型的转换。

### 4.6. 不同表相同分区的数据存于同一节点

在分布式数据库中，如果多个分区的数据表要连接（join）通常十分耗时，因为涉及到的分区可能在不同的节点上，需要在不同节点之间复制数据。为解决这个问题，DolphinDB 推出了共存储位置的分区机制，确保同一个分布式数据库里所有表在相同分区的数据存储在相同的节点上。这样的安排，保证了这些表在连接的时候非常高效。DolphinDB 当前版本对采用不同分区机制的多个分区表不提供连接功能。

```
dateDomain = database("", VALUE, 2018.05.01..2018.07.01)
symDomain = database("", RANGE, string('A'..'Z') join `ZZZZZ)
stockDB = database("dfs://stockDB", COMPO, [dateDomain, symDomain])

quoteSchema = table(10:0, `sym`date`time`bid`bidSize`ask`askSize, [SYMBOL,DATE,TIME,DOUBLE,INT,DOUBLE,INT])
stockDB.createPartitionedTable(quoteSchema, "quotes", `date`sym)

tradeSchema = table(10:0, `sym`date`time`price`vol, [SYMBOL,DATE,TIME,DOUBLE,INT])
stockDB.createPartitionedTable(tradeSchema, "trades", `date`sym)
```

上面的例子中，quotes 和 trades 两个分区表采用同一个分区机制。

## 5. 导入数据到分布式数据表

DolphinDB 是为 OLAP 设计的系统，主要是解决海量结构化数据的快速存储和计算，以及通过内存数据库和流数据计算引擎实现高性能的数据处理。DolphinDB 不适合数据频繁更改的 OLTP 业务系统。DolphinDB 的数据写入与 Hadoop HDFS 类似，快速在每个分区或文件的末尾批量插入数据。插入的数据会压缩存储到磁盘，一般压缩比例在 20%~25%。数据一旦追加到基于磁盘的数据表后，不能快速更新或删除某些符合条件的记录，必须以分区为单位对数据表进行修改。这也是分区原则中提到单个分区不宜过大的原因之一。

### 5.1. 多副本机制

DolphinDB 允许为每一个分区保留多个副本，默认的副本个数是 2，可以修改控制节点的参数 dfsReplicationFactor 来设置副本数量。
设置冗余数据的目的有两个： （1）当某个数据节点失效或者或磁盘数据损坏时，系统提供容错功能继续提供服务； （2）当大量并发用户访问时，多副本提供负载均衡的功能，提高系统吞吐量，降低访问延时。
DolphinDB 通过两阶段事务提交机制，确保数据写入时，同一副本在多节点之间的数据强一致性。
在控制节点的参数文件 controller.cfg 中，还有一个非常重要的参数 dfsReplicaReliabilityLevel。该参数决定是否允许多个副本驻留在同一台物理服务器的多个数据节点上。在开发阶段，可允许在一个机器上配置多个节点，同时允许多个副本驻留在同一台物理服务器（dfsReplicaReliabilityLevel=0），但是在生产阶段需要设置成为 1，否则起不到容错备份的作用。

### 5.2. 事务机制

DolphinDB 对基于磁盘（分布式文件系统）的数据库表的读写支持事务，也就是说确保事务的原子性，一致性，隔离性和持久化。DolphinDB 采用多版本机制实现快照级别的隔离。在这种隔离机制下，数据的读操作和写操作互相不阻塞，可以最大程度优化数据仓库读的性能。
为了最大程度优化数据仓库查询、分析、计算的性能，DolphinDB 对事务作了一些限制：
- 首先，一个事务只能包含写或者读，不能同时进行写和读。
- 其次，一个写事务可以跨越多个分区，但当通过 database 建库且设置参数 atomic='TRANS'，则同一个分区不能被多个 writer 并发写入。当一个分区被某一个事务 A 锁定之后，另一个事务 B 试图再次去锁定这个分区时，系统立刻会抛出异常导致事务 B 失败回滚。当设置 atomic='CHUNK'时，无此限制。

### 5.3. 多 Writer 并行写入

DolphinDB 中，单个数据表可有几百万个分区，这为高性能的并行数据加载创造了条件。特别是将海量数据从其它系统导入 DolphinDB 时，或者需要将实时数据以准实时的方式写入数据仓库时，并行加载对于性能尤为重要。
下面的例子将股票报价数据（quotes）并行加载到基于日期和股票代码的复合分区数据库 stockDB。数据存储在 csv 文件 中，每个文件保存一天的报价数据。对每个文件，通过文件名产生 jobId 前缀，并通过命令 submitJob 提交后台程序调用 loadTextEx 函数将数据加载到 stockDB 数据库中。通过 pnodeRun 将上述任务发送到集群的每个数据节点进行并行加载。
在上面的例子中，database 的参数使用默认值（atomic='TRANS'），若多个 writer 并行加载数据，则需要确保这些 writer 不会同时往同一个分区写入数据，否则会导致事务失败 。在上面的例子中，每一个文件存储了一天的数据，而 quotes 表的一个分区字段是日期，从而确保所有加载数据的作业不会产生有重叠的事务。

### 5.4. 数据导入的常用方法

DolphinDB 的分布式数据库提供标准方法 append! 函数批量追加数据到到数据库。各种数据导入方法实际上就是直接或间接的调用这个函数将数据写入到数据库。后面的所有例子，都使用 5.4 中创建的 stockDB 的 quotes 表。

```
n = 1000000
syms = `IBM`MSFT`GM`C`FB`GOOG`V`F`XOM`AMZN`TSLA`PG`S
time = 09:30:00 + rand(21600000, n)
bid = rand(10.0, n)
bidSize = 1 + rand(100, n)
ask = rand(10.0, n)
askSize = 1 + rand(100, n)
quotes = table(rand(syms, n) as sym, take(2018.05.04..2018.05.11,n) as date, time, bid, bidSize, ask, askSize)

loadTable("dfs://stockDB", "quotes").append!(quotes);
```


#### 5.4.1. 从文本文件导入数据

DolphinDB 提供三个函数 loadText ， ploadText 和 loadTextEx 加载文本数据。

```
workDir = "C:/DolphinDB/Data"
if(!exists(workDir)) mkdir(workDir)
quotes.saveText(workDir + "/quotes.csv")
quotes.saveText(workDir + "/quotes_new.csv")
```

使用 loadText 或 ploadText 将数据从文件加载到内存，然后再调用 append! 函数。这种方法适合于数据量小于物理内存的情况，因为数据将被全部导入内存。 ploadText 和 loadText 的区别在于前者采用并行方法加载文本文件。

```
t=loadText(workDir + "/quotes_new.csv")
loadTable("dfs://stockDB", "quotes").append!(t)
```

loadTextEx 直接将文本数据导入到数据库分区表，是 DolphinDB 推荐使用的加载文本数据的方法。它的优点是：并行处理速度快，而且文件尺寸可远远大于物理内存。 loadTextEx 运行时，帮助用户调用了 append! 函数。

```
db = database("dfs://stockDB")
loadTextEx(db, "quotes", `date`sym, workDir + "/quotes.csv")
```


#### 5.4.2. 订阅一个流数据，批量写入

DolphinDB 支持流数据的处理。用户可以订阅一个流数据，将订阅到的流数据批量写入到分布式表中。详细内容，请参阅帮助文档关于流计算的部分。

```
dfsQuotes = loadTable("dfs://stockDB", "quotes")
saveQuotesToDFS=def(mutable t, msg): t.append!(select today() as date,* from msg)
subscribeTable(, "quotes_stream", "quotes", -1, saveQuotesToDFS{dfsQuotes}, true, 10000, 6)
```

上面的例子中，我们订阅了流数据表 quotes_stream，等待时间超过 6 秒或缓存的 quotes 记录达到 1 万条，批量写入到分布式表 dfs://stockDB/quotes 中。

#### 5.4.3. 通过 ODBC 导入数据

用户也可以通过 ODBC Plugin，将其它数据源中的数据导入到 DolphinDB 中。下面例子通过 ODBC 将 mysql 中的 quotes 表导入到 DolphinDB。
下载插件解压并拷贝 plugins/odbc 目录下所有文件到 DolphinDB server/plugins/odbc 目录下。

```
loadPlugin("plugins/odbc/odbc.cfg")
conn=odbc::connect("Driver=MySQL;Data Source = mysql-stock;server=127.0.0.1;uid=[xxx];pwd=[xxx];database=stockDB")
t=odbc::query(conn,"select * from quotes")
loadTable("dfs://stockDB", "quotes").append!(t)
```


#### 5.4.4. 通过 Programming API 导入数据

DolphinDB 提供了 Python, Java, C++, C#, R 以及 JavaScript 的编程接口。用户可在这些系统中准备好数据，然后调用 append! 函数，将数据导入到 DolphinDB 的分布式表。下面我们以 Java 为例，给出核心的代码。

```
DBConnection conn = new DBConnection();
```

连接并登录到 DolphnDB 服务器：

```
conn.connect("localhost", 8848, "admin", "123456");
```

定义函数 saveQuotes：

```
conn.run("def saveQuotes(t){ loadTable('dfs://stockDB','quotes').append!(t)}");
```

准备一个数据表，具体过程省略：

```
BasicTable quotes = ...
```

调用服务端函数 saveQuotes：

```
List<Entity> args = new ArrayList<Entity>(1);
args.add(quotes);
conn.run("saveQuotes", args);
```


## 6. 数据重分区和复制 DFS 表


### 6.1. 数据重分区

数据库的分区类型和分区方案一旦确定以后，就不能修改。如果要对数据重新分区，需要新建一个数据库，然后把原数据库中的数据导入到新数据库中。
例如，假设有一个组合分区的数据库 dfs://db1，第一层是按天分区，第二层根据股票代码范围划分为 30 个分区，创建数据库的代码如下：

```
login("admin","123456")
t=table(1:0,`timestamp`sym`qty`price,[TIMESTAMP,SYMBOL,DOUBLE,DOUBLE])
dates=2010.01.01..2020.12.31
syms="A"+string(1..500)
sym_ranges=cutPoints(syms,30)
db1=database("",VALUE,dates)
db2=database("",RANGE,sym_ranges)
db=database("dfs://db1",COMPO,[db1,db2])
db.createPartitionedTable(t,`tb1,`timestamp`sym)
```

现在要把以上数据库中的数据导入到新的数据库 dfs://db2 中。新数据库是组合分区，第一层依然是按天分区，第二层按照股票代码范围划分为 50 个分区，创建数据库的代码如下：
- 如果总数据量很小，可以直接把所有数据加载到内存表中，再把内存表中的数据保存到新的数据库中。

```
allData=select * from loadTable("dfs://db1","tb1")
tb2=loadTable("dfs://db2","tb2")
tb2.append!(allData)
```

- 但通常分布式表的数据量非常大，无法全量加载到内存中，可以用 repartitionDS 函数划分数据源，将数据划分为若干个内存能够容纳的小数据块，再通过 map-reduce 的方法将数据块分批加载到内存并保存到新的数据库中。这样做不仅可以解决内存不够的问题，而且通过并行加载提升性能。 repartitionDS 函数的语法如下：

```
repartitionDS(query, [column], [partitionType], [partitionScheme], [local=true])
```

下例按天将数据划分为多个小数据块，分批将数据写入新的数据库中：

```
def writeDataTo(dbPath, tbName, mutable tbdata){
	loadTable(dbPath,tbName).append!(tbdata)
}

datasrc=repartitionDS(<select * from loadTable("dfs://db1","tb1")>,`date,VALUE,dates)
mr(ds=datasrc, mapFunc=writeDataTo{"dfs://db2","tb2"}, parallel=true)
```

上例中 repartitionDS 函数中 local=true，表示会把重分区后的 chunk 数据都汇总到当前的协调节点，做进一步的 map-reduce 处理。如果当前节点的资源有限，可以将 local 设置为 false。
mr 函数中 parallel=true 表示小数据块会并行加载到内存和写入到数据库，只有满足以下两个条件才能将 parallel 设置为 true：（1）内存充足；（2）两个 map 子任务不会同时写入新数据库中的某个分区。否则要将 parallel 设置为 false，local 设置为 true。假如新数据库 dfs://db3 的第一层分区是按日期的范围进行分区，每个月一个分区：
由于 repartitionDS 函数按天划分的多个小数据块对应新数据库中的同一个分区，而 DolphinDB 不允许同时对一个分区进行写入，因此要 parallel 设置为 false。
repartitionDS 目前支持 VALUE 和 RANGE 两种分区方法。上例中，如果内存充足，我们也可以按月划分数据：

```
months=date(2010.01M..2021.01M)
datasrc=repartitionDS(<select * from tb1>,`date,RANGE,months) //按月划分
mr(ds=datasrc, mapFunc=writeDataTo{"dfs://db2","tb2"}, parallel=false)
```


### 6.2. 复制 DFS 表

如果只需复制 DFS 表，不改变数据的分区类型和分区方案，可以使用 sqlDS 函数划分数据源。例如，把 6.1 中表 tb1 的内容复制到同一个数据库的表 tb1_bak 中：

```
//创建tb1_bak
db=database("dfs://db1")
t=table(1:0,`timestamp`sym`qty`price,[TIMESTAMP,SYMBOL,DOUBLE,DOUBLE])
db.createPartitionedTable(t,`tb1_bak,`timestamp`sym)

//把表tb1的内容写入到表tb1_bak中
def writeDataTo(dbPath, tbName, mutable tbdata){
	loadTable(dbPath,tbName).append!(tbdata)
}

datasrc=sqlDS(<select * from tb1>)
mr(ds=datasrc, mapFunc=writeDataTo{"dfs://db1","tb1_bak"}, parallel=true)
```

当然， repartitionDS 的方法也适用于复制 DFS 表，但是使用 sqlDS 的性能更好。

```
datasrc=repartitionDS(<select * from tb1>,`date,VALUE)
mr(ds=datasrc, mapFunc=writeDataTo{"dfs://db1","tb1_bak"}, parallel=true)
```


## 7. 查询分区表注意事项

系统在执行分布式查询时，首先根据 WHERE 条件确定需要的分区，然后把查询发送到相关分区所在的节点，最后整合这些分区的结果返回给用户。
大多数分布式查询只涉及分布式表的部分分区。系统会根据关系运算符（<, <=, =, ==, >, >=, in, between）和逻辑运算符（or，and）在加载和处理数据前确定相关的分区，避免全表扫描，从而节省大量时间。下面的例子可以帮助理解 DolphinDB 如何确定相关分区。以下脚本创建了分布式表 pt，其中分区字段是 date，分区类型是 RANGE，从 1990.01.01 开始，每 2 个月为一个分区。

```
n=10000000
id=take(1..1000, n).sort()
date=1989.12.31+take(1..10000, n)
x=rand(1.0, n)
y=rand(10, n)
t=table(id, date, x, y)
db=database("dfs://rangedb1", RANGE, date(1990.01M+(0..200)*2))
pt = db.createPartitionedTable(t, `pt, `date)
pt.append!(t);

pt=db.loadTable(`pt);
```

以下类型的查询可以在加载和处理数据前缩小数据范围：

```
select * from pt where date>1990.04.01 and date<1990.06.01;
```

系统确定了两个相关分区：[1990.03.01, 1990.05.01) 和[1990.05.01, 1990.07.01)。

```
select * from pt where date between 1990.12.01:1990.12.10;
```

系统确定了一个相关分区：[1990.11.01, 1991.01.01)。

```
select count(*) from pt where date between 1990.08.01:1990.12.01 group by date;
```

系统确定了三个相关分区：[1990.07.01, 1990.09.01)、[1990.09.01, 1990.11.01) 和[1990.11.01, 1991.01.01)。

```
select * from pt where y<5 and date between 1990.08.01:1990.08.31;
```

系统确定了一个相关分区：[1990.07.01, 1990.09.01)。注意，系统忽略了 y<5 的条件。加载了相关分区后，系统会根据 y<5 的条件进一步筛选数据。
以下类型的查询不能确定相关分区，会全表扫描。对于数据量非常大的分区表，会耗费大量时间，应当尽量避免。

```
select * from pt where date+10>1990.08.01;

select * from pt where 1990.08.01<date<1990.09.01;

select * from pt where month(date)<=1990.03M;

select * from pt where y<5;

announcementDate=1990.08.01

select * from pt where date<announcementDate-3;

select * from pt where y<5 or date between 1990.08.01:1990.08.31;
```



---

> 来源: `in_memory_table.html`


# DolphinDB 内存表详解

内存表不仅可以直接用于存储数据，实现高速数据读写，而且可以缓存计算引擎的中间结果，加速计算过程。本教程主要介绍DolphinDB 内存表的分类、使用场景以及在数据操作与表结构(schema)操作上的异同。

## 内存表类别

根据不同的使用场景以及功能特点，DolphinDB 内存表可以分为以下5种：
- 常规内存表
- 键值内存表
- 索引内存表
- 流数据表
- MVCC内存表

### 常规内存表

常规内存表是 DolphinDB 中最基础的表结构，支持增删改查等操作。SQL 查询返回的结果通常存储在常规内存表中，等待进一步处理。
使用 table 函数可创建常规内存表。可根据指定的 schema（字段类型和字段名称）以及表容量（capacity）和初始行数（size）来生成内存表，亦可通过已有数据（矩阵，表，数组和元组）来生成内存表。
使用第一种生成方法的好处是可以预先为表分配内存。当表中的记录数超过容量（capacity）时，系统会自动扩充表的容量。扩充时系统首先会分配更大的内存空间（增加20%到100%不等），然后复制旧表到新的表，最后释放原来的内存。对于规模较大的表，扩容的成本会比较高。因此，如果我们可以事先大概估计表的行数，可以在创建内存表时预先为之分配一个合理的容量。如果表的初始行数为0，系统会生成空表。如果初始行数不为0，系统会生成一个指定行数的表，表中各列的值都为默认值。例如：

```
//创建一个空的常规内存表
t=table(100:0,`sym`id`val,[SYMBOL,INT,INT])

//创建一个10行的常规内存表
t=table(100:10,`sym`id`val,[SYMBOL,INT,INT])
select * from t

sym id val
--- -- ---
    0  0  
    0  0  
    0  0  
    0  0  
    0  0  
    0  0  
    0  0  
    0  0  
    0  0  
    0  0
```

table 函数也允许通过已有的数据来创建一个常规内存表。下例是通过多个数组来创建。

```
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
t=table(sym,id,val);
```

常规内存表是 DolphinDB 中应用最频繁的数据结构之一，仅次于数组。SQL 语句的查询结果，分布式查询的中间结果都存储在常规内存表中。当系统内存不足时，该表并不会自动将数据溢出到磁盘，而是 Out Of Memory 异常。因此我们进行各种查询和计算时，要注意中间结果和最终结果的 size。当某些中间结果不再需要时，请及时释放。关于常规内存表增删改查的各种用法，可以参考另一份教程 内存分区表加载和操作 ：内存表的数据处理。

### 键值内存表

键值内存表是 DolphinDB 中支持主键的内存表。指定主键（可为表中一个或多个列）的值，可以快速返回表中的对应记录。键值内存表支持增删改查等操作，但是主键值不允许更新。键值内存表通过哈希表来记录每一个键值对应的行号，因此对于基于键值的查找和更新具有非常高的效率。
使用 keyedTable 函数可创建键值内存表。该函数与 table 函数非常类似，唯一不同之处是增加了一个参数指定键值列的名称。

```
//创建空的键值内存表，主键由sym和id字段组成
t=keyedTable(`sym`id, 1:0, `sym`id`val, [SYMBOL,INT,INT])
//注意：此种方式创建键值内存表时，初始行数（size）必须为0。

//使用向量创建键值内存表，主键由sym和id字段组成
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
t=keyedTable(`sym`id,sym,id,val)
```

也可通过 keyedTable 函数将常规内存表转换为键值内存表。例如：

```
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
tmp=table(sym, id, val)
t=keyedTable(`sym`id, tmp)
```

- 数据插入和更新
向键值内存表中添加新记录时，系统会自动检查新记录的主键值。如果新记录的主键值在表中不存在，则向表中添加新的记录；如果新记录的主键值与已有记录的主键值重复，会更新表中该主键值对应的记录。请看下面的例子。
首先，向空的键值内存表中插入新记录，新记录中的主键值为 AAPL, IBM 和 GOOG。

```
t=keyedTable(`sym,1:0,`sym`datetime`price`qty,[SYMBOL,DATETIME,DOUBLE,DOUBLE]);
insert into t values(`APPL`IBM`GOOG,2018.06.08T12:30:00 2018.06.08T12:30:00 2018.06.08T12:30:00,50.3 45.6 58.0,5200 4800 7800);
t;

sym  datetime            price qty 
---- ------------------- ----- ----
APPL 2018.06.08T12:30:00 50.3  5200
IBM  2018.06.08T12:30:00 45.6  4800
GOOG 2018.06.08T12:30:00 58    7800
```

再次向表中插入一批主键值为 AAPL, IBM 和 GOOG 的新记录。

```
insert into t values(`APPL`IBM`GOOG,2018.06.08T12:30:01 2018.06.08T12:30:01 2018.06.08T12:30:01,65.8 45.2 78.6,5800 8700 4600);
t;

sym  datetime            price qty 
---- ------------------- ----- ----
APPL 2018.06.08T12:30:01 65.8  5800
IBM  2018.06.08T12:30:01 45.2  8700
GOOG 2018.06.08T12:30:01 78.6  4600
```

可以看到，表中记录条数没有增加，但是主键对应的记录已经更新。
继续往表中插入一批新记录，新记录本身包含了重复的主键值 MSFT。

```
insert into t values(`MSFT`MSFT,2018.06.08T12:30:01 2018.06.08T12:30:01,45.7 56.9,3600 4500);
t;

sym  datetime            price qty 
---- ------------------- ----- ----
APPL 2018.06.08T12:30:01 65.8  5800
IBM  2018.06.08T12:30:01 45.2  8700
GOOG 2018.06.08T12:30:01 78.6  4600
MSFT 2018.06.08T12:30:01 56.9  4500
```

- 查询数据
使用函数 sliceByKey 可以获取键值内存表中一行或多行含有参数指定内容的数据。示例如下：
首先，创建键值内存表 t，其中主键为 sym 与 side。

```
t = keyedTable(`sym`side, 10000:0, `sym`side`price`qty, [SYMBOL,CHAR,DOUBLE,INT])
insert into t values(`IBM`MSFT`GOOG, ['B','S','B'], 10.01 10.02 10.03, 10 10 20)
```

随后，使用函数 sliceByKey ，其中参数指定的内容为 sym, side 与 price。

```
sliceByKey(t, [["IBM", "GOOG"], ['B', 'S']],`sym`side`price)

sym side price
--- ---- -----
IBM B    10.01
```

注意：本例中，因为键值内存表 t 的主键为 sym 与 side，所以函数 sliceByKey 中必须含有 sym 与 side 的信息，即["IBM", "GOOG"], ['B', 'S']，若缺少任何一项都会导致 sliceByKey 函数无法获取数据。
- 应用场景
(1) 键值表对单行的更新和查询有非常高的效率，是数据缓存的理想选择。与 Redis 相比，DolphinDB 中的键值内存表兼容 SQL 的所有操作，可以完成根据键值更新和查询以外的更为复杂的计算。
(2) 键值表可作为时间序列聚合引擎的输出表，实时更新输出表的结果。具体请参考教程 在DolphinDB中计算K线 。

### 索引内存表

索引内存表是 DolphinDB 中支持键值字段的内存表，通过指定表中的一个或多个字段作为键值字段。与键值内存表类似，索引内存表也支持增删改查等操作，键值字段也不允许更新。与键值内存表的不同之处在于，在查询时，只需指定第一个键值字段，无需指定全部的键值字段即可快速返回结果。
使用 indexedTable 函数创建索引内存表。该函数与 keyedTable 非常类似。

```
//创建空的索引内存表，键值字段包括sym和id
t=indexedTable(`sym`id,1:0,`sym`id`val,[SYMBOL,INT,INT])
//注意：此种方式创建索引内存表时，初始行数（size）必须为0。

//使用向量创建索引内存表，键值字段包括sym和id
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
t=indexedTable(`sym`id,sym,id,val)
```

- 数据插入和更新
向索引内存表中添加新记录时，操作方法和系统检查方式与键值内存表一致。
- 查询数据
与键值内存表相似，索引内存表也可以通过使用函数 sliceByKey 来获取一行或多行含有参数指定内容的数据，不同的是索引内存表只需指定前 n 个键值字段的信息即可，示例如下：

```
t = indexedTable(`sym`side, 10000:0, `sym`side`price`qty, [SYMBOL,CHAR,DOUBLE,INT])
insert into t values(`IBM`MSFT`GOOG, ['B','S','B'], 10.01 10.02 10.03, 10 10 20)
```

本次 sliceByKey 函数中仅包含了键值字段 sym 的信息：

```
sliceByKey(t, ["IBM", "GOOG"],`sym`side`price)

sym side price
--- ---- -----
IBM  B    10.01
GOOG B    10.03
```

- 应用场景
键值内存表在指定所有主键的情况下才能达到查询的最佳性能。在实际应用中，不一定每次都需要查询所有键值字段的信息。若有时只需要指定部分键值字段信息，可使用索引内存表。
首先分别生成键值内存表 t1 与索引内存表 t2：

```
id1=shuffle(1..1000000)
id2=shuffle(1000001..2000000)
val=rand(100.0, 1000000)
t1=keyedTable(`id1`id2, id1, id2, val)
t2=indexedTable(`id1`id2, id1, id2, val);
```

测试键值内存表与索引内存表在查询是否包含所有键值的情况下的耗时：

```
timer(100) select * from t1 where id1=100 and id2=1000001;
Time elapsed: 1 ms

timer(100) select * from t1 where id1=100;
Time elapsed: 1034.265 ms

timer(100) select * from t2 where id1=100 and id2=1000001;
Time elapsed: 0.999 ms

timer(100) select * from t2 where id1=100;
Time elapsed: 0.998 ms
```


### 流数据表

流数据表顾名思义是为流数据设计的内存表，是流数据发布和订阅的媒介。流数据表具有天然的流表对偶性(stream-table duality)：发布一条消息等价于往流数据表中插入一条记录，订阅消息等价于将流数据表中新到达的数据推向客户端应用。对流数据的查询和计算都可以通过 SQL 语句来完成。
使用 streamTable 函数可创建流数据表。 streamTable 的用法和 table 函数完全相同。

```
//创建空的流数据表
t=streamTable(1:0,`sym`id`val,[SYMBOL,INT,INT])

//使用向量创建流数据表
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
t=streamTable(sym,id,val)
```

也可使用 streamTable 函数将常规内存表转换为流数据表。例如：

```
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
tmp=table(sym, id, val)
t=streamTable(tmp)
```

流数据表也支持创建键值列，可以通过函数 keyedStreamTable 来创建键值流表。但与键值表的设计目的不同，键值流表的目的是为了在高可用场景（多个发布端同时写入）下，避免重复消息。通常 key 就是消息的 ID。
- 数据操作特点
由于流数据具有一旦生成就不会发生变化的特点，因此流数据表不支持更新和删除记录，只支持查询和添加记录。流数据会不断增加，而内存是有限的。为解决这个矛盾，流数据表引入了持久化机制，在内存中保留最新的一部分数据，将之前的数据持久化在磁盘上。若用户订阅的数据不在内存时，直接从磁盘上读取。可使用函数 enableTableShareAndPersistence 启用持久化，细节请参考 流数据教程 。
- 应用场景
共享的流数据表在流计算中发布数据。订阅端通过 subscribeTable 函数来订阅和消费流数据。

### MVCC内存表

MVCC内存表存储了多个版本的数据，当多个用户同时对MVCC内存表进行读写操作时，互不阻塞。MVCC 内存表的数据隔离采用了快照隔离模型，用户读取到的是读操作之前已经存在的版本，即使这些数据在读取的过程中被修改或删除了，也不会影响正在进行的读操作。这种多版本的方式能够支持用户对内存表的并发访问。需要说明的是，当前的MVCC内存表实现比较简单，更新和删除数据时锁定整个表，并使用 copy-on-write 技术复制一份数据，因此对数据删除和更新操作的效率不高。在后续的版本中，我们将实现行级的 MVCC 内存表。
使用 mvccTable 函数创建 MVCC 内存表。例如：

```
//创建空的MVCC内存表
t=mvccTable(1:0,`sym`id`val,[SYMBOL,INT,INT])

//使用向量创建MVCC内存表
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
t=mvccTable(sym,id,val)
```

若要将 MVCC 内存表的数据持久化到磁盘，只需创建时指定持久化的目录和表名即可。例如，

```
t=mvccTable(1:0,`sym`id`val,[SYMBOL,INT,INT],"/home/user1/DolphinDB/mvcc","test")
```

系统重启后，可使用 loadMvccTable 函数将磁盘中的数据加载到内存中：

```
loadMvccTable("/home/user1/DolphinDB/mvcc","test")
```

可使用 mvccTable 函数将常规内存表转换为 MVCC 内存表：

```
sym=`A`B`C`D`E
id=5 4 3 2 1
val=52 64 25 48 71
tmp=table(sym, id, val)
t=mvccTable(tmp)
```

- 应用场景
当前的 MVCC 内存表适用于读多写少，并有持久化需要的场景。譬如动态的配置系统，需要持久化配置项，配置项的改动不频繁，以新增和查询操作为主，非常适合 MVCC 表。

## 共享内存表

DolphinDB 中的内存表默认只在创建内存表的会话中使用，不支持多用户多会话的并发操作，对其它会话也不可见。如果希望内存表能被其他用户使用，保证多用户并发操作的安全，必须共享内存表。所有类型的内存表均可共享。在 DolphinDB 中，使用 share 命令将内存表共享。

```
t=table(1..10 as id,rand(100,10) as val)
share t as st
//或者share(t,`st)
```

以上代码将表t共享为表 st。
使用 undef 函数可以删除共享表：

```
undef(`st,SHARED)
```


### 保证对所有会话可见

内存表仅在当前会话可见，在其它会话中不可见。共享之后，其它会话可以通过访问共享变量来访问内存表。例如，在当前会话中把表 t 共享为表 st：

```
t=table(1..10 as id,rand(100,10) as val)
share t as st
```

可以在其它会话中访问变量 st。例如，在另外一个会话中向共享表 st 插入一条数据：

```
insert into st values(11,200)
select * from st

id val
-- ---
1  1  
2  53 
3  13 
4  40 
5  61 
6  92 
7  36 
8  33 
9  46 
10 26 
11 200
```

切换到原来的会话，可以发现，表 t 中也增加了一条记录。

```
select * from t

id val
-- ---
1  1  
2  53 
3  13 
4  40 
5  61 
6  92 
7  36 
8  33 
9  46 
10 26 
11 200
```


### 保证线程安全

在多线程的情况下，内存表中的数据很容易被破坏。共享则提供了一种保护机制，能够保证数据安全，但同时也会影响系统的性能。
常规内存表、流数据表和 MVCC 内存表都支持多版本模型，常规内存表和流数据表允许一写多读， MVCC 内存表允许多读多写。读写互不阻塞，写的时候可以读，读的时候可以写。读数据时不上锁，允许多个线程同时读取数据，读数据时采用快照隔离(snapshot isolation)。常规内存表和流数据表写数据时必须加锁，同时只允许一个线程修改内存表；MVCC 内存表同时允许多个线程修改内存表。写操作包括添加、删除或更新。添加记录一律在内存表的末尾追加，无论内存使用还是CPU使用均非常高效。常规内存表和 MVCC 内存表支持更新和删除，且采用了 copy-on-write 技术，也就是先复制一份数据（构成一个新的版本），然后在新版本上进行删除和修改。由此可见删除和更新操作无论内存和CPU消耗都比较高。当删除和更新操作很频繁，读操作又比较耗时（不能快速释放旧的版本），容易导致 OOM 异常。
键值内存表与索引内存表写入时需维护内部索引，读取时也需要根据索引获取数据。因此这两种内存表的共享采用了不同的方法，无论读写都必须加锁。写线程和读线程，多个写线程之间，多个读线程之间都是互斥的。并发使用时，对键值内存表与索引内存表应尽量避免耗时的查询或计算，否则会使其它线程长时间处于等待状态。

## 分区内存表

当内存表数据量较大时，可对内存表进行分区，以充分利用多核进行并行计算。分区后一个内存表由多个子表（tablet）构成，内存表不使用全局锁，锁由每个子表独立管理，这样可以显著增加读写并发能力。DolphinDB 支持对内存表进行值分区、范围分区、哈希分区和列表分区，但不支持组合分区。使用函数 createPartitionedTable 创建内存分区表。
- 创建分区常规内存表

```
t=table(1:0,`id`val,[INT,INT])
db=database("",RANGE,0 101 201 301)
pt=db.createPartitionedTable(t,`pt,`id)
```

- 创建分区键值内存表

```
kt=keyedTable(1:0,`id`val,[INT,INT])
db=database("",RANGE,0 101 201 301)
pkt=db.createPartitionedTable(kt,`pkt,`id)
```

类似的，可创建分区索引内存表。
- 创建分区流数据表
创建分区流数据表时，需要传入多个流数据表作为模板，每个流数据表对应一个分区。写入数据时，直接往这些流表中写入；而查询数据时，需要查询分区表。

```
st1=streamTable(1:0,`id`val,[INT,INT])
st2=streamTable(1:0,`id`val,[INT,INT])
st3=streamTable(1:0,`id`val,[INT,INT])
db=database("",RANGE,1 101 201 301)
pst=db.createPartitionedTable([st1,st2,st3],`pst,`id)

st1.append!(table(1..100 as id,rand(100,100) as val))
st2.append!(table(101..200 as id,rand(100,100) as val))
st3.append!(table(201..300 as id,rand(100,100) as val))

select * from pst
```

- 创建分区 MVCC 内存表
与创建分区流数据表一样，创建分区 MVCC 内存表，需要传入多个MVCC内存表作为模板。每个表对应一个分区。写入数据时，直接往这些表中写入；而查询数据时，需要查询分区表。

```
mt1=mvccTable(1:0,`id`val,[INT,INT])
mt2=mvccTable(1:0,`id`val,[INT,INT])
mt3=mvccTable(1:0,`id`val,[INT,INT])
db=database("",RANGE,1 101 201 301)
pmt=db.createPartitionedTable([mt1,mt2,mt3],`pst,`id)

mt1.append!(table(1..100 as id,rand(100,100) as val))
mt2.append!(table(101..200 as id,rand(100,100) as val))
mt3.append!(table(201..300 as id,rand(100,100) as val))

select * from pmt
```

由于分区内存表不使用全局锁，创建以后不能再动态增删子表。

### 增加查询的并发性

分区表增加查询的并发性有三层含义：（1）键值表在查询时也需要加锁，分区表由子表独立管理锁，相当于把锁的粒度变细了，因此可以增加读的并发性；（2）批量计算时分区表可以并行处理每个子表；（3）如果SQL查询的过滤指定了分区字段，那么可以缩小分区范围，避免全表扫描。
以键值内存表为例，下例对比在分区和不分区的情况下，并发查询的性能。首先，创建包含500万行数据的模拟数据集。

```
n=5000000
id=shuffle(1..n)
qty=rand(1000,n)
price=rand(1000.0,n)
kt=keyedTable(`id,id,qty,price)
share kt as skt

id_range=cutPoints(1..n,20)
db=database("",RANGE,id_range)
pkt=db.createPartitionedTable(kt,`pkt,`id).append!(kt)
share pkt as spkt
```

在另外一台服务器上模拟10个客户端同时查询键值内存表。每个客户端查询10万次，每次查询一条数据，统计每个客户端查询10万次的总耗时。

```
def queryKeyedTable(tableName,id){
	for(i in id){
		select * from objByName(tableName) where id=i
	}
}
conn=xdb("192.168.1.135",18102,"admin","123456")
n=5000000

jobid1=array(STRING,0)
for(i in 1..10){
	rid=rand(1..n,100000)
	s=conn(submitJob,"evalQueryUnPartitionTimer"+string(i),"",evalTimer,queryKeyedTable{`skt,rid})
	jobid1.append!(s)
}
time1=array(DOUBLE,0)
for(j in jobid1){
	time1.append!(conn(getJobReturn,j,true))
}

jobid2=array(STRING,0)
for(i in 1..10){
	rid=rand(1..n,100000)
	s=conn(submitJob,"evalQueryPartitionTimer"+string(i),"",evalTimer,queryKeyedTable{`spkt,rid})
	jobid2.append!(s)
}
time2=array(DOUBLE,0)
for(j in jobid2){
	time2.append!(conn(getJobReturn,j,true))
}
```

time1 是10个客户端查询未分区键值内存表的耗时，time2 是10个客户端查询分区键值内存表的耗时，单位是毫秒。

```
time1
[6719.266848,7160.349678,7271.465094,7346.452625,7371.821485,7363.87979,7357.024299,7332.747157,7298.920972,7255.876976]

time2
[2382.154581,2456.586709,2560.380315,2577.602019,2599.724927,2611.944367,2590.131679,2587.706832,2564.305815,2498.027042]
```

可以看到，每个客户端查询分区键值内存表的耗时要低于查询未分区内存表的耗时。
查询未分区的内存表，可以保证快照隔离。但查询一个分区内存表，不再保证快照隔离。如前面所说分区内存表的读写不使用全局锁，一个线程在查询时，可能另一个线程正在写入而且涉及多个子表，从而可能读到一部分写入的数据。

### 增加写入的并发性

以分区的常规内存表为例，我们可以同时向不同的分区写入数据。

```
t=table(1:0,`id`val,[INT,INT])
db=database("",RANGE,1 101 201 301)
pt=db.createPartitionedTable(t,`pt,`id)

def writeData(mutable t,id,batchSize,n){
	for(i in 1..n){
		idv=take(id,batchSize)
		valv=rand(100,batchSize)
		tmp=table(idv,valv)
		t.append!(tmp)
	}
}

job1=submitJob("write1","",writeData,pt,1..100,1000,1000)
job2=submitJob("write2","",writeData,pt,101..200,1000,1000)
job3=submitJob("write3","",writeData,pt,201..300,1000,1000)
```

上面的代码中，同时有3个线程对 pt 的3个不同的分区进行写入。需要注意的是，要避免同时对相同分区进行写入。例如，下面的代码可能会导致系统异常退出。

```
job1=submitJob("write1","",writeData,pt,1..300,1000,1000)
job2=submitJob("write2","",writeData,pt,1..300,1000,1000)
```

上面的代码定义了两个写入线程，并且写入的分区相同，这样会造成线程不安全，可能会导致系统崩溃、数据错误等后果。为了保证每个分区数据的安全性和一致性，可将分区内存表共享，这样即可定义多个线程同时对相同分区写入。

```
share pt as spt
job1=submitJob("write1","",writeData,spt,1..300,1000,1000)
job2=submitJob("write2","",writeData,spt,1..300,1000,1000)
```


## 数据操作比较


### 增删改查

下表总结了不同类型内存表在共享/分区的情况下支持的增删改查操作。
|  |  | 增加 | 删除 | 更新 | 查询 |
| --- | --- | --- | --- | --- | --- |
| 常规内存表 | 未共享未分区 | √ | √ | √ | 全表扫描 |
| 共享 | 全表扫描 |
| 分区 | 分区内扫描 |
| 共享分区 | 分区内扫描 |
| 键值/索引内存表 | 未共享未分区 | √ | √ | √ | 哈希/排序树 |
| 共享 | 哈希/排序树 |
| 分区 | 分区内哈希 |
| 共享分区 | 分区内哈希 |
| 流数据表 | 未共享未分区 | √ | × | × | 全表扫描 |
| 共享 | 全表扫描 |
| 分区 | 分区内扫描 |
| 共享分区 | 分区内扫描 |
| MVCC内存表 | 未共享未分区 | √ | √ | √ | 全表扫描 |
| 共享 | 全表扫描 |
| 分区 | 分区内扫描 |
| 共享分区 | 分区内扫描 |
- 常规内存表、键值内存表、索引内存表、MVCC内存表都支持增删改查操作。流数据表仅支持增加数据和查询，不支持删除和更新操作。
- 对于键值内存表与索引内存表，如果查询的过滤条件中包含主键或键值字段，查询的性能会得到明显提升。
- 对于分区内存表，如果查询的过滤条件中包含分区列，系统能够缩小需要扫描的分区范围，从而提升查询的性能。

### 并发性

在没有写入的情况下，所有内存表都允许多个线程同时查询。在有写入的情况下，各类内存表的并发性有所差异。下表总结了各类内存表在共享/分区的情况下支持的并发读写情况。
|  |  | 一写一读或一写多读 | 多写多读（写入相同分区） | 多写多读（写入不同分区） |
| --- | --- | --- | --- | --- |
| 常规内存表 | 未共享未分区 | × | × | / |
| 共享 | √ | √ |
| 分区 | × | × | × |
| 共享分区 | √ | √ | √ |
| 键值/索引内存表 | 未共享未分区 | √ | × | / |
| 共享 | √ | √ |
| 分区 | √ | × | √ |
| 共享分区 | √ | √ | √ |
| 流数据表 | 未共享未分区 | √ | × | / |
| 共享 | √ | √ |
| 分区 | √ | × | √ |
| 共享分区 | √ | √ | √ |
| MVCC内存表 | 未共享未分区 | √ | √ | / |
| 共享 | √ | √ |
| 分区 | √ | √ | √ |
| 共享分区 | √ | √ | √ |
- 共享表允许并发读写。
- 对于没有共享的分区表，不允许多线程对相同分区同时写入。

### 持久化

- 常规内存表、键值内存表以及索引内存表均不支持数据持久化。一旦节点重启，内存中的数据将全部丢失。
- MVCC内存表支持持久化。在创建 MVCC 内存表时，可以指定持久化的路径。例如：

```
t=mvccTable(1:0,`id`val,[INT,INT],"/home/user/DolphinDB/mvccTable")
t.append!(table(1..10 as id,rand(100,10) as val))
```

系统重启后，可使用 loadMvccTable 函数将磁盘中的数据加载到内存中。例如：

```
t=loadMvccTable("/home/user/DolphinDB/mvccTable","t")
```

- 要对流数据表进行持久化，首先要配置流数据持久化的目录 persistenceDir，再使用 enableTableShareAndPersistence 函数将流数据表共享，并持久化到磁盘上。请注意，该函数只能用于空的流数据表。例如，将流数据表t共享并持久化到磁盘上：

```
t=streamTable(1:0,`id`val,[INT,DOUBLE])
enableTableShareAndPersistence(t,`st)
```

流数据表启用了持久化后，内存中仍然会保留流数据表中部分最新的记录。默认情况下，内存会保留最新的10万条记录。可以根据需要调整这个值。
流数据表持久化可以设定采用异步/同步、压缩/不压缩的方式。通常情况下，异步模式能够实现更高的吞吐量。
系统重启后，执行 enableTableShareAndPersistence 函数，会将持久化的数据加载到内存。

## 表结构操作比较

内存表的结构操作包括新增列、删除列、修改列（内容和数据类型）以及调整列的顺序。下表总结了5种类型内存表在共享/分区的情况下支持的结构操作。
|  |  | 通过addColumn函数新增列 | 通过update语句新增列 | 删除列 | 修改列(内容和数据类型) | 调整列的顺序 |
| --- | --- | --- | --- | --- | --- | --- |
| 常规内存表 | 未共享未分区 | √ | √ | √ | √ | √ |
| 共享 | √ | √ | √ | √ | √ |
| 分区 | × | √ | √ | × | × |
| 共享分区 | × | × | × | × | × |
| 键值/索引内存表 | 未共享未分区 | √ | √ | √ | √ | √ |
| 共享 | √ | √ | √ | √ | √ |
| 分区 | × | √ | √ | × | × |
| 共享分区 | × | × | × | × | × |
| 流数据表 | 未共享未分区 | √ | × | × | × | × |
| 共享 |
| 分区 | × |
| 共享分区 |
| MVCC 内存表 | 未分区未共享 | × | √ | × | × | × |
| 共享 | √ |
| 分区 | √ |
| 共享分区 | × |
- 分区表以及MVCC内存表不能通过 addColumn 命令新增列。
- 分区表可以通过 update 语句来新增列，但是流数据表不能通过 update 语句来新增列。

## 小结

DolphinDB 支持5种类型内存表，还支持共享和分区，能够满足内存计算和流计算的各种需求。


---

> 来源: `data_import_details.html`


# 数据导入最容易忽略的十个细节

数据导入是使用 DolphinDB 的重要一环。无论是从磁盘文件（如 csv 文件、txt 文件等）导入数据，还是使用插件从其他来源导入，如果忽略了一些操作细节，会导致导入失败或导入结果不符合预期。
本文将介绍使用 DolphinDB 进行数据导入时，最容易忽略的 10 个细节，涉及了数据格式、数据类型、导入速率、数据预处理、连接失败、分区冲突等方面，并给出了正确的解决方案，一起来看看吧。

## 1. 表头包含数字时的文件导入技巧

loadText 和 ploadText 是使用 DolphinDB 导入文本文件时最常使用的函数。但由于 DolphinDB 中列名必须以中文或英文字母开头，对于以数字开头的表头， loadText 和 ploadText 函数的处理规则如下：
- 不指定 containHeader 时，导入文本文件时将以字符串格式读取第一行数据，并根据该数据解析列名。但如果文件第一行记录中某列记录以数字开头，那么加载文件时系统会使用 col0 , col1 , … 等作为列名。
- 指定 containHeader = true 时，系统将第一行数据视为标题行，并解析出列名。如果文件第一行记录中某列记录以数字开头，那么加载文件时系统会使用 “c” 加列名作为列名。
如果忽略该细节，可能导致导入结果的列名不符合预期。
如导入下面的 colName.csv ，有列名以数字开头，原始结构如下：
| id | name | totalSale | 2023Q1 | 2023Q2 | 2023Q3 |
| --- | --- | --- | --- | --- | --- |
| 1 | shirt | 8000 | 2000 | 3000 | 3000 |
| 2 | skirt | 10000 | 2000 | 5000 | 3000 |
| 3 | hat | 2000 | 1000 | 300 | 700 |

### 1.1. 错误操作

不指定 containHeader 时，系统将会使用 col0 , col1 , … 等作为列名。
执行 loadText("/xxx/colName.csv") ，导入结果如下：
需要注意的是，由于 col0 和 col1 可以被视作一组具有离散值的变量，符合枚举类型的特征，被默认解析为 SYMBOL 类型；而 col2 - col5 这几列数据都是整数形式，被默认解析为 INT 类型，因此 col2 原本的列名 totalSale 无法写入该列，为空； col3 - col5 原本的列名只保留了开头可以解析为数字的 2023 部分。

### 1.2. 正确操作

指定 containHeader = true 时，系统会使用 “c” 加列名作为数字开头列的列名。
执行 loadText("/xxx/colName.csv", containHeader = true) ，导入结果如下：
对于数字开头的列名，指定 containHeader = true，可以保证数据完整性，仅在列名前添加字符 “c”，是目前最佳的解决方案。因此，当使用 loadText 函数导入的文本文件表头有列名以数字开头时，应该指定 containHeader = true。

## 2. 自动解析数据类型的导入技巧

loadText 、 ploadText 和 loadTextEx 函数都提供了 schema 参数，用于传入一个表对象，以指定各字段的数据类型，并照此类型来加载数据。使用 loadText 、 ploadText 和 loadTextEx 函数时，如果用户不指定 schema 参数，会对数据文件进行随机抽样，并基于样本决定每列的数据类型。
由于自动解析数据类型的方法是对数据文件进行随机抽样，因此不一定每次都能准确决定各列的数据类型。为此，DolphinDB 提供了 extractTextSchema 函数，能够查看 DolphinDB 对数据文件进行自动解析数据类型的结果。在导入数据前，建议使用 extractTextSchema 确认对数据类型的自动解析结果；如果自动解析结果不符合预期，可以修改对应类型，并将修改后的表作为 loadText 、 ploadText 和 loadTextEx 的 schema 参数，再进行数据导入。
如导入下面的 type.csv ，原始结构如下：
| id | ticker | price |
| --- | --- | --- |
| 1 | 300001 | 25.80 |
| 2 | 300002 | 6.85 |
| 3 | 300003 | 7.19 |

### 2.1. 错误操作

执行 loadText("/xxx/type.csv") ，导入结果如下：
执行 extractTextSchema("/xxx/type.csv") ，可以看到， ticker 列被默认解析为 INT 类型，不符合预期：

### 2.2. 正确操作

可以使用 extractTextSchema 函数先得到自动解析结果，再将 ticker 列的类型指定为 SYMBOL 类型，并用修改后的结果作为 loadText 函数的 schema 参数值进行导入，即可得到预期结果：

```
schema = extractTextSchema("/xxx/type.csv")
update schema set type =`SYMBOL where name = "ticker"
loadText("/xxx/type.csv", schema=schema)
```


## 3. 日期与时间格式数据的手动导入方法

使用 loadText 函数导入符合日期、时间格式的数据时，如果用户不指定 schema 参数，会将符合日期、时间格式的数据优先解析为对应类型。规则如下：
- 当加载的数据文件中包含了表达日期、时间的数据时，满足分隔符要求的这部分数据（日期数据分隔符包含 ”-”、”/” 和 ”.”，时间数据分隔符为 ”:”）会解析为相应的类型。例如，”12:34:56” 解析为 SECOND 类型；”23.04.10” 解析为 DATE 类型。
- 对于不包含分隔符的数据，形如 ”yyMMdd” 的数据同时满足 0<=yy<=99，0<=MM<=12，1<=dd<=31 的条件时，会被优先解析成 DATE 类型；形如 ”yyyyMMdd” 的数据同时满足 1900<=yyyy<=2100，0<=MM<=12，1<=dd<=31 的条件时，会被优先解析成 DATE 类型。
如导入下面的 notdate.csv ，原始结构如下：
| id | ticker | price |
| --- | --- | --- |
| 1 | 111011 | 133.950 |
| 2 | 111012 | 125.145 |
| 3 | 111013 | 113.240 |

### 3.1. 错误操作

直接使用 loadText 函数导入 DolphinDB，由于 ticker 列是形如 "yyMMdd" 的数据，且同时满足 0<=yy<=99，0<=MM<=12，1<=dd<=31 的条件，会被优先解析成 DATE 类型。
执行 loadText("/xxx/notdate.csv") ，导入结果如下：

### 3.2. 正确操作

导入此类符合日期、时间格式的数据时，无需使用 extractTextSchema 函数查看自动解析结果，可提前在 schema 参数中将 ticker 列指定解析为 SYMBOL 类型，再进行导入，即可得到预期结果。

```
schema = table(`id`ticker`price as name, `INT`SYMBOL`DOUBLE as type)
loadText("/xxx/notdate.csv", schema = schema)
```


## 4. 提升数据导入效率的 loadTextEx 函数使用建议

在一些实际应用场景中，用户导入数据文件的下一步，都是将其写入分布式数据库进行存储，而不需要在内存中进行额外操作。DolphinDB 为此提供了 loadTextEx 函数，能够直接把数据文件入库，不需要把数据加载到内存表，既简化了步骤，又节省了时间。
如导入下面的 data.csv ，原始结构如下：
| ID | date | vol |
| --- | --- | --- |
| 23 | 2023.08.08 | 34.461863990873098 |
| 27 | 2023.08.08 | 4.043174418620766 |
| 36 | 2023.08.08 | 5.356599518563599 |
| 98 | 2023.08.09 | 5.630887264851481 |
| … | … | … |

### 4.1. 初级操作

如果先使用 loadText 函数把该文件加载到内存，再写入数据库，耗时 771.618ms。

```
timer{
    t = loadText("/xxx/data.csv")
    loadTable("dfs://test_load","data").append!(t)
}

>> Time elapsed: 771.618ms
```


### 4.2. 进阶操作

直接使用 loadTextEx 函数将该文件入库，耗时 384.097ms。

```
timer{
    loadTextEx(database("dfs://test_load"), "data", "date", "/xxx/data.csv", sortColumns=`id)
}

>> Time elapsed: 384.097ms
```

数据量越大，两种方式的耗时差异越明显。

## 5. 在 loadTextEx 中使用 transform 参数预处理数据

在一些实际应用场景中，用户既希望能够使用 loadTextEx 函数直接把数据入库，又希望能够对数据做一些简单处理。针对这种需求，可以在 loadTextEx 中使用 transform 参数，对数据进行处理后再入库。这种做法无需把数据加载到内存来处理，简化了操作步骤。
例如，导入下面的 dataWithNULL.csv 时， vol 列有空值，需要先将 vol 列的空值填充为 0，再入库：
| ID | date | vol |
| --- | --- | --- |
| 52 | 2023.08.08 | 5.08143 |
| 77 | 2023.08.09 |  |
| 35 | 2023.08.08 | 0.22431 |
| 99 | 2023.08.09 |  |
| … | … | … |

### 5.1. 初级操作

如果先使用 loadText 函数把该文件加载到内存，使用 nullFill! 去除空值，再写入数据库，耗时 802.23ms。

```
timer{
    t = loadText("/xxx/dataWithNULL.csv")
    t.nullFill!(0.0)
    loadTable("dfs://test_load","data").append!(t)
}

>> Time elapsed: 802.23ms
```


### 5.2. 进阶操作

使用 loadTextEx 函数配合 transform 参数将该文件入库，耗时 385.086ms。

```
timer{
    loadTextEx(database("dfs://test_load"), "data", "date", "/xxx/dataWithNULL.csv", transform = nullFill!{, 0.0}, sortColumns=`id)
}

>> Time elapsed: 385.086 ms
```

可以看到， transform 参数使数据预处理变得非常便捷，无需在内存中额外处理数据，可以一步入库。

## 6. 避免将长整型时间戳数据以 TIMESTAMP 类型导入

在许多数据中，常常用长整型（LONG 类型）来表示 Unix 毫秒时间戳，即从 1970 年 1 月 1 日（UTC/GMT 的午夜）开始所经过的毫秒数。在导入这样的数据时，如果不预先指定导入类型，将会把这种形式的时间戳直接作为长整型导入，显示的是长整型数据；而如果指定这种形式的时间戳为 TIMESTAMP 类型，将其导入后并不会自动转换为对应的 TIMESTAMP，而会返回空值。因此，这两种做法的结果都不符合预期。
正确的做法是，先把原始数据以 LONG 类型导入，再使用 timestamp 函数手动转换为对应的 TIMESTAMP。
如导入下面的 time.csv ，原始结构如下：
| timestamp | ticker | price |
| --- | --- | --- |
| 1701199585108 | SH2575 | 9.05991137959063 |
| 1701101960267 | SH1869 | 9.667245978489518 |
| 1701292328832 | SH1228 | 19.817104414105415 |
| 1701186220641 | SH2471 | 3.389011020772159 |

### 6.1. 错误操作

如果指定把 timestamp 列作为 TIMESTAMP 类型导入，该列导入结果将会全为空值：

```
schema = extractTextSchema("/xxx/time.csv")
update schema set type = "TIMESTAMP" where name = "timestamp"
loadText("/xxx/time.csv", schema=schema)
```


### 6.2. 正确操作

正确做法应该是把 timestamp 列以 LONG 类型直接导入，然后使用 replaceColumn! 和 timestamp 函数，将其手动转换为 TIMESTAMP 类型，即可得到预期结果：

```
t = loadText("/xxx/time.csv")
replaceColumn!(t, `timestamp, timestamp(exec timestamp from t))
t
```


## 7. 长整型转换为 TIMESTAMP 类型时的时区注意事项

有的数据库在存储时间数据时，会将其转换为全球统一的 Unix 时间戳，并单独存储时区信息（即 UTC 偏移量）。 而 DolphinDB 将时间转换为本地时间戳直接存储，不会单独存储时区信息，具体可参考： 时区处理 。
因此，如果原始数据文件中的长整型是带时区信息的长整型，即已经加减了 UTC 偏移量的数据，导入 DolphinDB 进行转换时会被视为零时区数据处理，可能导致结果与预期不符。
如导入下面的 localtime.csv ，原始结构如下：
| timestamp | ticker | price |
| --- | --- | --- |
| 1701331200000 | SH3149 | 8.676103590987622 |
| 1701331200000 | SH0803 | 12.16052254475653 |
| 1701331200000 | SH2533 | 12.076009283773601 |
| 1701331200000 | SH3419 | 0.239130933769047 |
其中， timestamp 列应为由北京时间 2023.11.30T16:00:00.000 转换的长整型：

### 7.1. 错误操作

直接把 timestamp 列以 LONG 类型直接导入，然后将其手动转换为 TIMESTAMP 类型后，得到的是零时区时间，和北京时间相差 8 个小时：

```
t = loadText("/xxx/localtime.csv")
replaceColumn!(t, `timestamp, timestamp(exec timestamp from t))
t
```


### 7.2. 正确操作

对于长整型转换为 TIMESTAMP 默认为零时区的现象，可以在转换类型时，利用 DolphinDB 内置的时区转换函数 localtime ，把零时区时间转换成本地时间（本文测试的本地时间为东八区时间）：

```
t = loadText("/xxx/localtime.csv")
replaceColumn!(t, `timestamp, localtime(timestamp(exec timestamp from t)))
t
```

也可以利用 DolphinDB 内置时区转换函数 convertTZ ，完成两个指定时区之间的转换：

```
t = loadText("/xxx/localtime.csv")
replaceColumn!(t, `timestamp, convertTZ(timestamp(exec timestamp from t), "UTC", "Asia/Shanghai"))
t
```


## 8. 使用标准格式的连接字符串连接 ODBC

通过 ODBC 插件可以连接其它数据源，将其他数据源的数据导入到 DolphinDB。在连接其他数据源时，有可能遇到报错： FATAL: password authentication failed for user "xxx" ，但用户名和密码均正确，导致排查问题来源时没有头绪。许多情况下，这是因为使用的 ODBC 连接字符串有问题。有关连接字符串的标准格式，请参阅 连接字符串参考 。

### 8.1. 错误操作

如下面的场景，使用 DSN 连接字符串连接 PostgreSQL，运行后出现报错：

```
odbcConfig = "DSN=PIE_PGSQL;Server="+ip+";port="+port+";DATABASE="+db+";Uid="+userName+";Password="+password+";"
conn = odbc::connect(odbcConfig)

>> FATAL: password authentication failed for user "pie_dev"
```

Linux 版本的 ODBC 插件基于 unixODBC 开发，isql 是 unixODBC 提供的基本工具，能够快速排查 ODBC 相关问题。使用相同的 DSN 连接字符串运行 isql，出现了相同的报错：

```
[28P01][unixODBC]FATAL: password authentication failed for user "pie_dev"
[ISQL]ERROR: Could not SQLDriverConnect
```

说明该问题与 DolphinDB 的 ODBC 插件无关，可能是连接字符串有问题。

### 8.2. 正确操作

查阅 连接字符串参考 ，发现 PostgreSQL 的 ODBC 标准连接字符串如下：
按照标准连接字符串形式，参考如下方式修改连接字符串：

```
odbcConfig = "Driver={PostgreSQL};Server="+ip+";port="+port+";DATABASE="+db+";Uid="+userName+";Password="+password+";"
conn = odbc::connect(odbcConfig)
```


## 9. 同磁盘下数据解压和导入的并行处理注意事项

许多时候，用户的原始数据文件是压缩包形式的，需要先解压再进行导入。如果用户采取的是一边解压一边导入的方式，且原始数据所在磁盘与解压后存储 DolphinDB 数据库的磁盘是同一块时，会降低导入的速率。如果忽略这个细节，可能导致数据导入的耗时额外增加。
如导入下面的 mdl_6_28_0.csv ，大小为 4.3 GB：

### 9.1. 错误操作

在该 DolphinDB 数据存储的磁盘上进行一个 .zip 文件的解压任务，同时使用 loadTextEx 函数将另一个已解压的 csv 文件入库，耗时约 1m41s：

```
timer loadTextEx(database("dfs://test_load"), "data", "SecurityID", "/xxx/mdl_6_28_0.csv", sortColumns=`SecurityID`UpdateTime)

>> Time elapsed: 101156.78ms
```

在导入过程中，观察磁盘读写速率如下：
可以看到，当在同一块磁盘同时解压数据和导入数据时，数据导入的速率降低了许多。

### 9.2. 正确操作

磁盘空闲，没有解压任务在执行时，直接使用 loadTextEx 函数入库相同的文件，耗时约 46s：
在导入过程中，观察 DolphinDB 磁盘读写速率如下：

## 10. 避免并行写入同一分区时的冲突解决方案

在导入数据并写入分布式数据库时，有可能会存在并行写入同一分区的情况。在这种场景下，根据创建数据库时设置的 atomic 参数的不同，可能出现以下不同的情况：
- atomic 参数设置为 ’TRANS’，写入事务的原子性层级为事务，即一个事务写入多个分区时，若某个分区被其他写入事务锁定而出现写入冲突，则该事务的写入全部失败。因此，该设置下，不允许并发写入同一个分区。
- atomic 参数设置为 ’CHUNK’，写入事务的原子性层级为分区。若一个事务写入多个分区时，某分区被其它写入事务锁定而出现冲突，系统会完成其他分区的写入，同时对之前发生冲突的分区不断尝试写入，尝试数分钟后仍冲突才放弃。此设置下，允许并发写入同一个分区，但由于不能完全保证事务的原子性，可能出现部分分区写入成功而部分分区写入失败的情况。同时由于采用了重试机制，写入速度可能较慢。
当发生分区冲突导致写入失败时，将会抛出错误代码 S00002，错误信息为： <ChunkInTransaction>filepath has been owned by transaction ，表示某个分区已经被一个事务锁定，新的事务无法再次锁定相同分区。如果忽略此细节，可能导致数据导入任务失败。
下面的 dfs://test_month 数据库，按月分区，且 atomic 参数为 ’TRANS’：

### 10.1. 错误操作

并行导入下面的 csv 文件入库，他们入库的目标分区都为 2023.08M：
使用 submitJob 提交导入任务：

```
filePath = "/xxx/dailyData"
for (i in files(filePath).filename){
    submitJob("loadCSVTest", "load daily files to monthly db", loadTextEx, database("dfs://test_month"), "data", "date", filePath+i)
}
```

查看任务完成情况，可以看到，只有一个任务导入成功，别的任务都因为分区冲突而失败，抛出错误代码 S00002：

### 10.2. 正确操作

对于这种情况，我们可以串行导入文本文件，避免并行写入同一分区时的冲突：

```
def loadDaily(filePath){
	for (i in files(filePath).filename){
		loadTextEx(database("dfs://test_month"), "data",`date,filePath+i,sortColumns=`ID)
	}
}

path = "/xxx/dailyData/"
submitJob("loadCSVTest","load daily files to monthly db",loadDaily, path)
```

查看任务完成情况，可以看到，数据能够导入成功，没有报错：

## 11. 总结

本文介绍了使用 DolphinDB 进行数据导入时，最容易忽略的 10 个细节，涵盖了数据格式、数据类型、导入速率、数据预处理、连接失败、分区冲突等多个方面。忽略了这些细节，可能导致数据导入失败或导入结果不符合预期。用户了解了这些细节，并合理进行数据导入，可以极大提升数据导入的速度和质量。
