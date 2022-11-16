---
layout: post
title: Sigmod Programming Contest 2018
date: 2022-11-20 00:00:00 +0800
tags:
- contest
---

阿里云数据库今年搞了个编程大赛，采用的是 [Sigmod 2018 年编程大赛](https://db.in.tum.de/sigmod18contest/task.shtml) 的赛题。任务是在一组预先定义好的表上执行多批次的 SQL 查询，每个查询都会指定一组表、一组 JOIN 条件（Predicate）和 Selection（Aggregations），即每个查询都是典型的 SPJA 查询。

比赛提供了一个简单的代码框架，其中包含解析器，执行器，执行器 Join 使用的 HashJoin，在此基础上，我做了一些简单的优化，优化后的结果：

![](/assets/2022/dber_never_mess_up.jpg)

#### Optimizer

一、对 SQL 语句进行改写:

1. 去掉一些重复的 join 条件，减少不必要的 Self Join
```txt
original query: 5 0 8|0.0=1.0&1.0=0.0&1.0=2.0&0.0<10219&2.0<26996|1.0 0.1
rewrite query: 5 0 8|0.0=1.0&1.0=2.0&0.0<10219&2.0<26996|1.0 0.1
```

2. 去掉一些多余的 filter 条件，减少 filterScan 耗时
```txt
original query: 31 13 31 31|0.1=1.0&0.1=2.2&2.1=3.2&3.1>2271881&3.1<3852590&3.1<5146007|1.1 0.0 0.1
rewrite query: 31 13 31 31|0.1=1.0&0.1=2.2&2.1=3.2&3.1>2271881&3.1<3852590|1.1 0.0 0.1
```

3. 把条件传递给相同 join 条件的另一端，以减少 HashJoin build/probe 的数据量
```txt
original query: 26 14 32|0.1=1.0&1.0=2.2&0.1<38361346&2.0>4300244|1.1 1.0
rewrite query: 26 14 32|0.1=1.0&1.0=2.2&2.2<38361346&1.0<38361346&2.0>4300244&0.1<38361346|1.1 1.0
```

4. 增加同一个表上的 join 条件，期望能够通过单表的 Self Join 获取更小的中间结果
```txt
original query: 8 0 7|0.1=1.0&0.2=1.0&1.0=2.2&0.2<239914|2.0 0.0 2.0
rewrite query: 8 0 7|0.1=0.2&0.1=1.0&0.2=1.0&1.0=2.2&0.2<239914|2.0 0.0 2.0
```

5. 识别一些 impossible 的查询，直接返回结果
```txt
original query: 28 27 28|0.3=1.2&0.1=2.2&2.1<2980720&1.3<9572&2.1=3694644|2.0 0.3 1.1
```

二、调整 JOIN 顺序：

赛题给出的 Left Deep Tree 没有做任何优化。我通过对数据进行 sample，获取每一列数据的 min/max/ndv，对 JOIN 的顺序进行了排序，原则是能做 SelfJoin 的 Predicate 尽量先做，根据 FilterInfo 将过滤结果量较少的 Predicate 往前排，以期得到更小的中间结果。

#### Executor

一、HashJoin 并发

对原有的 HashJoin 进行了并发优化，在 build 阶段使用最多 4 个线程将 left 数据集分段构建 hash table，然后在 probe 阶段使用 32 个线程（将 right 数据集分成 8 段，每段使用四个线程分别去对四个 hash table 做 probe），中间结果（左右数据集的位置对）保存为 `std::vector<std::array<uint64_t, 2>>`，最后对这些数据进行 copy2Result。因为已经知道了结果集的大小，因此首先对 tmpResult 的各列做 reserve，然后使用 32 个线程直接在对应的位置赋值（tmpResult 的 size() 不会被上层算子使用，因此这么干没有问题），多线程之间不需要进行同步。

后面借鉴 [PaperCup](https://db.in.tum.de/sigmod18contest/downloads/papercup.pdf) 的思路重写了 HashJoin:

- 对 leftResult 分桶，并开辟一块新的连续内存将数据分发到各自的桶中
- 对 rightResult 做同样分桶
- 对桶号相同的左右两边的数据进行 HashJoin，如果右边的数据量太多，对其进一步切分后再分发给不同的线程去 Join

二、AGG 下推

将 Checksum 算子计算直接推到下面的算子（通常是 HashJoin 或 SelfJoin），子算子直接在各任务中的中间结果中保存 sum 结果而非 Join 的数据值。

三、SelfJoin & FilterScan 并发

同样使用多线程去过滤，最后将结果再合并到一起。

对于只有一个条件的 FilterScan，将 applyFilter 的 switch 拿到 for 循环的外边。

#### 做了没效果的优化

- 延迟物化: tmpResults 只保存各表的行号（使用 uint32_t，能够节省一定内存），在 medium 数据集测试有一定的效果，但在大数据集没啥效果

#### 想做未做的优化

- SIMD: 由于省去了最后一层 HashJoin 的中间结果，Sum Agg 只需返回结果，因此只有 FilterScan 中能够进行一些加速，不确定收益，所以没做
- Rewriter: 将 min/max 也传递给对应的 SelectInfo；通过对数据进行索引，获得 PK/FK 等信息，据此删除一些中间 Join
- Bushy Tree: 虽然对 JOIN 顺序进行了调整，不过最后的执行计划依然是一颗 Left Deep Tree。获取可以生成一棵 Bushy Tree 来获取更好的并行？

#### 不足

- 优化器做的比较简单，比如下面的查询，改写之后多出了一个 predicate，虽然新增的 predicate 能够过滤更多的数据，但也增加了一个 SelfJoin。理想的状态是新增一个 predicate 肯定能去掉一个旧的，但是这种 case 并不很多，没有进一步优化。
```txt
original query: 7 0|0.1=1.0&0.2=1.0&0.2<794389|0.0
rewrite query: 7 0|0.1=0.2&0.1=1.0&0.2=1.0&0.1<794389&0.2<794389&1.0<794389|0.0
```

- 代码缺乏模块化，导致代码有点乱。

#### Summary

这次比赛对入门数据库有很大的帮助，rewriter、optimizer、executor 都有所涉及，非常棒的比赛！

我的代码路径: [https://github.com/wolfdb/DBProgramContest](https://github.com/wolfdb/DBProgramContest)。

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [Leaderboard](https://db.in.tum.de/sigmod18contest/leaders.shtml)<br>
2 [Quickstep: A Data Platform Based on the Scaling-In Approach](https://minds.wisconsin.edu/bitstream/handle/1793/76552/TR1847.pdf)<br>
3 [Quickstep source](https://github.com/UWQuickstep/quickstep)<br>
4 [vsb_ber0134](https://db.in.tum.de/sigmod18contest/downloads/vsb_ber.pdf)<br>
5 [PaperCup](https://db.in.tum.de/sigmod18contest/downloads/papercup.pdf)<br>
</span>
