---
layout: post
title:  "使用Graphviz画网络运营商链路丢包率"
date:   2016-08-04 22:30:00 +0800
categories: others
tags:
- graphviz
---

假设现在有各个运营商之间的路径`edges = []`，列表的每一个元素都是诸如`（prev_isp, isp）`的元组，`nodes = dict()`是各运营商的数据字典，包括该点经过的次数、总丢包和国家，`edges_cnt = dict()`记录每条边经过的次数，`edges_loss = dict()`则记录每条边的目的节点的丢包个数。根据这些信息使用[Graphviz][graphviz]将运营商之间的链路质量展现出来。

{% highlight ruby %}
from graphviz import Digraph

colors = {"GH":"lightblue2", "ZA":"deepskyblue", "GB":"goldenrod2",
          "EU":"burlywood2", "RW":"lightsteelblue", "US":"coral3",
	  "IN":"darkorange1", "IE":"cyan", "NG":"firebrick",
	  "BR":"blue", "PT":"gold1", "NL":"darkolivegreen3",
	  "FR":"bisque4", "EG":"steelblue3"}

# {isp: [cnt, loss, country]}
nodes = dict()
# (prev_isp, isp)
edges = []
# {'%s%s' % (prev_isp, isp): edge_cnt}
edges_cnt = dict()
# {'%s%s' % (prev_isp, isp): edge_loss}
edges_loss = dict()

def generate_graph():
  u = Digraph('love', filename = 'love.gv')
  u.body.append('size = "10, 10"')
  u.node_attr(color='lightblue2', style='filled')
  
  # set all nodes color
  for k, v in nodes:
    u.node(k, color = colors[v[2]])

  # draw all edges
  edges.sort()
  for edge in edges:
    _key = "%s%s" % (edge[0], edge[1])
    # 在每条边上标出该线路的经过次数及丢包率
    _label = "%s loss: %.4f" % (edges_cnt[_key], (edges_loss[_key] / (edges_cnt[_key] * 4)))
    u.edge(edge[0], edge[1], label = _label)

  u.view()
{% endhighlight %}

[Graphviz][graphviz]还提供了很多种属性来画出各种不同的图形，如设置形状`u.attr('node', shape = 'box')`、画子图等功能，在之后的工作中如果遇到会陆续添加到本文中。

[graphviz]: https://pypi.python.org/pypi/graphviz
