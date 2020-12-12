---
layout: post
title: MySQL 源码解读 —— Binlog 事件格式解析
date: 2020-11-01 08:00:00 +0800
tags:
- mysql
- binlog
---

Binary Log(i.e. Binlog) 是一组包含了 MySQL 数据库实例数据修改信息的日志文件，主要用于 **主从复制(Replication)** 和 **特定的数据恢复操作**，是否开启由 `log-bin` 选项（Read Only）控制。本文对 Binlog 事件的格式进行解析，为深入理解 Binlog 的工作机理奠定基础。

#### binlog_format

Binlog 有两种日志格式:

1. Statement-based logging(SBL): 基于 SQL 语句的日志记录，事件包含产生数据更改（插入，更新，删除）
2. Row-based logging(RBL): 基于行的日志记录，事件描述对单个行的更改

上述两种格式各有优势(SBL 日志量小，节省I/O；RBL 能保证正确性)，为了兼顾**性能**和**正确性**，MySQL 提供了 `Mixed logging`: 默认情况下使用 SBL，并根据需要自动切换到 RBL。

有些文章将 RBL 称为 row-based replication(RBR)，SBL 称为 statement-based replication(SBR)，Mixed logging 称为 mixed-based replication(MBR)。严格地讲这有点用词不当，因为日志的记录过程可以独立于主从复制过程，详见 [Row-Based Binary Logging][row-based-binary-logging]。

#### Row-based logging

RBL 由 MySQL 5.1 版本引进，*可能* 是使用最广泛的日志格式，本文主要介绍这种日志格式。

```
mysql> show variables like '%binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.01 sec)
```

*在 RBL 中，有些更改仍然使用 SBL，如 CREATE TABLE，ALTER TABLE 或 DROP TABLE*

#### [Binary_log_event][Binary_log_event]

Binlog 是以一个个事件存储在日志文件中的，所有事件都继承自 Binary_log_event，该类规定了事件的基本格式:

- Common-Header: 所有事件都具有的头，长度均为 19 字节
- Post-Header: 特定事件的头，不同事件头的长度定义在 `enum_post_header_length` 中定义
- Body: 事件内容
- Footer: 事件结尾

Common-Header 由 [Log_event_header][Log_event_header] 定义，其各字段分别为:

| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| when | 4 字节无符号整型, 表示为 struct timeval | 查询开始的时间，自1970年以来的秒数
| type_code | 1字节 | 事件类型，见 [Log_event_type][Log_event_type]
| unmasked_server_id | 4 字节无符号整型 | 创建该事件的 server id
| data_written | 4 字节无符号整型 | 该事件的总长度，以字节为单位。包含 Common-Header，Post-Header 和 Body
| log_pos | 4 字节无符号整型 | 下一个事件在 master 节点 binlog 中的位置，距离文件开头的字节数
| flags | 2 字节 | 取决于 binlog 版本，表示不多于 16 个标志

<br>
下面我们来看一个刚部署的 MySQL 实例生成的 binlog 文件内容，然后据此来反观代码的实现。

```
root@ubuntu-bionic:/data/data# file binlog.000001
binlog.000001: MySQL replication log, server id 1 MySQL V5+, server version 8.0.22
root@ubuntu-bionic:/data/data# stat binlog.000001
  File: binlog.000001
  Size: 475             Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d      Inode: 796591      Links: 1
Access: (0640/-rw-r-----)  Uid: (  999/   mysql)   Gid: ( 1002/   mysql)
Access: 2020-11-01 06:06:56.066493368 +0000
Modify: 2020-11-01 06:04:20.190493368 +0000
Change: 2020-11-01 06:04:20.190493368 +0000
 Birth: -
```

使用 `hexdump` 查看 binlog.000001 文件的这 475 个字节都对应了哪些事件:

```
root@ubuntu-bionic:/data/data# hd -v binlog.000001
00000000  fe 62 69 6e 86 4e 9e 5f  0f 01 00 00 00 79 00 00  |.bin.N._.....y..|
00000010  00 7d 00 00 00 01 00 04  00 38 2e 30 2e 32 32 00  |.}.......8.0.22.|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 86 4e 9e 5f 13  |............N._.|
00000050  00 0d 00 08 00 00 00 00  04 00 04 00 00 00 61 00  |..............a.|
00000060  04 1a 08 00 00 00 08 08  08 02 00 00 00 0a 0a 0a  |................|
00000070  2a 2a 00 12 34 00 0a 28  01 23 37 87 52 86 4e 9e  |**..4..(.#7.R.N.|
00000080  5f 23 01 00 00 00 1f 00  00 00 9c 00 00 00 80 00  |_#..............|
00000090  00 00 00 00 00 00 00 00  0b 4d b2 0c e4 4f 9e 5f  |.........M...O._|
000000a0  22 01 00 00 00 4f 00 00  00 eb 00 00 00 00 00 01  |"....O..........|
000000b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000c0  00 00 00 00 00 00 00 00  02 00 00 00 00 00 00 00  |................|
000000d0  00 01 00 00 00 00 00 00  00 e0 b9 8b 66 05 b3 05  |............f...|
000000e0  fc 3f 01 96 38 01 00 dd  ad db 17 e4 4f 9e 5f 02  |.?..8.......O._.|
000000f0  01 00 00 00 f0 00 00 00  db 01 00 00 00 00 0c 00  |................|
00000100  00 00 00 00 00 00 00 00  00 3a 00 00 00 00 00 00  |.........:......|
00000110  01 20 00 a0 45 00 00 00  00 06 03 73 74 64 04 ff  |. ..E......std..|
00000120  00 ff 00 ff 00 05 06 53  59 53 54 45 4d 0c 01 6d  |.......SYSTEM..m|
00000130  79 73 71 6c 00 0d bc e6  02 11 03 00 00 00 00 00  |ysql............|
00000140  00 00 12 ff 00 00 41 4c  54 45 52 20 55 53 45 52  |......ALTER USER|
00000150  20 27 72 6f 6f 74 27 40  27 6c 6f 63 61 6c 68 6f  | 'root'@'localho|
00000160  73 74 27 20 49 44 45 4e  54 49 46 49 45 44 20 57  |st' IDENTIFIED W|
00000170  49 54 48 20 27 63 61 63  68 69 6e 67 5f 73 68 61  |ITH 'caching_sha|
00000180  32 5f 70 61 73 73 77 6f  72 64 27 20 41 53 20 27  |2_password' AS '|
00000190  24 41 24 30 30 35 24 08  70 25 13 40 41 3e 3d 59  |$A$005$.p%.@A>=Y|
000001a0  2b 77 1f 21 5d 3d 4b 38  7d 7d 5b 43 42 70 6c 32  |+w.!]=K8}}[CBpl2|
000001b0  76 47 46 49 77 43 69 46  78 6b 6c 4d 2f 61 77 39  |vGFIwCiFxklM/aw9|
000001c0  65 44 65 54 37 39 51 68  6f 48 35 35 41 4a 38 51  |eDeT79QhoH55AJ8Q|
000001d0  37 33 71 6d 32 31 27 6e  42 a7 b5                 |73qm21'nB..|
000001db
```

**Magic Number**

binlog 文件以 `fe 62 69 6e` 四字节魔术字作为开头。

**第一个事件**

跳过魔术字，看 Common-Header:

```
root@ubuntu-bionic:/data/data# hd -s4 -n19 binlog.000001
00000004  86 4e 9e 5f 0f 01 00 00  00 79 00 00 00 7d 00 00  |.N._.....y...}..|
00000014  00 01 00                                          |...|
00000017
```

![FORMAT_DESCRIPTION_EVENT](/assets/2020/fde.png)

<h5 align="center">图1: First Event Common-Header</h5>

可以看出:

- 事件发生时间: 2020-11-01T05:58:30+00:00
- 事件类型: FORMAT_DESCRIPTION_EVENT
- server id: 1
- 事件长度: 121 字节
- 下一个字节起始位置: 125 字节，由于这是第一个事件，该值也很容易根据事件长度和 Magic Number 的长度推算出来 121 + 4 = 125

由 FORMAT_DESCRIPTION_EVENT 容易在 `enum_post_header_length` 中找到 `FORMAT_DESCRIPTION_HEADER_LEN` 为 97 字节，同时看 [Format_description_event][Format_description_event] 的 Post-Header 结构如下:


| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| binlog_version | 2 字节无符号整型 | binlog 版本
| server_version | 50 字节 | 服务器版本
| create_timestamp | 4 字节无符号整型，表示为 time_t | 创建该事件的时间
| common_header_len | 1 字节 | 所有事件 Common-Header 的长度，通常为 LOG_EVENT_HEADER_LEN，即 19
| post_header_len | 1 字节无符号整型数组(1 * 40) | 每个事件 post_header_len 的长度，根据 enum_post_header_length 进行初始化

<br>
使用 hd 查看这 97 字节数据:

```
root@ubuntu-bionic:/data/data# hd -v -s23 -n97 binlog.000001
00000017  04 00 38 2e 30 2e 32 32  00 00 00 00 00 00 00 00  |..8.0.22........|
00000027  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000037  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000047  00 00 00 00 86 4e 9e 5f  13 00 0d 00 08 00 00 00  |.....N._........|
00000057  00 04 00 04 00 00 00 61  00 04 1a 08 00 00 00 08  |.......a........|
00000067  08 08 02 00 00 00 0a 0a  0a 2a 2a 00 12 34 00 0a  |.........**..4..|
00000077  28                                                |(|
00000078
```
<div align="center">
  <img src="/assets/2020/fde_post_header.png"/>
</div>

<h5 align="center">图2: Format_description_event Post-Header</h5>

上面分析的 Format_description_event 的字节数为 19 + 97 = 116，还差 5 个字节:

```
root@ubuntu-bionic:/data/data# hd -v -s120 -n5 binlog.000001
00000078  01 23 37 87 52                                    |.#7.R|
0000007d
```

其中第一个字节 [enum_binlog_checksum_alg][enum_binlog_checksum_alg] = 1 代表了 binlog checksum 使用 BINLOG_CHECKSUM_ALG_CRC32 算法，后面四个字节即为 CRC32 的计算值。

**第二个事件**

Common-Header:

```
root@ubuntu-bionic:/data/data# hd -v -s125 -n19 binlog.000001
0000007d  86 4e 9e 5f 23 01 00 00  00 1f 00 00 00 9c 00 00  |.N._#...........|
0000008d  00 80 00                                          |...|
00000090
```
![PREVIOUS_GTIDS_LOG_EVENT](/assets/2020/PREVIOUS_GTIDS_LOG_EVENT.png)

<h5 align="center">图3: Second Event Common-Header</h5>

这是一个 PREVIOUS_GTIDS_LOG_EVENT 事件。

**第三个事件**

Common-Header:

```
root@ubuntu-bionic:/data/data# hd -v -s156 -n19 binlog.000001
0000009c  e4 4f 9e 5f 22 01 00 00  00 4f 00 00 00 eb 00 00  |.O._"....O......|
000000ac  00 00 00                                          |...|
000000af
```

![ANONYMOUS_GTID_LOG_EVENT](/assets/2020/ANONYMOUS_GTID_LOG_EVENT.png)

<h5 align="center">图4: Third Event Common-Header</h5>

这是一个 ANONYMOUS_GTID_LOG_EVENT 事件。

**第四个事件**

Common-Header:

```
root@ubuntu-bionic:/data/data# hd -v -s235 -n19 binlog.000001
000000eb  e4 4f 9e 5f 02 01 00 00  00 f0 00 00 00 db 01 00  |.O._............|
000000fb  00 00 00                                          |...|
000000fe
```

![QUERY_EVENT](/assets/2020/QUERY_EVENT.png)

<h5 align="center">图5: Fourth Event Common-Header</h5>

这是一个 QUERY_EVENT 事件。其 log_pos 即下一个事件的起始位置与 binlog 的文件大小相同，说明这是最后一个事件。

通过 `show binlog events` 来查看所发生过的事件：

```
mysql> show binlog events;
+---------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                          |
+---------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------+
| binlog.000001 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.22, Binlog ver: 4                                                                                                                             |
| binlog.000001 | 125 | Previous_gtids |         1 |         156 |                                                                                                                                                               |
| binlog.000001 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                          |
| binlog.000001 | 235 | Query          |         1 |         475 | ALTER USER 'root'@'localhost' IDENTIFIED WITH 'caching_sha2_password' AS '$A$005p%@A>=Y+w!]=K8}}[CBpl2vGFIwCiFxklM/aw9eDeT79QhoH55AJ8Q73qm21' /* xid=3 */ |
+---------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

与通过 hexdump 分析 binlog 文件所得到事件相同。观察上面的结果还能得出一个结论：Anonymous_Gtid 事件 和 Query 事件都是由 `ALTER USER` 操作触发的，因为事件的时间戳相同。

**Sysbench write-only Flamegraph**

对新部署的 MySQL 实例（没有任何业务）所产生的 binlog 进行分析能让人对事件的格式有一个直观的感受，因为事件少方便分析。但实际业务中，对于 RBL 来讲，最多的事件可能是如下四种:

- [TABLE_MAP_EVENT][TABLE_MAP_EVENT]
- [WRITE_ROWS_EVENT][WRITE_ROWS_EVENT]
- [UPDATE_ROWS_EVENT][UPDATE_ROWS_EVENT]
- [DELETE_ROWS_EVENT][DELETE_ROWS_EVENT]

通过 Sysbench 的 write-only 来对 MySQL 下发业务，同时对 mysqld 进行性能取样，生成火焰图。

```
root@ubuntu-bionic:~# sysbench --threads=4 --time=600 --report-interval=10 --db-driver=mysql│  --db-debug[=on|off] print database-specific debug information [off]
 --mysql-host=127.0.0.1 --mysql-user=root --mysql-password=huawei@123 oltp_write_only --tabl│
es=4 --table-size=1000000 prepare

root@ubuntu-bionic:~# sysbench --threads=4 --time=600 --report-interval=10 --db-driver=mysql│udev            3.9G     0  3.9G   0% /dev
 --mysql-host=127.0.0.1 --mysql-user=root --mysql-password=huawei@123 oltp_write_only --tabl│tmpfs           798M  612K  798M   1% /run
es=4 --table-size=1000000 run
```

火焰图: [sysbench-write-only-with-binlog-on.svg](/assets/2020/sysbench-write-only-with-binlog-on.svg)

从火焰图中可以看出，[binlog_log_row][binlog_log_row] 占据了很可观的一部分计算资源，在后面的文章中，笔者将对 RBL 模式下主要事件的生成及写文件操作进行详细的分析。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [MySQL Internals: Binary Log Overview][binary-log-overview]<br>
2 [Setting The Binary Log Format][binary-log-setting]<br>
3 [Row-Based Binary Logging][row-based-binary-logging]<br>
4 [High-Level Binary Log Structure and Contents][binary-log-structure-and-contents]<br>
5 [SHOW BINLOG EVENTS Statement][show-binlog-events]<br>
</span>

[binary-log-overview]: https://dev.mysql.com/doc/internals/en/binary-log-overview.html
[binary-log-setting]: https://dev.mysql.com/doc/refman/8.0/en/binary-log-setting.html
[row-based-binary-logging]: https://dev.mysql.com/doc/internals/en/row-based-binary-logging.html
[binary-log-structure-and-contents]: https://dev.mysql.com/doc/internals/en/binary-log-structure-and-contents.html
[Binary_log_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L797
[Log_event_header]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L634
[Log_event_type]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L265
[Format_description_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/control_events.h#L230
[binlog_log_row]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/sql/handler.cc#L7684
[show-binlog-events]: https://dev.mysql.com/doc/refman/8.0/en/show-binlog-events.html
[enum_binlog_checksum_alg]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L410
[TABLE_MAP_EVENT]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L296
[WRITE_ROWS_EVENT]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L326
[UPDATE_ROWS_EVENT]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L327
[DELETE_ROWS_EVENT]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/binlog_event.h#L328