---
layout: post
title: Z-order 技术简介
date: 2021-08-01 00:00:00 +0800
tags:
- z-order
---

Z-order 又叫作 Morton-order 或 Morton code（莫顿码），是一种空间填充曲线，将多维数据映射到一维，同时保留数据点的局部性，由 G. M. Morton 于 1966 年发明。

在数据分析场景中，通常会通过创建索引的方式来加速查询，但对于多维查询，如果不包含第一个索引字段，并不能有效过滤不想干数据。适用 Z-order 技术可以将多个字段压缩成一维数据（z-value），z-value 的映射规则保证了按照它进行排序后，原来在多维维度临近的数据在一维曲线上依然是彼此临近的。

z-value 的计算规则也比较简单，将每一维的数据表示为比特，然后进行位交叉，如何下图是对 x: [0,7]，y: [0:7] 的数据进行 z-order 计算，得到如下图所示的结果:

![](https://upload.wikimedia.org/wikipedia/commons/2/29/Z-curve45.svg)

在实际生成 z-value 的过程中，需要考虑更多的问题，对于无符号整型可以直接转换为 bits 位，但其它类型需要进行一定的处理:

- 有符号数: 为了避免排序中负数大于正数，将二进制最高位进行反转，这样转换后就能保证正数一定是大于负数的
- Float/Double: 转换成 Integer/Long，然后按照有符号数进行 crossing and merging
- Decimal/Date/TimeStamp: 转换成 Long，直接进行转换
- UTF-8 String: 由于字符串长度不固定，如果不足8字节进行填充，超过8字节则截断为8字节

但如上处理之后，依然会有两个问题:

- 生成 z-value 的字段如果不是从 0 开始递增，那么生成的只是完整 z 曲线的一部分，不能保证很好的聚合效果
- 对于有公共前缀的字符串，则参与计算 z-value 的字节毫无意义

为了解决这两个问题，Hudi 使用了 Boundary-base Interleaved Index。

- 对数据进行 sample
- 对参与 z-order 计算的每一个字段确定边界值，类似计算 histogram
- 每个字段的值都能映射到对应的边界，使用边界值的 index 值取参与 z-value 的计算，由于 index 从 0 开始递增，满足生成一个完整 z 曲线的要求

很多产品都使用了 z-order 排序:

- Delta Lake
- Hudi
- Iceberg
- HBase
- Amazon DynamoDB
- [ClickHouse](https://github.com/ClickHouse/ClickHouse/pull/41753)

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [Z-order curve](https://en.wikipedia.org/wiki/Z-order_curve)<br>
2 [morton-nd: A header-only compile-time Morton encoding/decoding library for N dimensions](https://github.com/morton-nd/morton-nd)<br>
3 [Data skipping with Z-order indexes for Delta Lake](https://docs.databricks.com/delta/data-skipping.html)<br>
4 [RFC-28 Support Z-order curve](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181307144)<br>
5 [How Z-Ordering in Apache Iceberg Helps Improve Performance](https://www.dremio.com/blog/how-z-ordering-in-apache-iceberg-helps-improve-performance/)<br>
6 [Z-Order Indexing for Multifaceted Queries in Amazon DynamoDB: Part 1](https://aws.amazon.com/cn/blogs/database/z-order-indexing-for-multifaceted-queries-in-amazon-dynamodb-part-1/)<br>
7 [Z-order indexing for multifaceted queries in Amazon DynamoDB: Part 2](https://aws.amazon.com/cn/blogs/database/z-order-indexing-for-multifaceted-queries-in-amazon-dynamodb-part-2/)<br>
</span>

