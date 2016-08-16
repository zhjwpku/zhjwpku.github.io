---
layout: post
title:  "Kubernetes学习备忘"
date:   2016-08-15 19:30:00 +0800
categories: docker
tags:
- k8s
- docker
---

由于项目需要，现开始研究Kubernetes。本文记录从2016.8.15日开始到2016.8.19日学习Kubernetes的要点和心得，
希望通过一周的学习，能对Kubernetes有一个大致的了解。

**Kubernetes是什么**

Kubernetes是Google开源的容器集群管理系统。它构建在Docker技术之上，为容器化的应用提供资源调度、部署运
行、服务发现、扩容缩容等一整套功能。Google从2004年开始使用容器技术，于2006年发布Cgroup，Google内部开
发的集群资源管理平台Borg和Omega

{% highlight ruby %}
from graphviz import Digraph

colors = {"GH":"lightblue2", "ZA":"deepskyblue", "GB":"goldenrod2",
          "EU":"burlywood2", "RW":"lightsteelblue", "US":"coral3",
	  "IN":"darkorange1", "IE":"cyan", "NG":"firebrick",
	  "BR":"blue", "PT":"gold1", "NL":"darkolivegreen3",
	  "FR":"bisque4", "EG":"steelblue3"}

{% endhighlight %}

节点的颜色参照[Graphviz_color][graphviz_color]来设置。
[graphviz]: https://pypi.python.org/pypi/graphviz
[graphviz_color]: http://www.graphviz.org/content/crazy
