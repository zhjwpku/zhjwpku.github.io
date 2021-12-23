---
layout: post
title: Awesome Flink Learning Resources
date: 2021-11-01 00:00:00 +0800
tags:
- flink
---

Apache [Flink](https://flink.apache.org/) 学习资料，包括但不限于文章、书籍、教程、视频。

<h4>Blogs</h4>

- [Streaming 101: The world beyond batch][stream101] by Tyler Akidau (Aug 5, 2015)
- [Streaming 102: The world beyond batch][stream102] by Tyler Akidau (Jan 20, 2016)
- **Exactly Once Semantics**
  - [High-throughput, low-latency, and exactly-once stream processing with Apache Flink™][high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink] by Kostas Tzoumas (August 5, 2015)
  - [An Overview of End-to-End Exactly-Once Processing in Apache Flink][end-to-end-exactly-once-apache-flink] by Piotr Nowojski & Mike Wintersv (Mar 1, 2018)
  - [Fault Tolerance via State Snapshots][fault_tolerance] from Flink docs
  - [Stateful Stream Processing][stateful-stream-processing] from Flink docs
  - [From Aligned to Unaligned Checkpoints - Part 1: Checkpoints, Alignment, and Backpressure][from-aligned-to-unaligned-checkpoints-part-1] by Arvid Heise & Stephan Ewen (Oct 15, 2020)

<h4>Books</h4>

- [Introduction to Apache Flink][intro_to_flink] by Ellen Friedman, Kostas Tzoumas (Nov, 2016)

<h4>Papers</h4>

- [Nephele: Efficient Parallel Data Processing in the Cloud](https://paper-notes.zhjwpku.com/scheduler/nephele.html) - Nephele 是 Flink 的前身，文章主要介绍 Flink 的调度
- [Lightweight Asynchronous Snapshots for Distributed Dataflows](https://paper-notes.zhjwpku.com/distributedsystem/abs.html) - 介绍 Flink Checkpoint 依赖的分布式快照算法

<h4>Videos</h4>

- [尚硅谷Java版Flink][atguigu_flink] &[【学习笔记】][atguigu_flink_notes]

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [Awesome Flink](https://github.com/wuchong/awesome-flink)<br>
2 [Readings in Streaming Systems](https://github.com/lw-lin/streaming-readings)<br>
</span>

[intro_to_flink]: https://www.oreilly.com/library/view/introduction-to-apache/9781491977132/
[stream101]: https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/
[stream102]: https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/
[atguigu_flink]: https://www.bilibili.com/video/BV1qy4y1q728
[atguigu_flink_notes]: https://ashiamd.github.io/docsify-notes/#/study/BigData/Flink/%E5%B0%9A%E7%A1%85%E8%B0%B7Flink%E5%85%A5%E9%97%A8%E5%88%B0%E5%AE%9E%E6%88%98-%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0
[high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink]: https://www.ververica.com/blog/high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink
[end-to-end-exactly-once-apache-flink]: https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html
[fault_tolerance]: https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/
[stateful-stream-processing]: https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/
[from-aligned-to-unaligned-checkpoints-part-1]: https://flink.apache.org/2020/10/15/from-aligned-to-unaligned-checkpoints-part-1.html