---
layout: post
title:  "RADOS: 一个可扩展、可靠的PB级分布式存储集群服务"
date:   2016-08-06 22:30:00 +0800
categories: ceph
tags:
- thesis
- translation
---

本文是对[Sage Weil][sage]博士期间发表的论文之一[RADOS: A Scalable, Reliable Storage Service for Petabyte-scale Storage Clusters][rados]的翻译。

<h4>摘要</h4>

基于块(原文是Brick)和对象的存储架构作为一种增强存储集群的可扩展性的方式得以出现。然而，现有的系统仍然将存储节点看做一种消极的设备，而忽视了它们
智能和自主的能力。我们提出RADOS的设计与实现，它可以利用存在于单个存储节点的智能性将可靠的对象存储服务扩展到成千上万的设备。RADOS保持一致的数据
访问和强安全语义，并使用集群映射(cluster map)支持节点半自治地管理副本,检测失败及进行故障恢复。本文的实现提供了出色的性能,可靠性和可扩展性，使得
用户有一种只有一个逻辑对象存储的错觉。

<h4>分类和主题描述</h4>

C.4 [系统新能]: 可靠性，可用性，可服务性; D.4.3 [文件系统管理]: 分布式文件系统; D.4.7 [机构和设计]: 分布式系统

<h4>通用术语</h4>

设计，性能，可靠性

<h4>关键字</h4>

集群存储，PB级分布存储，对象存储

<h4>1. 介绍</h4>

提供可靠、高效的分布式存储是一个一直困扰系统设计者的挑战。为文件系统、数据库以及相关抽象层提供高吞吐、低延迟的存储对诸多应用的性能至关重要。
新兴的从存储块或对象存储设备（OSDs）构建的集群存储架构试图将低层级的块分配抉择和安全实施分配到智能的存储设备，简化了数据的陈列并通过用户直接
获取数据以消除I/O瓶颈[1, 6, 7, 8, 21, 27, 29, 31]. OSD

{% highlight ruby %}
{% endhighlight %}

[sage]: https://en.wikipedia.org/wiki/Sage_Weil
[rados]: http://ceph.com/papers/weil-rados-pdsw07.pdf
