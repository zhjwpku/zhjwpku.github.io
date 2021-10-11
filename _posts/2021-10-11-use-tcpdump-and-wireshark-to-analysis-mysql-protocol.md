---
layout: post
title: 使用 tcpdump 和 wireshark 分析 MySQL 协议握手过程
date: 2021-10-11 00:00:00 +0800
tags:
- mysql
- tcpdump
- wireshark
---

使用 tcpdump 对 MySQL 进行抓包是一种定位线上问题的重要手段，如结合 [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html) 识别慢查询；使用 [wireshark](https://gitlab.com/wireshark/wireshark) 对报文内容进行分析，这对很多 *MySQL Compatible* 的产品非常重要。本文使用 tcpdump 和 wireshark 对 MySQL 的握手过程进行抓包分析。

**实验环境**

```
OS: macOS Big Sur
MySQL version: 8.0.25
tcpdump version tcpdump version 4.9.3 -- Apple version 100
libpcap version 1.9.1
LibreSSL 2.8.3
Wireshark Version 3.4.8
```

#### Step 1 抓包

MySQL server 监听默认端口号 3306，使用如下命令抓包

```shell
➜  ~ sudo tcpdump -X -i any src port 3306 or dst port 3306 -w tcpdump_3306_"`date +"%Y-%m-%d-%s"`".pcap
tcpdump: data link type PKTAP
tcpdump: listening on any, link-type PKTAP (Apple DLT_PKTAP), capture size 262144 bytes
```

使用 `Ctrl + C` 即可停止抓包，查看生成的包文件

```shell
➜  ~ ll
total 32
-rw-r--r--  1 root  staff    15K Oct 11 14:44 tcpdump_3306_2021-10-11-1633934437.pcap
```

#### Step 2 使用 wireshark 进行分析

首先对 client 连接服务端请求进行分析，对应的命令为:

```
➜  ~ mysql -ucanal -pcanal -h127.0.0.1
```

使用 wireshark 查看结果:

![mysql client](/assets/2021/mysql_client.jpg)

图中基于 Protocol 列进行排序（忽略过多的 TCP ACK 消息），可以看出，TCP 握手成功后，服务端首先发送了 Greeting 消息（No.9），附带了协议版本和服务器版本，之后客户端发送了 Login Request，但由于 MySQL 客户端和服务器之间默认使用信道加密，后面的消息都为 TLSv1.3，因此这些消息都为密文。

使用非加密的方式重新进行连接:

```
➜  ~ mysql -ucanal -pcanal -h127.0.0.1 --ssl-mode=DISABLED
```

使用 wireshark 查看结果：

![mysql client --ssl-mode=DISABLE](/assets/2021/mysql_client_disable_ssl.jpg)

同样基于 Protocol 列排序，可以看到现在 MySQL 的交互清晰展现了出来，下面详细看下各个包的内容。

**Frame 9**

![Frame 9](/assets/2021/mysql_server_greeting.jpg)

TCP 连接建立之后，服务器首先给客户端发送 Greeting 消息，告诉客户端自己的版本、分配的连接ID、服务器的能力（Capabilities）、认证插件等信息。可以理解为 MySQL 的握手是由 Server 端发起的。

**Frame 13**

![Frame 13](/assets/2021/mysql_client_login_request.jpg)

客户端将登录信息（username/passwd）及客户端具有的能力、最大发送包大小等信息发送给服务端。

**Frame 17**

![Frame 17](/assets/2021/mysql_server_auth_switch_request.jpg)

Auth Switch Request 的解释参照 [Protocol::AuthSwitchRequest](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::AuthSwitchRequest)。

**Frame 21**

![Frame 21](/assets/2021/mysql_server_response.jpg)

服务器对用户发送来的用户名密码进行验证，返回验证成功的消息，该消息也意味着 MySQL 协议握手的结束。

**Frame 25**

![Frame 25](/assets/2021/mysql_client_send_request.jpg)

客户端在握手成功之后发起了一个 sql 请求，wireshark 对该消息的 payload 未能解析出来，但从包的二进制内容中可以看出，客户端进行了如下请求:

```sql
select @@version_comment limit 1;
```

**Frame 29**

![Frame 29](/assets/2021/mysql_server_response_homebrew.jpg)

wireshark 对服务器的响应也存在一些问题，不过从二进制内容后面对应的 ASCII 码中可以看出，服务器返回了 `Homebrew`。在客户端直接查询进行验证:

![select version comment](/assets/2021/mysql_select_version_comment.jpg)

至此，mysql 连接的过程分析完毕。附上本文分析的 pcap 文件: [tcpdump_3306_2021-10-11-1633935657.pcap](/assets/2021/tcpdump_3306_2021-10-11-1633935657.pcap)

#### 一些技巧

1. 使用 Wireshark 的 TCP Stream 可以跟踪一个命令所有交互的包，方便在一堆包中定位需要的信息:

![mysql tcp stream](/assets/2021/mysql_wireshark_tcp_stream.jpg)

![mysql tcp stream result](/assets/2021/mysql_wireshark_tcp_stream_result.jpg)

2. wireshark 根据端口号识别响应的协议，假设 MySQL server 监听在 3303，则需要进行如下设置:

![wireshark mysql tcp port setting](/assets/2021/mysql_wireshark_tcp_port.jpg)

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [tcpdump Cheat Sheet](https://cdn.comparitech.com/wp-content/uploads/2019/06/tcpdump-cheat-sheet-1.pdf)<br>
2 [Use tcpdump to monitor mysql](https://gist.github.com/gstark/10268260)<br>
3 [MySQL Internals Manual 14.2.5 Connection Phase Packets](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html)
</span>
