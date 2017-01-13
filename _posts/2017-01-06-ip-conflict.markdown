---
layout: post
title: IP 冲突解决方案
date: 2017-01-06 22:00:00 +0800
tags:
- arping
---

最近在公司办公的时候经常连接不上 ssh 服务器，连接被拒绝，使用 `arping` 工具查看是否有 IP 冲突。

{% highlight shell %}
## arping - send ARP REQUEST to a neighbour host
[root@boba ~]# arping -I em1  10.0.63.202
ARPING 10.0.63.202 from 10.0.63.203 em1
Unicast reply from 10.0.63.202 [XX:18:7X:X5:BA:D0]  0.641ms
Unicast reply from 10.0.63.202 [YY:02:FY:Y5:7E:36]  174.195ms
Unicast reply from 10.0.63.202 [YY:02:FY:Y5:7E:36]  144.569ms
{% endhighlight %}

可以看出有两个网络设备占用了 10.0.63.202 这个 IP，那么如何将自己额设备绑定到这个特定的 IP 呢？

**方法一**

如果拥有网关的登陆权限，则可以在网关上将 IP 和 设备的网络设备 MAC 值进行绑定，如果能够设置网关的 ARP 表项，则可将已有的动态绑定 IP/MAC 对转换为静态绑定并将其删除。

**方法二**

如果没有网关的登陆权限，则可使用 `arping` 的 -U 选项主动向网关上传自己的 IP/MAC 对以更细邻居的 ARP 缓存。

{% highlight shell %}
[root@c3po ~]# arping -Uq -s 10.0.63.204 -I em1 10.0.63.2 &
{% endhighlight %}

*注：不建议使用第二种方法，因为 ARP 是以广播的方式发包*

但是至此，依然很有可能连接不上 SSH 服务器，因为你的笔记本上可能还保存着之前的 ARP 表项。在 Window 下更改 ARP 地址转换表。使用管理员权限运行 `cmd.exe`。

{% highlight shell %}
## 显示存在的ARP表项
C:\Windows\system32>arp -a
## 删除已存在的动态绑定
C:\Windows\system32>arp -d 10.0.63.204
## 添加静态绑定
C:\Windows\system32>arp -s 10.0.63.204 XX:18:7X:X8:50:d7
{% endhighlight %}

*注：本文不保证一定能解决IP冲突*
