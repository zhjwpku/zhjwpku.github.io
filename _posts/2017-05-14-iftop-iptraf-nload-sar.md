---
layout: post
title: 网络测试工具之 — iftop、iptraf、nload、sar
date: 2017-5-14 15:00:00 +0800
tags:
- iftop
- iptraf
- nload
- sar
---

在 [网络测试工具之 — tcpping、hping、mtr][ref] 一文中，笔者介绍了使用测试网络延迟的三个工具。本篇介绍四个监测网络流量的工具。

<h4>iftop</h4>

安装：

```shell
[root@alice ~]# yum install -y iftop
```

运行：

```shell
[root@alice ~]# iftop -i em1
```

![iftop](/assets/201705/iftop.png)

<h4>iptraf</h4>

安装：

```shell
[root@alice ~]# yum install -y iptraf
```

运行：

```shell
[root@alice ~]# iptraf-ng
```

![iptraf-ng](/assets/201705/iptraf-ng.png)

**IP traffic monitor**

![ip-traffic](/assets/201705/iptraf-ip-traffic.png)

**General interface statistics**

![interfaces](/assets/201705/iptraf-interfaces.png)

**Detailed interface statistics**

![detaied-interface](/assets/201705/iptraf-detailed-interface.png)


<h4>nload</h4>

安装：

```shell
[root@alice ~]# yum install -y nload
```

运行：

```shell
[root@alice ~]# nload
```

![nload](/assets/201705/nload.png)

通过点击左右键来切换不同的网络接口。

<h4>sar</h4>

sar 包含在 sysstat 工具集中，存在该包的工具还有 iostat, mpstat。

安装：

```shell
[root@alice ~]#  yum install -y sysstat
```

运行：

```shell
# 每间隔一秒打印一个结果，统共打印十次
[root@alice ~]# sar -n DEV 1 10
# 每间隔两秒打印一个结果，知道 Ctrl+C 结束
[root@alice ~]# sar -n DEV 2
```

![sar](/assets/201705/sar.png)

**总结**

以上四个工具都能测出实时的网络流量，个人比较喜欢 iftop，从 manpage 来看好像也更强大。不过以上工具貌似对回看支持的不好，比如想把几天的结果都存储在数据库里，然后再集中分析查看，如果是这种用例，推荐使用 ntop，见 [How to setup ntop on Centos 7](https://hostingwikipedia.com/setup-ntop-centos-7/)。

[ref]: /2016/12/17/tcpping-hping-mtr.html
