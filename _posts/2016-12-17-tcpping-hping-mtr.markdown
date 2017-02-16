---
layout: post
title: 网络测试工具之 — tcpping、hping、mtr
date: 2016-12-17 21:00:00 +0800
tags:
- tcpping
- hping
- mtr
---

本文介绍三个网络测试工具，分别是 [tcpping][tcpping]、[hping][hping] 和 [mtr][mtr]。

<h4>tcpping</h4>

`ping` 通过发送 ICMP 消息来测试网络 RTT (Round-Trip Time)，但是因特网中的路由器可能会设置防火墙禁止 ICMP，即使没有被墙，在网络状况很差的情况下，路由器或主机会丢弃 ICMP 消息而优先传输 TCP包。

`tcpping` 工具工作在 TCP 层，通过发送伪造的 TCP SYN 包并侦听来自服务器或中间设备返回的 SYN/ACK 或 RST 。代码不到1000行，调用 libpcap 和 libnet 提供的接口，打印与 `ping` 近乎相同测试结果。

*tips: 使用 nmap 进行端口扫面，然后对扫描到端口进行 tcpping 测试。open 的端口返回 SYN/ACK，closed 的端口返回 RST。*

**Install**

{% highlight shell %}
# Install depencies
$ sudo apt-get install build-essential
$ sudo apt-get install libnet1-dev
$ sudo apt-get install libpcap-dev
$ sudo apt-get install xmltoman

# Build and install
$ git clone https://github.com/jwyllie83/tcpping.git
$ cd tcpping
$ make
$ sudo make install
{% endhighlight %}

**Usage**

{% highlight shell %}
$ man tcpping

tcpping(1)                           General Commands Manual                          tcpping(1)

NAME
    tcpping - ping(8) written using TCP SYN probes

SYNOPSIS
    tcpping [-v] [-c count] [-p port] [-i interval] [-I interface] [-t ttl] [-S srcaddress]
    remote_host

DESCRIPTION
    tcpping(1) is a utility designed to emulate standard ping(8) in nearly every meaningful way
    and only diverge when necessary. It sends out  forged  TCP  SYN  packets  and listens for a
    SYN/ACK, RST, or ICMP Time Exceeded message.  It counts and  reports on these results using
    an interface that is nearly identical to standard UNIX ping.

    tcpping(1) works well in situations where ICMP messages are either thought to be less resp-
    onsive (through ICMP rate-limiting) or filtered entirely via firewalls.

OPTIONS
    -v      Display more verbose output

    -c COUNT
            Send COUNT packets and exit

    -p PORT
            Send packets to PORT instead of TCP port 80

    -i INTERVAL
            Wait INTERVAL seconds between packets (can be a decimal). Defaults to 1s

    -I INTERFACE
            Send packets from, and probe for responses on, the given INTERFACE. Defaults to the
            first external UP interface though it is not very robust.

    -t TTL  Set TTL as the IP TTL for the probes. Defaults to "sufficiently high"

    -S SRCADDRESS
            Set SRCADDRESS as the source address instead of the default IP of INTERFACE

SECURITY
    tcpping(1)  requires the CAP_NET_RAW capability and is therefore installed as set-uid root.
    Though numerous steps are taken to ensure safety here (clearing the environment, safe input
    checks) there is always some inherent risk.

    It should also be noted that TCP SYN packets can  overwhelm and  crash  some servers as TCP
    SYN packets  yielding  a SYN/ACK  will typically  allocate resources on the server. Issuing
    this command with a very short interval to a server listening on that port is effectively a
    SYN flood which the server may or may not handle gracefully.

    More information about SYN floods can be found here: http://en.wikipedia.org/wiki/SYN_flood

...
{% endhighlight %}

<h4>hping</h4>

`hping3` 是一个能够发送自定义 TCP/IP 包（报文内容、包大小）并显示目标回复的网络工具，它甚至能够在支持的协议下传输文件。你可以用 hping3 进行如下操作：

- 防火墙测试
- 高级端口扫描
- 使用不同的协议、包大小、TOS(type of servie)及IP分片进行网络性能的测试
- 发现路径的 MTU
- 在严格的防火墙环境下传输文件
- 多协议路由跟踪
- Firewalk-like usage
- 远程操作系统探测
- TCP/IP 栈审计
- ... etc.

**Install**

{% highlight shell %}
# Ubuntu 16.04
$ sudo apt-get install hping3

# CentOS 7
$ sudo yum install -y hping3.x86_64
{% endhighlight %}

**Usage**

{% highlight shell %}
# hping3 默认发送TCP报文到目标及的0号端口
$ sudo hping3 -c 4 -S -p 80 baidu.com
HPING baidu.com (enp0s31f6 180.149.132.47): S set, 40 headers + 0 data bytes
len=46 ip=180.149.132.47 ttl=51 id=23105 sport=80 flags=SA seq=0 win=8192 rtt=7.8 ms
len=46 ip=180.149.132.47 ttl=51 id=43051 sport=80 flags=SA seq=1 win=8192 rtt=3.7 ms
len=46 ip=180.149.132.47 ttl=51 id=26740 sport=80 flags=SA seq=2 win=8192 rtt=3.6 ms
len=46 ip=180.149.132.47 ttl=51 id=37006 sport=80 flags=SA seq=3 win=8192 rtt=3.5 ms

--- baidu.com hping statistic ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 3.5/4.7/7.8 ms

# 设置包大小
$ sudo hping3 -c 4 -S -p 80 -d 800 baidu.com
HPING baidu.com (enp0s31f6 220.181.57.217): S set, 40 headers + 800 data bytes
len=46 ip=220.181.57.217 ttl=51 id=65530 sport=80 flags=SA seq=0 win=512 rtt=3.6 ms
len=46 ip=220.181.57.217 ttl=51 id=39618 sport=80 flags=SA seq=1 win=512 rtt=3.3 ms
len=46 ip=220.181.57.217 ttl=51 id=16599 sport=80 flags=SA seq=2 win=512 rtt=3.0 ms
len=46 ip=220.181.57.217 ttl=51 id=7578 sport=80 flags=SA seq=3 win=512 rtt=2.8 ms

--- baidu.com hping statistic ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 2.8/3.2/3.6 ms

# -1 指定使用ICPM协议进行测试
$ sudo hping3 -1 -c 4 -S -p 80 baidu.com
HPING baidu.com (enp0s31f6 180.149.132.47): icmp mode set, 28 headers + 0 data bytes
len=46 ip=180.149.132.47 ttl=51 id=29309 icmp_seq=0 rtt=7.7 ms
len=46 ip=180.149.132.47 ttl=51 id=45877 icmp_seq=1 rtt=3.6 ms
len=46 ip=180.149.132.47 ttl=51 id=11526 icmp_seq=2 rtt=3.3 ms
len=46 ip=180.149.132.47 ttl=51 id=27752 icmp_seq=3 rtt=3.3 ms

--- baidu.com hping statistic ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 3.3/4.5/7.7 ms

# 端口扫面 --scan
$ sudo hping3 -8 1-1000,8888,known -S baidu.com
Scanning baidu.com (180.149.132.47), port 1-1000,8888,known
1193 ports to scan, use -V to see all the replies
+----+-----------+---------+---+-----+-----+-----+
|port| serv name |  flags  |ttl| id  | win | len |
+----+-----------+---------+---+-----+-----+-----+
80 http       : .S..A...  51 10036  8192    46
443 https      : .S..A...  51 47048  8192    46
All replies received. Done.
Not responding ports: (1 tcpmux) (2 nbp) (3 ) (4 echo) (5 ) (6 zip) (7 echo) (8 ) (9 discard) (10 ) 
(11 systat) (12 ) (13 daytime) (14 ) (15 netstat) (16 ) (17 qotd) (18 msp) (19 chargen) (20 ftp-data) 
(21 ftp) (22 ssh) (23 telnet) (24 ) (25 smtp) ...

# traceoute using ICMP
$ sudo hping3 --traceroute -V -1 baidu.com
using enp0s31f6, addr: 10.0.63.179, MTU: 1500
HPING baidu.com (enp0s31f6 123.125.114.144): icmp mode set, 28 headers + 0 data bytes
hop=1 TTL 0 during transit from ip=10.0.63.2 name=UNKNOWN   
hop=1 hoprtt=3.6 ms
hop=2 TTL 0 during transit from ip=124.65.173.193 name=UNKNOWN   
hop=2 hoprtt=7.5 ms
hop=3 TTL 0 during transit from ip=124.65.244.17 name=UNKNOWN   
hop=3 hoprtt=3.4 ms
hop=4 TTL 0 during transit from ip=61.148.7.253 name=UNKNOWN   
hop=4 hoprtt=7.2 ms
hop=5 TTL 0 during transit from ip=124.65.60.117 name=UNKNOWN   
hop=5 hoprtt=6.1 ms
hop=6 TTL 0 during transit from ip=123.126.8.178 name=UNKNOWN   
hop=6 hoprtt=138.0 ms
hop=7 TTL 0 during transit from ip=123.125.248.90 name=UNKNOWN   
hop=7 hoprtt=4.2 ms
... 卡死在这了，不知道为啥 ...

# '-s 5050'表示使用本地5050端口进行TCP扫描
$ sudo hping3 --traceroute -V -S -p 80 -s 5050 baidu.com
{% endhighlight %}

<h4>MTR</h4>

`mtr` 在单个网络诊断工具中集成了`traceroute`和`ping`。当它启动时，它会监测运行mtr的主机和目标主机之间的网络连接。在确定机器之间的每个网络跳的地址之后，它向每个机器发送序列ICMP ECHO请求，以确定到每个机器的链路的质量，并打印每一跳的运行统计。与 hping 类似，它也可以发送 TCP SYN 或 UDP 请求。

**Install**

{% highlight shell %}
# Ubuntu 16.04
$ sudo apt-get install mtr

# CentOS 7
$ sudo yum install -y mtr
{% endhighlight %}

**Usage**

{% highlight shell %}
$ mtr -r -c 10 baidu.com
Start: Thu Dec 29 19:40:37 2016
HOST: ubuntu                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1.|-- 10.0.63.2                  0.0%    10    0.5   0.5   0.4   0.6   0.0
 2.|-- 124.65.173.193             0.0%    10    9.4   4.6   3.5   9.4   1.7
 3.|-- 124.65.244.17              0.0%    10    1.2   1.2   0.9   1.5   0.0
 4.|-- 61.148.7.253               0.0%    10    4.3   3.8   2.6   4.3   0.0
 5.|-- 202.96.12.45               0.0%    10    5.0   4.3   1.5   5.3   1.1
 6.|-- 202.96.12.77               0.0%    10    2.0   3.3   1.7   6.2   1.6
 7.|-- 219.158.4.154             10.0%    10    1.8   3.0   1.6   5.3   1.3
 8.|-- 219.158.40.178            90.0%    10    5.8   5.8   5.8   5.8   0.0
 9.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0
10.|-- 180.149.128.114           50.0%    10    2.8   3.0   2.8   3.3   0.0
11.|-- 180.149.128.130            0.0%    10    6.2 257.6   5.6 871.4 357.9
12.|-- 180.149.129.2              0.0%    10    3.3   2.8   2.6   3.3   0.0
13.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0
14.|-- 180.149.132.47             0.0%    10    2.5   2.5   2.2   2.8   0.0

# -C 表示以CSV格式输出，-n 表示不进行hostname的解析
$ mtr -r -C -c 10 baidu.com
MTR.0.86;1483012433;OK;baidu.com;1;10.0.63.2;423
MTR.0.86;1483012433;OK;baidu.com;2;124.65.173.193;93129
MTR.0.86;1483012433;OK;baidu.com;3;124.65.244.17;1111
MTR.0.86;1483012433;OK;baidu.com;4;61.148.7.117;778
MTR.0.86;1483012433;OK;baidu.com;5;61.51.112.225;4792
MTR.0.86;1483012433;OK;baidu.com;6;124.65.194.93;5538
MTR.0.86;1483012433;OK;baidu.com;7;219.158.6.42;4917
MTR.0.86;1483012433;OK;baidu.com;8;219.158.38.242;103262
MTR.0.86;1483012433;OK;baidu.com;9;???;0
MTR.0.86;1483012433;OK;baidu.com;10;???;0
MTR.0.86;1483012433;OK;baidu.com;11;???;0
MTR.0.86;1483012433;OK;baidu.com;12;220.181.17.146;103209
MTR.0.86;1483012433;OK;baidu.com;13;???;0
MTR.0.86;1483012433;OK;baidu.com;14;???;0
MTR.0.86;1483012433;OK;baidu.com;15;220.181.57.217;101357
{% endhighlight %}

本文仅介绍了 hping3 和 mtr 工具简单的使用方法，后面遇到高级用法会持续进行更新。


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Testing firewall rules with Hping3 - examples][hping3example].<br>
</span>

[tcpping]: https://github.com/jwyllie83/tcpping
[hping]: https://github.com/antirez/hping
[mtr]: https://github.com/traviscross/mtr
[hping3example]: http://0daysecurity.com/articles/hping3_examples.html
