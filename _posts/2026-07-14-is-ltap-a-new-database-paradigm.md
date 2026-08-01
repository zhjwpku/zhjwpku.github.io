---
layout: post
title: "LTAP: 新的数据库范式，还是云原生 PostgreSQL 与 Lakehouse 的重新组合？"
date: 2026-07-14 00:00:00 +0800
tags:
- database
- postgres
- lakehouse
- ltap
- databricks
---

2026 年 6 月，Databricks 发布了 LTAP（Lake Transactional/Analytical Processing），宣称可以让事务与分析引擎直接访问湖上同一份数据，不再需要 CDC、ETL 或分析副本。随后发布的[技术博客](https://www.databricks.com/blog/lakebase-ltap-rethinking-database-storage)进一步解释了 Lakebase 如何从类似 Neon 的存算分离 PostgreSQL 架构演进到 LTAP。

那么，LTAP 是否代表一个新的数据库范式？

我的结论是: **LTAP 值得被视为一种新的数据库架构模式，但现在把它称为已经成立的新数据库范式还为时过早。**

它使用的 WAL、存算分离、对象存储、列式格式、MVCC 和多引擎计算都不是新技术。LTAP 真正重要的变化是重新定义了哪一种物理表示才是数据库的持久化真相:

- 在传统 PostgreSQL 和 Neon 式架构中，PostgreSQL page 是主要的数据表示，分析系统需要另一份列式数据
- 在 LTAP 中，Delta/Iceberg 管理的 Parquet 列式数据成为持久化的 canonical copy
- PostgreSQL heap page、索引以及相关的行式结构退化为可以重建的 serving/cache representation

这不是简单地把 PostgreSQL 文件放到对象存储，也不只是把 CDC 藏到平台内部。它试图让事务引擎和分析引擎真正共享一个持久化表示。但是，目前公开材料还没有充分解释这种表示如何完整承载 PostgreSQL 的物理语义、事务可见性和冷读性能，也没有给出足以验证整个架构的 benchmark。

本文基于 Databricks 的[发布公告](https://www.databricks.com/company/newsroom/press-releases/databricks-launches-ltap-first-lake-transactionalanalytical)、[LTAP 技术博客](https://www.databricks.com/blog/lakebase-ltap-rethinking-database-storage)、Neon 的[架构文档](https://neon.com/docs/introduction/architecture-overview)以及 PostgreSQL 的公开文档，尝试区分已经公开的事实、可以合理推导的设计，以及仍然没有答案的问题。

## 从 HTAP 到 LTAP

事务和分析长期以来存在不同的物理访问需求:

- OLTP 偏好点查、小范围更新、低延迟和高并发，行存更合适
- OLAP 偏好扫描少数列、向量化执行、大规模聚合，列存更合适

传统方案通常有三类。

第一类是独立的 OLTP 和 OLAP 系统，通过 CDC 或 ETL 同步。它能实现很好的资源隔离，但代价是两份数据、同步延迟、复杂的数据管道和两套治理边界。

第二类是 HTAP。它尝试在一个数据库系统中同时服务事务与分析。有的系统使用统一存储，有的系统在内部维护行存和列存两种表示。它减少了外部数据管道，但两种 workload 仍然可能争抢资源，系统内部也仍需维护不同物理表示。

第三类是 Neon、Aurora 等云原生数据库采用的存算分离。以 Neon 为例，PostgreSQL compute 生成 WAL，Safekeeper 负责近期 WAL 的持久化，PageServer 消费 WAL、保存不同 LSN 下的 page version，并按需向 compute 返回 8 KB PostgreSQL page。冷数据可以下沉到对象存储，但对象存储中的内容仍然围绕 page reconstruction 组织，分析引擎不能把它直接当作高效列存扫描。

LTAP 不是在 compute 层合并 OLTP 和 OLAP，而是在 storage 层合并两者的数据源:

```text
                              +----------------------+
                              | Lakehouse / AP Engine|
                              +----------+-----------+
                                         |
                                         | Parquet + recent tail
                                         v
+------------+     WAL     +------------+------------+
| PostgreSQL | ----------> | Safekeeper / PageServer |
|  Compute   | <---------- | row-page serving/cache  |
+------------+   PG page   +------------+------------+
                                         |
                                         | WAL materialization
                                         | row -> column transcoding
                                         v
                              +----------+-----------+
                              | Delta / Iceberg      |
                              | Parquet object store |
                              | durable canonical copy|
                              +----------------------+
```

按照 Databricks 的描述，PageServer 在消费 WAL 并向对象存储 materialize 数据时，不再只生成 PostgreSQL page representation，而是直接把行格式转码成 Parquet。PostgreSQL 仍然执行事务，Lakehouse 引擎仍然执行分析，两边独立扩缩容，但底层共享由 Delta 或 Iceberg 管理的开放列式数据。

## LTAP 最重要的变化: 谁是 Source of Truth

Databricks 对 LTAP 的“single copy”表述很容易引起误解。它不可能意味着整个系统中任意时刻只有一份数据。

系统至少仍然会存在:

- Safekeeper 中尚未完成物化的 WAL
- 对象存储中的 Parquet 文件及其历史版本
- PageServer 中的 row page cache
- PostgreSQL compute 的本地磁盘缓存和 buffer pool
- 为可用性、缓存或对象存储自身冗余产生的额外副本

因此，LTAP 所说的一份数据，更准确地说是**一份持久化、受治理的 canonical representation**，而不是一份物理 bit copy。

更严格地说，在任意时刻，完整的可恢复状态也不只等于已经生成的 Parquet 文件。刚刚提交但尚未完成列式物化的更新仍然以 WAL 形式持久化在 Safekeeper 中。因此 LTAP 的即时 source of truth 应当理解为“已经物化的列式基线 + durable WAL tail”；只有当对应 WAL 被处理并满足回收条件后，列存才独立覆盖那一段历史。本文所说的“列存成为持久化真相”，指的是它取代 PG page 成为长期 canonical data representation，而不是 WAL 可以被取消。

这一区别非常重要。一个系统完全可以保留多份缓存，却只有一个 source of truth。判断某份数据是不是 source of truth，不取决于它有多大，而取决于:

- 丢失后能否从其他持久化状态完整重建
- 正确性和恢复协议是否依赖它继续存在
- 它是否拥有独立的 durability contract
- 发生冲突时，以哪一种表示为准

所以，即使 PageServer 的行式 cache 在某些部署中接近全量，只要它可以被整体删除并从列存与 WAL 中重建，它在架构上仍然是 cache，而不是第二个 source of truth。

但是，这个定义并不能消除工程问题。一个 cache 可以在语义上是可丢弃的，却在性能和可用性上极其关键。如果丢失一个接近全量的 PageServer cache 会造成数小时的延迟抖动或不可接受的冷启动，那么它虽然不是持久化真相，实际运维地位仍然会非常接近一份 serving replica。

Databricks 还明确提到，在 LTAP 的过渡上线阶段会同时向对象存储写入行格式和列格式以校验正确性。因此至少在这个阶段，系统确实存在两种 durable representation，只是官方将其定义为迁移期的验证机制，而不是最终架构。

## 问题一: 无损存储不等于 PostgreSQL 语义

Databricks 表示，多数 PostgreSQL 类型可以直接映射到 Parquet；无法无损映射的值，例如 `NaN`、正负无穷、超出 Parquet decimal 范围的 `NUMERIC` 以及扩展类型，会放在同一张表的 structured overflow field 中，并保留 canonical PostgreSQL text，从而可以重建原始 PostgreSQL bytes。

这回答了 round trip 的问题，但没有完整回答 query semantics 的问题。

Parquet 是存储格式，不是类型系统和算子实现。即使一个值能够无损保存，分析引擎是否能以 PostgreSQL 语义计算它，仍然取决于分析引擎是否实现了对应的:

- 类型解码器
- 比较、排序、算术和聚合算子
- collation 和字符比较规则
- 时区及日期时间规则
- `NaN`、Infinity 和 NULL 的边界行为
- extension type 的 operator class 和自定义函数

例如，PostgreSQL 的 unconstrained `NUMERIC` 可以远远超过常见 decimal 的精度范围。把它的 canonical text 放进 overflow field，确实可以保证未来还原时不丢值，但通用 Spark 或其他 AP engine 并不会因此自动获得 PostgreSQL arbitrary-precision numeric 的完整算术语义。

因此更合理的能力分层可能是:

1. 能直接映射的常见类型，使用 AP engine 的 native vectorized operator 高效计算
2. 需要兼容处理的特殊值，通过额外的 type-aware operator 或 fallback path 计算
3. AP engine 不认识的 extension type，虽然可以读取和 round trip，但只能作为 opaque value，或退化成 text/binary 上的有限操作

公开文章只说明 overflow field “directly queryable”，没有说明任意 AP engine 都能对它执行 PostgreSQL-compatible computation。**可查询不等于算子语义兼容，存储无损也不等于计算无损。** 在看到类型兼容矩阵、算子实现和 conformance test 之前，不能假设 LTAP 已经完整统一了 PostgreSQL 与 Lakehouse 的类型语义。

## 问题二: 只有列值和 CTID，不足以重建一个 PG Page

Databricks 披露，每一个物化到列存的 row version 都携带 physical heap address，也就是 block number 和 offset number。这样 PostgreSQL 仍然可以定位 tuple，并把 heap page 作为点查加速结构。中间 row version 会为了 MVCC 和 PITR 被保留，但不会暴露给 Iceberg/Delta reader；PostgreSQL index 不转码成列，而是由 hot cache tier 服务或重建。

这是一个关键线索，但还不是完整的 page format。

根据 PostgreSQL 的[Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)，一个普通 heap page 除了列值，还包括:

- `PageHeaderData`，其中有 page LSN、checksum、free-space boundary 和 prune hint
- `ItemIdData` line pointer 数组，每项保存 offset、length 和状态
- `HeapTupleHeaderData`，其中有 `xmin`、`xmax`、`cid`、`ctid`、`infomask` 等字段
- NULL bitmap、alignment padding 和 varlena header
- inline、compressed 或 external TOAST representation
- free space 和 tuple 在 page 内的物理摆放

其中有些信息可以重建。例如 checksum 可以重新计算，free-space boundary 可以根据新生成的 page layout 计算，line pointer 可以根据 block/offset 以及 tuple length 生成。它们不一定要逐字节持久化。

但另一些信息不能只从业务列值推导出来:

- `xmin`、`xmax`、command id 和 visibility hint bits
- HOT chain 与 `ctid` 的关系
- tuple 是否使用 external TOAST、具体 TOAST pointer 和 chunk
- 与给定 LSN 对应的 page state
- FSM、VM 以及 index page 所需的派生状态

因此，要兑现“heap pages remain fully reconstructable”，LTAP 至少需要下列信息源之一:

- 在 Parquet 的隐藏列中保存 tuple header、版本和物理布局元数据
- 保留足够的 WAL，并从某个可用基线继续 replay
- 设计一个比原始 page 更高层、但足以确定性生成 page 的 canonical schema
- 组合以上几种方法

目前公开文章没有给出 Parquet schema、隐藏列、TOAST 编码、page reconstruction index 或 WAL retention 之间的具体关系。它说明了目标，也透露了 block/offset 和多版本保留这两个机制，但还不足以验证任意 PostgreSQL heap page 是否能够高效、确定性地重建，更不能确认重建结果是否要求 bit-identical。

这里还需要区分两种“重建”:

- **语义等价 page**: PostgreSQL 可以正确读写，tuple identity 和 MVCC 行为保持不变，但 page 的空闲空间、tuple byte offset、hint bit 或 checksum 可以重新生成
- **字节完全相同的 page**: 每个 header、padding、hint bit 和物理位置都与历史 page 一致

LTAP 为运行 PostgreSQL 通常只需要前者。Databricks 所说的 exact representation 明确覆盖了 value round trip，但公开信息并不能证明整个 page 都采用后者。

## 问题三: PageServer 的 Cache 到底会有多大

LTAP 的 OLTP read path 至少包含三层 cache:

1. PostgreSQL buffer pool
2. compute 节点的 local disk cache
3. PageServer 的 row-format page cache

只有这些层都 miss，才需要从对象存储的 columnar source 重建 row page。Databricks 强调 compute 可以配置与传统数据库相同容量的内存和本地磁盘，因此热数据命中率可以与单机 PostgreSQL 接近。

这个说法能够解释 steady-state hot path，但没有回答 cold path。

从列式文件重建一个随机 8 KB PostgreSQL page，并不是典型的 Parquet 访问模式。系统需要快速完成:

- 从 relation、block 和 offset 定位相关 row versions
- 找到目标 LSN 或 snapshot 下正确的版本
- 读取分散在 column chunk、row group 或增量文件中的隐藏元数据
- 合并尚未 materialize 的 WAL tail
- 重新组成 heap tuple 和 page
- 必要时恢复 TOAST、VM、FSM 或 index 所需状态

列式压缩可以显著降低网络流量，但“传输字节少”不等于“随机页重建便宜”。真正决定性能的是物理聚簇方式、row group 大小、block/offset 索引、增量文件组织、read amplification、并行预取和 cache admission policy。

这也意味着 PageServer cache 的理想大小会随 workload 改变:

- 热集很小、访问集中时，只缓存少量 row page 就可能足够
- 大范围但重复的 OLTP working set 可能需要很大的 row cache
- 随机访问整个数据集时，cache 可能逐渐接近全量
- scale-to-zero、故障切换或 cache eviction 后，冷页性能会成为核心指标

所以，“PageServer 最终会不会保存一份接近全量的 PG page”不能只靠架构图回答。现实部署中完全可能出现这种情况，尤其当活跃数据库规模小于 PageServer 本地 SSD，或者延迟 SLO 要求极高时。但即便如此，它仍然可以是一份 rebuildable materialized view，而非 source of truth。

真正缺失的是公开数据。目前没有看到 LTAP 针对以下场景的 benchmark:

- PageServer row cache 完全冷启动
- cache 只覆盖数据集 1%、10% 或 50% 时的 OLTP P50/P99 latency
- 随机点查触发 columnar-to-page reconstruction 的 read amplification
- PageServer 故障后重新 warming 的时间和资源消耗
- 数据集远大于 row cache 时的吞吐与成本
- index rebuild、VACUUM、TOAST 和大事务对重建路径的影响

在这些数据出现之前，“row page 只是很小的 cache”和“row page 实际上会接近全量”都只是未经验证的 workload-dependent hypothesis。

## 问题四: PG MVCC 如何映射到 AP Snapshot

这是 LTAP 正确性上最关键、公开说明却最简略的部分。

Databricks 的描述是: 分析查询开始时先向 PostgreSQL 获取当前 LSN，然后读取已经 materialize 到对象存储的数据，再从 PageServer 获取少量尚未落入 lake 的近期变化，最终合并成该 LSN 下的一致视图。中间 row version 对 Iceberg/Delta reader 不可见，分析引擎只看到 table-wide consistent snapshot。

但 PostgreSQL snapshot 不是只有一个 LSN。

PostgreSQL 的 MVCC 可见性还涉及 transaction id、事务状态、snapshot 的 `xmin`/`xmax`/in-progress xid 集合、command id、subtransaction，以及 commit/abort record。一个大型事务可以产生大量 WAL，而 commit record 最后才出现。仅仅因为某个 tuple version 的 WAL 位于目标 LSN 之前，不代表它对该 snapshot 可见。

因此，一个合理的 LTAP 实现至少需要类似下面的 visibility protocol。这里是基于 PostgreSQL 语义的推导，并非 Databricks 已公开的具体实现:

1. PageServer 可以提前把未提交事务的 row version 写入物理列存，但不能把它发布到 AP-visible snapshot
2. materializer 需要记录每个 version 对应的 XID、command/版本信息和 commit status
3. 看到 commit record 后，相关版本才有资格进入可见状态；abort 的版本始终不可见
4. Iceberg/Delta snapshot 需要对应一个 committed visibility frontier，而不只是“已经处理到的 WAL byte offset”
5. 对象存储快照之后的 recent tail 也必须使用相同的事务可见性规则过滤
6. 多表事务需要跨表共享一致的读边界，否则 AP reader 可能看到事务更新了 A 表、却还没有更新 B 表

这也解释了 Databricks 所说的“separating durability from visibility”: row version 可以为了恢复而先持久化，但在事务提交之前不能出现在分析视图中。

不过，公开材料还没有回答这些边界条件:

- 获取的是 WAL insert LSN、flush LSN、commit LSN，还是一个单独的 consistent LSN
- 长事务已经物化的数据如何从 Iceberg/Delta reader 隐藏
- 一个 Lakehouse query 跨多张表时是否共享同一数据库级 snapshot
- prepared transaction、subtransaction 和 DDL 如何处理
- PostgreSQL 的 repeatable read snapshot 能否被 AP engine 精确复现
- row version 的 GC 如何同时满足 PostgreSQL vacuum、PITR retention 和 Lakehouse snapshot retention
- analytical snapshot publication 与 catalog metadata commit 如何原子化

因此，技术博客描述了一个可信的总体方向，但“查询开始时获取一个 LSN”只是协议入口，不是完整的 MVCC 答案。

## LTAP 与 HTAP 的真正区别

Databricks 强调 LTAP 不等于 HTAP，因为它没有强迫一个 engine 同时做好事务和分析。这个区别成立，但需要更精确地表达。

| 架构 | 持久化表示 | 计算引擎 | 是否需要数据同步 |
| --- | --- | --- | --- |
| OLTP + CDC + OLAP | 行存与列存各一份 | 分离 | 需要外部或隐藏 pipeline |
| 典型 HTAP | 一份统一存储或内部双格式 | 通常属于同一数据库系统 | 系统内部维护 |
| Neon 式 disaggregated PG | 以 PG page/WAL reconstruction 为中心 | PostgreSQL | AP 仍需要另一条路径 |
| LTAP | 开放列存为 canonical copy，PG page 为 cache | PostgreSQL + Lakehouse engines | 宣称不需要第二份 durable copy |

LTAP 的核心不是“同时支持 TP 和 AP”，而是下面三点同时成立:

1. 两类 workload 使用不同的专用 compute engine
2. 两类 engine 直接建立在同一个开放、列式、可治理的 durable store 上
3. OLTP 所需的行式物理结构可以从这份 durable store 和 WAL 中重建

如果第三点能够在完整 PostgreSQL 语义和可接受性能下成立，那么 LTAP 确实不同于把 CDC pipeline 藏起来，也不同于传统的双格式 HTAP。

## 它是不是一个新的数据库范式

可以从三个层次回答。

### 从单项技术看: 不是

LTAP 依赖的核心组件都已有长期积累:

- WAL 和 MVCC 来自成熟事务数据库
- SafeKeeper/PageServer 来自 Neon 式 disaggregated PostgreSQL
- 对象存储和开放表格式来自 Lakehouse
- 一份数据、多计算引擎是现代数据平台一直追求的方向
- row/column 双重表示和 workload isolation 也是 HTAP 的经典问题

Databricks 自己也明确承认，存算分离并不新。

### 从组合方式看: 有新意

LTAP 最有价值的创新是把已有组件组合成一个新的不变量:

> 开放列存是事务数据唯一的 durable canonical representation，传统 PostgreSQL page 只是按需生成的 serving format。

这是一次值得重视的 storage inversion。过去通常是事务行存为真相，列存是分析副本；LTAP 把这个关系反了过来。如果它成立，CDC 延迟、两份数据治理和分析副本选择就不再是外围工程问题，而是在存储层被消除。

这种组合足以形成一个有辨识度的新架构类别。

### 从“范式”看: 证据还不够

一个产品提出新术语，不代表行业范式已经形成。要成为数据库范式，LTAP 至少还需要证明:

- **语义完整性**: PostgreSQL 类型、MVCC、DDL、extension、TOAST、index、VACUUM 和 PITR 都有清晰契约
- **性能可预测性**: hot path、cold path、故障恢复和 cache miss 都有公开结果
- **一致性可解释**: PG snapshot、LSN 与 Iceberg/Delta snapshot 的映射可以被形式化描述
- **架构可复现**: 不只是一个厂商的专有实现，其他系统也能采用相同原则
- **优势可持续**: 与成熟 CDC、HTAP 和双格式系统相比，在成本、延迟和复杂度上有稳定收益

另外，根据发布时的信息，LTAP 仍处于即将作为 Lakebase 能力推出和逐步上线的阶段，Databricks 也承认当前还在 transitional rollout 中进行双写校验。现在更准确的说法是: **LTAP 是一个有潜力成为新范式的 architecture thesis，而不是已经被生产数据验证的 industry paradigm。**

## 最后

LTAP 最值得关注的地方，不是又发明了一个介于 OLTP 和 OLAP 之间的缩写，而是提出了一个非常激进的判断:

> PostgreSQL page 不必继续作为数据库最终的持久化形态。

如果列式对象存储能够完整保存 PostgreSQL 的值、物理 identity、版本历史和事务可见性，并且能够以足够低的代价重建 row page，那么 OLTP 与 OLAP 的边界确实会被重新划分。事务数据库将更像一个建立在共享 durable state 之上的 compute/serving engine，而不再独占自己的物理数据文件。

但也正因为这个判断如此激进，真正决定 LTAP 成败的不会是架构图中的“one copy”，而是图中没有展开的部分: overflow value 如何计算、tuple metadata 如何编码、未提交 WAL 如何隐藏、随机冷页如何重建、row cache 到底需要多大，以及 cache 丢失后的 P99 延迟是多少。

所以，对“LTAP 是否是新的数据库范式”最合适的回答是:

**概念上，它提出了一个新的 storage-centric architecture；工程上，它仍是一组等待公开细节与 benchmark 验证的主张；行业上，它目前是范式候选，而不是范式结论。**

## 参考资料

- [Databricks Launches LTAP: The First Lake Transactional/Analytical Processing Architecture](https://www.databricks.com/company/newsroom/press-releases/databricks-launches-ltap-first-lake-transactionalanalytical)
- [From monolith to Lakebase to LTAP: rethinking the database from storage up](https://www.databricks.com/blog/lakebase-ltap-rethinking-database-storage)
- [Neon architecture](https://neon.com/docs/introduction/architecture-overview)
- [PostgreSQL Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
- [PostgreSQL MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [PostgreSQL Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html)
