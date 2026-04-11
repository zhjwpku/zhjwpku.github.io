---
layout: post
title: "向量检索中的 Filter 技术: In-index Filtering、Post-filtering 与 Pre-filtering"
date: 2026-04-11 00:00:00 +0800
tags:
- vector-search
- rag
- database
- ai
---

向量检索解决的是“谁和查询最相似”，但真实系统很少只问这个问题。业务真正要问的通常是:

- 用户有权限看到的最相似内容是什么
- 某个租户下最相似的文档是什么
- 某个时间窗口内最相似的商品或事件是什么
- 某种语言、某个地区、某种状态下最相似的记录是什么

这时，系统要做的不只是 similarity search，还要把结构化条件一起纳入检索流程。也就是说，**向量检索几乎总是要配合 filter**。

Neo4j 在 2026 年 1 月的预览版本里，把这一问题总结得很清楚: 向量检索中的过滤，通常可以分成三类:

- `In-index filtering`: 在向量索引内部过滤
- `Post-filtering`: 先向量检索，再过滤
- `Pre-filtering`: 先过滤候选集，再做相似度计算

虽然原文使用的是 Cypher 和 Neo4j 的例子，但这三个模式并不依赖图数据库。无论你使用的是向量数据库、搜索引擎、关系型数据库、文档数据库，还是自建 ANN 服务，本质上都绕不开这三种技术路线。本文就围绕这三种 filter 技术，系统梳理它们的原理、性能特征、适用场景和工程上的取舍。

## 为什么向量检索一定会遇到 Filter

如果没有 filter，向量检索的目标很简单: 从全量向量集合中找到与查询向量最接近的 Top K。

但一旦进入生产环境，几乎所有系统都会遇到额外约束:

- 多租户隔离: 只能搜当前 tenant 的数据
- 权限控制: 用户只能看自己有权限访问的记录
- 生命周期约束: 只看未删除、未归档、已发布的数据
- 业务范围限定: 只看某个品类、语言、地区、时间段
- 图谱或关系条件: 只看某个项目、某个作者、某类实体关联下的数据

这些条件如果处理不好，会立刻引出三个典型问题:

### 1. 结果不对

向量最相似的记录可能根本不满足业务约束。比如搜出来的文档语义很相关，但属于别的租户，或者已经归档。

### 2. Top K 不够

如果先搜 Top 10，再过滤掉 8 个，最后只剩 2 个，用户看到的结果数量和质量都会下降。

### 3. 性能不稳定

filter 越苛刻，系统为了凑齐足够结果，往往需要扩大候选集，导致延迟上升，而且召回还未必稳定。

所以，filter 不是向量检索的附属功能，而是决定系统正确性、延迟和召回率的核心设计点。

## 三种 Filter 路线

先给出一个总体定义:

- `In-index filtering`: 把部分过滤条件放进向量索引内部，让索引在搜索过程中直接跳过不符合条件的向量
- `Post-filtering`: 先做 ANN 检索拿到候选，再在索引外按结构化条件过滤
- `Pre-filtering`: 先用结构化查询缩小候选集，再只在候选集内做向量相似度计算

如果把整个检索过程抽象成一个流水线，可以写成:

```text
Query -> Candidate Generation -> Similarity Ranking -> Result Filtering -> Top K
```

三种方案的差别，本质上就是 **filter 被放在流水线的哪个阶段**。

## 一、In-index Filtering

### 定义

In-index filtering 指的是: 在 ANN 索引执行搜索时，同时带上结构化谓词，让索引“看起来像只包含满足条件的向量”。

Neo4j 在文章里给出的 Cypher 写法大致如下:

```cypher
CREATE VECTOR INDEX documentIndex
  IF NOT EXISTS
  FOR (document:Document)
  ON document.embedding
  WITH [document.author, document.published_year]
```

查询时:

```cypher
MATCH (document)
  SEARCH document IN (
    VECTOR INDEX documentIndex
    FOR $query_vec
    WHERE document.author = 'David'
      AND document.published_year >= 2020
    LIMIT $top_k
  ) SCORE AS score
RETURN document, score
```

这里最关键的点不是 Cypher 语法，而是这件事的实现思想:

- 向量索引里不仅保存 embedding
- 还保存一部分可过滤的 metadata
- 搜索时先判断 metadata 是否满足条件
- 只有满足条件的向量才参与最终候选扩展和排序

### 它为什么快

ANN 索引通常使用 HNSW、IVF、PQ 等结构。普通 post-filter 的做法是先按向量相似度找若干邻居，再把不合格的结果丢掉。这样做的问题是，很多相似度计算和图遍历是“白做了”。

In-index filtering 则不同。它让索引在搜索过程中就知道:

- 哪些向量根本不可能成为合法结果
- 哪些区域即使相似，也不需要继续扩展
- 应该把计算预算优先用在满足 filter 的候选上

因此它通常同时带来两个效果:

- 延迟更稳定
- 在同等延迟下召回更高

Neo4j 的测试结论也是这个方向: 无论 filter 较宽还是较窄，in-index filtering 都能保持较低延迟和较高召回。

### 它的本质约束

In-index filtering 的优势很明显，但限制也同样明确:

### 1. 只能过滤索引里“看得见”的字段

如果某个条件没有作为 metadata 存进索引，就没法在索引内部过滤。

例如这些通常适合做 in-index filtering:

- `tenant_id`
- `language`
- `category`
- `published_year`
- `status`
- `is_deleted`

而这些往往不适合直接塞进索引:

- 用户与资源之间的复杂权限关系
- 需要跨表 join 才能得到的条件
- 图遍历、多跳关系、路径约束
- 高度动态、组合复杂的业务规则

### 2. 索引设计要提前规划

一旦某些 metadata 没放进索引，后面通常要重建索引才能支持该类过滤。这意味着索引 schema 设计会变成一项前置工作。

### 3. 支持的谓词类型通常是受限的

很多系统只支持简单谓词，例如:

- 等值
- 范围
- `IN`
- 布尔条件

复杂逻辑表达式、子查询、关系型连接，通常不在 in-index filtering 的能力边界里。

### 通用伪代码

对任何数据库，这一模式都可以抽象成:

```text
vector_search(
  query = q,
  top_k = 10,
  filter = {
    tenant_id = 42,
    language = "zh",
    published_at >= "2025-01-01"
  }
)
```

如果底层引擎能在索引内部执行 `filter`，那就是 in-index filtering。

### 适用场景

它特别适合:

- 多租户隔离
- 语言、地区、品类、时间等简单属性过滤
- 需要稳定低延迟的在线检索
- Top K 必须稳定返回
- 数据规模很大，不能接受大量 over-fetch

一句话概括: **当 filter 可以被简单属性表达，并且这些属性值得进入索引时，优先考虑 in-index filtering。**

## 二、Post-filtering

### 定义

Post-filtering 是最直观、最常见、也最容易先做出来的一种方案:

1. 先从向量索引取回若干相似结果
2. 再用数据库查询或业务逻辑过滤
3. 最后从剩余结果里取 Top K

Neo4j 的例子如下:

```cypher
MATCH (document)
  SEARCH document IN (
    VECTOR INDEX documentIndex
    FOR $query_vec
    LIMIT $ef_search
  ) SCORE AS score
MATCH (project:Project)<-[:WRITTEN_FOR]-(document)
      -[:WRITTEN_BY]->(author:Author)
WHERE author.name = 'David'
  AND document.published_year >= 2020
  AND project.completed = true
RETURN document, project, author, score
ORDER BY score DESC
LIMIT $top_k
```

这里向量检索先发生，后面的 `MATCH` 和 `WHERE` 再做结构化条件过滤，这就是标准的 post-filtering。

### 为什么它最常见

因为它几乎不要求底层向量索引支持什么特殊能力。只要系统能:

- 返回 ANN 候选
- 对候选再做结构化过滤

就能实现。

所以在很多系统里，第一版向量检索基本都是 post-filtering。实现成本低，兼容性好，也容易和已有数据库能力整合。

### 它的核心问题: 过滤发生得太晚

假设你要的是“当前租户下 Top 10 相似文档”，但你先从全库拿 Top 10，再过滤 tenant，最终可能一个都不剩。

这时只能扩大初始候选数，比如:

- 不取 10 个，而是先取 100 个
- 过滤后如果还不够，再取 500 个
- 还不够就继续扩大

这就是常说的 `over-fetch`。

### Over-fetch 的代价

post-filtering 最大的问题，不是不能用，而是当 filter 很“窄”时，它会越来越昂贵。

举个简单例子:

- 如果 filter 只保留 50% 的数据，那么取 20 个候选，可能还比较容易凑齐 Top 10
- 如果 filter 只保留 1% 的数据，那么可能要先取 1000 个候选，才能留下 10 个有效结果
- 如果 filter 只保留 0.001% 的数据，over-fetch 很快会失控

这会带来两个后果:

- 延迟上升，因为 ANN 需要返回更多候选
- 召回下降，因为即便扩大候选，也不一定覆盖到真正属于过滤子集的最优邻居

换句话说，post-filtering 的根本缺陷在于:

**ANN 的搜索目标是“全局相似”，而不是“过滤子集内相似”。**

### 为什么它会伤害召回

设想全库里最相似的前 100 个向量，绝大部分都不满足 filter。那 ANN 返回的候选，天然就被“无效结果”占满了。真正属于过滤子集的优质结果，可能排在全库的第 5000 名之后，根本不会进候选池。

因此，post-filtering 取到的结果通常不是:

- “过滤子集中的 Top K”

而更像是:

- “全局 Top N 经过过滤后剩下的一小部分”

两者不是一回事。

### 它什么时候依然是正确选择

虽然有缺陷，post-filtering 仍然有很强的实用价值，尤其在下面几种情况下:

- filter 逻辑复杂，无法放进索引
- 需要图遍历、join、权限判断、规则引擎
- 需要在检索后扩展相关实体一起返回
- filter 选择性不高，筛掉的结果不多
- 系统处于早期阶段，优先要快速上线

### 通用伪代码

```text
candidates = ann_search(query = q, top_n = 200)
results = filter(candidates, business_predicate)
return top_k_by_score(results, 10)
```

如果 `top_n` 需要显著大于 `top_k`，就说明系统已经在依赖 over-fetch。

### 工程建议

如果必须使用 post-filtering，至少要注意三件事:

### 1. 把 `top_k` 和 `over_fetch` 分开

不要把“返回多少条”和“先抓多少候选”混为一谈。两者应该分别控制。

### 2. 监控过滤后命中率

核心指标包括:

- 候选命中率: `filtered_count / fetched_count`
- 最终 Top K 满足率
- 过滤前后平均延迟
- 不同 filter selectivity 下的召回率

### 3. 对窄 filter 设置退化策略

当某些 filter 极其苛刻时，应考虑:

- 切到 pre-filtering
- 切到 in-index filtering
- 降低 K
- 明确告诉用户结果不足，而不是盲目 over-fetch

## 三、Pre-filtering

### 定义

Pre-filtering 的流程与 post-filtering 正好相反:

1. 先通过结构化查询找出满足条件的候选集
2. 只在这个候选集上做向量相似度计算
3. 从中返回 Top K

Neo4j 的 Cypher 示例:

```cypher
MATCH (project:Project)<-[:WRITTEN_FOR]-(document:Document)
      -[:WRITTEN_BY]->(author:Author)
WHERE author.name = 'David'
  AND document.published_year >= 2020
  AND project.completed = true
WITH document, project, author,
     vector.similarity.cosine(document.embedding, $query_vec) AS score
RETURN document, project, author, score
ORDER BY score DESC
LIMIT $top_k
```

这里先由图查询找出候选文档，再计算余弦相似度。这种模式的关键，不在于 Cypher，而在于:

- 过滤子集先被准确地定义出来
- 相似度只在这个子集内计算

### 它最大的价值: 结果语义最正确

pre-filtering 求解的是:

- “满足 filter 的候选集合中的 Top K”

如果你对子集内每个向量都算一次相似度，再排序，那么得到的就是该子集上的精确最近邻，通常可视作 `ENN`。

这与 post-filtering 有本质不同。post-filtering 是先做全局 ANN，再裁剪；pre-filtering 是先定义合法空间，再在合法空间里求最优。

因此 pre-filtering 的最大优势是:

- 召回最稳定
- 语义最精确
- 最容易满足权限、合规、隔离等强约束

### 它的问题: 子集太大时会很慢

pre-filtering 的瓶颈也非常直接。假如过滤后还有几百万条候选，那你仍然要对这几百万条记录做相似度计算，代价很高。

因此，pre-filtering 是否高效，关键取决于 filter 能否把候选集缩得足够小。

如果 filter 后只剩:

- 几百条
- 几千条
- 几万条

那么 pre-filtering 往往很好用，甚至是最好的方案。

但如果 filter 后还有:

- 几十万条
- 上百万条

那么纯粹的 pre-filtering 通常就会成为瓶颈。

### 通用伪代码

```text
subset = structured_query(
  tenant_id = 42,
  language = "zh",
  category = "database"
)

scored = [
  (item, similarity(item.embedding, q))
  for item in subset
]

return top_k(scored, 10)
```

本质上，这是一种“先精确过滤，再精确排序”的思路。

### 适用场景

pre-filtering 特别适合:

- 权限或合规要求很强，不能先搜全局再过滤
- filter 能显著缩小候选集
- 关系约束复杂，需要 join、图遍历或规则推理
- 需要子集内精确 Top K，而不是近似结果
- 中小规模候选集上的高质量检索

一句话概括: **当 filter 足够强，能先把搜索空间压小，而且你需要对子集结果有强保证时，pre-filtering 是最稳的方案。**

## 三种方案的本质差异

下面把三种模式放在一起比较:

| 维度 | In-index filtering | Post-filtering | Pre-filtering |
|:-|:-|:-|:-|
| filter 发生位置 | 向量索引内部 | ANN 之后 | ANN 之前 |
| 对 filter 表达能力 | 弱到中 | 强 | 最强 |
| 查询延迟 | 通常低且稳定 | 中等，窄 filter 时恶化 | 小候选集快，大候选集慢 |
| 召回表现 | 通常较高 | 容易下降 | 可做到精确 |
| 是否需要 over-fetch | 通常不需要 | 经常需要 | 不需要 |
| 是否依赖索引 schema | 强依赖 | 弱依赖 | 不依赖向量索引 filter 能力 |
| 适合复杂关系条件 | 不适合 | 适合 | 最适合 |
| 适合超大规模在线检索 | 很适合 | 有条件适合 | 取决于过滤后规模 |

这里有一个非常实用的判断原则:

- 想要低延迟且 filter 简单: 用 `in-index filtering`
- 想要表达复杂业务规则: 用 `post-filtering` 或 `pre-filtering`
- 想要对子集结果做强保证: 用 `pre-filtering`

## Filter Selectivity 决定一切

讨论这三种方案时，最重要的概念之一是 **filter selectivity**，也就是 filter 后保留下来的数据比例。

例如:

- 保留 80% 数据: 宽 filter
- 保留 10% 数据: 中等 filter
- 保留 0.1% 数据: 窄 filter
- 保留 0.001% 数据: 极窄 filter

selectivity 会直接决定最优方案:

### 宽 filter

如果 filter 很宽，post-filtering 常常还能接受，因为被筛掉的候选不多，over-fetch 的代价有限。

### 窄 filter

如果 filter 很窄，post-filtering 会迅速失效，因为你需要抓取大量全局候选才能剩下足够结果。

此时更好的方案通常是:

- 用 in-index filtering，把 filter 直接推入 ANN
- 或用 pre-filtering，先把候选子集缩小后再精排

### 极窄 filter

如果 filter 极窄，pre-filtering 可能反而最划算。因为虽然它是精确算相似度，但候选集已经足够小，计算成本完全可控。

所以，系统设计时不要只问“支不支持 filter”，而要问:

- filter 平均会保留多少数据
- 最差情况下会保留多少数据
- 不同类型 filter 的 selectivity 分布如何

很多检索系统的性能问题，本质上都不是算法不够好，而是没有按 selectivity 对检索路径做分流。

## 为什么 In-index Filtering 本质上是“把标量检索推入 ANN”

这一点很值得单独说清楚。

传统数据库里，结构化过滤和排序通常是两件分开的事:

- 先用索引或谓词筛选行
- 再做排序或其他运算

但向量检索的特殊性在于，排序本身就非常昂贵。尤其在 ANN 场景下，候选扩展的方向会受到距离函数影响。如果 filter 不提前参与，ANN 实际上是在错误的搜索空间里工作。

从这个角度看，in-index filtering 的意义并不只是“少做几次过滤”，而是:

- 让 ANN 的搜索空间从一开始就是正确的
- 让近似搜索的近似对象变成“过滤后的子空间”

这也是为什么它相比 post-filtering 往往同时改善延迟和召回。因为它并不是简单少做一步，而是在更正确的空间里做搜索。

## 为什么 Pre-filtering 经常是权限系统的唯一正确答案

很多团队在做 RAG 时，一开始会把权限检查放在检索之后，觉得“先搜，再过滤掉无权限文档”就行。

这在体验和安全上都存在问题。

从结果正确性看:

- 你想找的是“用户有权限看到的最相关文档”
- 不是“全局最相关文档中过滤后剩下的文档”

从合规角度看:

- 某些系统甚至不允许未经授权的数据进入候选集或日志链路
- 更不能允许它参与 rerank、摘要或下游 LLM prompt 构造

因此，对于强权限场景，pre-filtering 往往不是性能优化选项，而是语义与合规上的基础要求。

如果候选集还不够小，可以考虑:

- 先按 tenant、ACL group、project 等做粗粒度预过滤
- 再在粗过滤后的集合上使用 in-index filtering 或局部 ANN

换言之，真正成熟的系统往往不是三选一，而是组合使用。

## 组合方案通常比单一方案更现实

现实系统里，经常会看到下面几种组合:

### 1. In-index filtering + Post-filtering

先把简单而高频的条件推入索引，例如:

- `tenant_id`
- `language`
- `status`

再在检索结果上做复杂业务逻辑，例如:

- 图遍历
- join
- 权限补充判断
- 实体扩展

这是非常实用的一种折中方案。Neo4j 原文也明确提到，in-index filtering 可以和 post-filtering 结合。

### 2. Pre-filtering + 精确排序

先用数据库能力缩小到一个明确的小集合，再直接做精确余弦相似度排序。适合权限强约束或高价值检索场景。

### 3. Pre-filtering + 局部 ANN

先通过结构化查询把候选缩到某个分区或租户，再在这个局部空间里执行 ANN。很多多租户系统本质上都在做这件事，只是实现方式不同。

## 如何做工程选型

下面给出一个可以直接落地的决策流程。

### 场景一: 过滤条件简单，且请求量大、延迟要求严格

例如:

- tenant
- 语言
- 上下架状态
- 品类
- 时间范围

优先选择 `in-index filtering`。如果底层数据库支持，就尽量把这些字段纳入索引 metadata。

### 场景二: 过滤规则复杂，涉及关系、权限或多表条件

优先考虑 `pre-filtering` 或 `post-filtering`。如果候选集能被明显缩小，优先 `pre-filtering`；如果不能，就考虑 `in-index + post` 的混合方案。

### 场景三: 必须保证“过滤子集内 Top K”语义正确

优先 `pre-filtering`。这是最能保证语义正确性的方式。

### 场景四: 先求能上线，再求最优

先用 `post-filtering` 是务实选择，但要从第一天开始监控:

- over-fetch 倍数
- 过滤后剩余比例
- Top K 满足率
- 不同 filter 的尾延迟

一旦这些指标恶化，就要准备迁移到 in-index 或 pre-filtering。

## 实践中的几个常见误区

### 误区一: “数据库支持 filter” 就等于问题解决了

不是。关键不在于有没有 filter 参数，而在于它是:

- 索引内过滤
- 还是检索后过滤

这两者的性能和召回差异很大。

### 误区二: over-fetch 可以无限解决 post-filtering 的问题

不是。over-fetch 只能缓解，不能根治。随着 filter 变窄，成本会快速上升，而且召回仍然可能不足。

### 误区三: pre-filtering 一定慢

也不是。如果 pre-filter 能把候选集压到很小，它反而可能是最快且最准的。

### 误区四: 所有 metadata 都应该塞进向量索引

不对。索引 metadata 不是越多越好。应该优先放入:

- 高频使用
- 表达简单
- 区分度高
- 对延迟影响大的过滤条件

复杂且低频的条件，通常不值得为它们重建或膨胀索引。

## 一个简单的通用思维模型

面对向量检索 filter 问题时，可以按下面三个问题判断:

### 1. 这个 filter 能否用简单标量字段表示

如果能，考虑 `in-index filtering`。

### 2. 这个 filter 能否先把候选集压到足够小

如果能，考虑 `pre-filtering`。

### 3. 这个 filter 是否依赖复杂业务逻辑、关系或后处理

如果是，考虑 `post-filtering`，或者和前两者组合。

把这三个问题串起来，通常就能找到比较合适的方案。

## 总结

向量检索中的 filter，不是一个附加小功能，而是整个检索系统设计的主轴之一。`In-index filtering`、`Post-filtering`、`Pre-filtering` 三种方式，分别代表了三种不同的系统取舍:

- `In-index filtering` 追求的是在正确子空间里的低延迟 ANN，最适合简单属性过滤和大规模在线检索
- `Post-filtering` 追求的是实现简单和表达灵活，但在窄 filter 下容易出现召回下降与延迟恶化
- `Pre-filtering` 追求的是对子集结果的精确控制，最适合强权限、复杂关系和高精度场景

真正成熟的系统，通常不会只押注一种方式，而是根据 filter 的表达复杂度、selectivity、权限要求和延迟预算，动态选择或组合这些策略。

如果只记住一句话，那就是:

**不要只问“向量数据库支不支持 filter”，而要问“filter 在检索流水线的哪个阶段执行”。**

这个问题，往往比相似度函数、embedding 维度甚至索引类型本身都更影响最终效果。

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 <a href="https://neo4j.com/blog/genai/vector-search-with-filters-in-neo4j-v2026-01-preview/">Vector search with filters in Neo4j v2026.01 (Preview)</a><br>
2 <a href="https://neo4j.com/docs/cypher-manual/current/clauses/search/">Neo4j Cypher Manual: SEARCH</a>
</span>
