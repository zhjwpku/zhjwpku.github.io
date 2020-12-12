---
layout: post
title: MySQL 源码解读 —— Binlog 行日志事件解析
date: 2020-11-07 10:00:00 +0800
tags:
- mysql
- binlog
---

在 [Binlog 事件格式解析][binlog-event-format] 一文中，笔者对 Format_description_event 的格式进行了解析，并在文末给出了 sysbench write-only test 的[火焰图][write-only-fg]。本文将对火焰图中 MySQL 的行日志事件进行解析。

#### FlameGraph

首先看下火焰图中的变更修改:

![handler::ha_write_row](/assets/2020/ha_write_row.png)

<h5 align="center">图 1: handler::ha_write_row</h5>

![handler::ha_update_row](/assets/2020/ha_update_row.png)

<h5 align="center">图 2: handler::ha_update_row</h5>

![handler::ha_delete_row](/assets/2020/ha_delete_row.png)

<h5 align="center">图 3: handler::ha_delete_row</h5>

如上三种操作分别对应 Write_rows_log_event, Update_rows_log_event 和 Delete_rows_log_event，除此之外，每种变更事件都伴随着一个 Table_map_log_event。

#### 事件的类结构

binlog 中事件的继承结构非常相似，通常都是 `*_event` 会有一个对应的 `*_log_event`, `*_event` 继承自 `Binary_log_event` 定义该事件的数据布局，`*_log_event` 则继承 `*_event` 和 `Log_event`，`Log_event` 定义了读写接口。如:

```
        Binary_log_event
               ^
               |
               |
    Table_map_event  Log_event
                \       /
                 \     /
                  \   /
           Table_map_log_event
```

特殊的是 Write_rows_log_event/Update_rows_log_event/Delete_rows_log_event 三种事件都是 row_based logging，其数据布局非常相似，主要区别在于 [row][row] 字段存储内容的不同:

- Write_rows_log_event 存储插入的数据
- Update_rows_log_event 存储修改前和修改后的数据
- Delete_rows_log_event 存储删除前的数据

binlog 为此抽象出一个 [Rows_log_event][Rows_log_event]，该事件满足上述的继承结构:

```
        Binary_log_event
               ^
               |
               |
         Rows_event  Log_event
                \       /
          <<vir>>\     /
                  \   /
              Rows_log_event
```

对于上面的三种变更事件，由于多了一层抽象，使得其继承结构变得有些复杂，如 Write_rows_log_event:

```
                         Binary_log_event
                                  ^
                                  |
                                  |
                                  |
                Log_event   B_l:Rows_event
                     ^            /\
                     |           /  \
                     |   <<vir>>/    \ <<vir>>
                     |         /      \
                     |        /        \
                     |       /          \
                  Rows_log_event    B_l:Write_rows_event
                             \          /
                              \        /
                               \      /
                                \    /
                                 \  /
                                  \/
                        Write_rows_log_event

  B_l: Namespace Binary_log
```

通过对三个变更事件菱形继承结构右侧的代码（如 Write_rows_event），发现将 Rows_log_event 实现为模板类会更简洁。

#### [Table_map_log_event][Table_map_log_event]

Table_map_log_event 记录了数据库表的元数据信息，包括数据库名，表名及字段类型等信息，具体的数据布局信息定义在 [Table_map_event][Table_map_event] 类:

| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| table_id | 6 字节无符号整型 | 表示该表的 id 值
| flags | 2 字节 | 代码注释中说是预留未来使用，其实现在已经使用了

<h6/>
<h5 align="center">表 1: Post-Header for Table_map_event</h5>

| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| database_name | 1 字节长度 + null 结尾字符串 | 因为行日志事件很多时候并不指定数据库名，所以在该事件中有必要记录数据库名。长度不含 null
| table_name | 1 字节长度 + null 结尾字符串 | 行日志事件所发生的表名
| column_count| Packed Integer (当有大量小数值的时候用于节省存储空间的一种编码格式，详见 [net_store_length][net_store_length]) | 数据表所含列数
| column_type | column_count 长度的字节数组 | 每字节代表一列数据的类型，枚举值见 [enum_field_types][enum_field_types]
| metadata_length | Packed Integer | 元数据长度
| metadata | metadata_length 个字节 | 有些列的类型需要存储元数据，详见[代码注释][Table_table_map_event_column_types]
| null_bits | 字节数组，数组长度 = int((column_count + 7) / 8) | 每一位表示对应列是否可以为 NULL
| optional metadata fields | Type(byte), Length(Packed Interge), Value(Length byte) | 详见[代码注释][Table_table_map_event_optional_metadata]

<h6/>
<h5 align="center">表 2: Body for Table_map_event</h5>

了解了 Table_map_event 的结构之后，下面看一个 binlog 文件中具体的表映射事件。

首先创建数据库并定义表结构:

```
create database zhjwpku;
use zhjwpku;
CREATE TABLE t
(
  id   INT NOT NULL,
  name VARCHAR(20) NOT NULL,
  date DATE NULL
) ENGINE = InnoDB;
```

写入一条数据 `INSERT INTO t VALUES(1, 'apple', NULL);` 然后观察 binlog 文件:

```
root@ubuntu-bionic:/data/data# mysqlbinlog --start-position=931646961 --hexdump binlog.000001
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 156
#201101  5:58:30 server id 1  end_log_pos 125 CRC32 0x52873723
# Position  Timestamp   Type   Master ID        Size      Master Pos    Flags
# 0000009c 86 4e 9e 5f   0f   01 00 00 00   79 00 00 00   7d 00 00 00   01 00
# 000000af 04 00 38 2e 30 2e 32 32  00 00 00 00 00 00 00 00 |..8.0.22........|
# 000000bf 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
# 000000cf 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
# 000000df 00 00 00 00 86 4e 9e 5f  13 00 0d 00 08 00 00 00 |.....N..........|
# 000000ef 00 04 00 04 00 00 00 61  00 04 1a 08 00 00 00 08 |.......a........|
# 000000ff 08 08 02 00 00 00 0a 0a  0a 2a 2a 00 12 34 00 0a |.............4..|
# 0000010f 28 01 23 37 87 52                                |...7.R|
#       Start: binlog v 4, server v 8.0.22 created 201101  5:58:30 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
hk6eXw8BAAAAeQAAAH0AAAABAAQAOC4wLjIyAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAACGTp5fEwANAAgAAAAABAAEAAAAYQAEGggAAAAICAgCAAAACgoKKioAEjQA
CigBIzeHUg==
'/*!*/;
# at 931646961
#201107 14:12:16 server id 1  end_log_pos 931647020 CRC32 0xda52b5cf
# Position  Timestamp   Type   Master ID        Size      Master Pos    Flags
# 3787cdf1 40 ab a6 5f   13   01 00 00 00   3b 00 00 00   2c ce 87 37   00 00
# 3787ce04 8c 00 00 00 00 00 01 00  07 7a 68 6a 77 70 6b 75 |.........zhjwpku|
# 3787ce14 00 01 74 00 03 03 0f 0a  02 50 00 04 01 01 00 02 |..t......P......|
# 3787ce24 03 fc ff 00 cf b5 52 da                          |......R.|
#       Table_map: `zhjwpku`.`t` mapped to number 140
```

使用 `mysqlbinlog` 可以很清楚的看出 Table_map_event 的 Common-Header 代表的各项，将之后的数据逐位解析，得到

- table_id: 8c 00 00 00 00 => 140
- flags: 01 00 => TM_BIT_LEN_EXACT_F
- database_name: 07 7a 68 6a 77 70 6b 75 00 => "zhjwpku"
- table_name: 01 74 00 => "t"
- column_count: 03 => 三列
- column_type: 03 0f 0a => 分别对应 MYSQL_TYPE_LONG、MYSQL_TYPE_VARCHAR、MYSQL_TYPE_DATE
- metadata_length: 02 => 只有 MYSQL_TYPE_VARCHAR 需要两字节的元数据
- metadata: 50 00 => 表示 varchar 最大长度为 80。定义表的时候定义的 varchar(20)，留个疑问后面确认
- null_bits: 04 => 第三字段可以为 NULL，即 date
- optional metadata fields:
  - SIGNEDNESS: 01 01 00 =>  所有字段都没有 unsign 标志
  - DEFAULT_CHARSET: 02 03 fc ff 00
- crc: cf b5 52 da

#### [Rows_log_event][Rows_log_event]

Rows_log_event 可以用来表示 Write_rows_log_event/Update_rows_log_event/Delete_rows_log_event 三种事件，其数据布局定义在 [Rows_event][Rows_event] 中:

| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| table_id | 6 字节无符号整型 | 表示该表的 id 值
| flags | 2 字节 | 代码注释中说是预留未来使用，其实现在已经使用了

<h6/>
<h5 align="center">表 3: Post-Header for Rows_event</h5>

| 字段名 | 格式 | 简要描述
|:-:|:-:|:-:
| extra_row_info| Extra_row_info | 本文忽略该字段，有兴趣读者自行查阅
| width | Packed Integer | 代表列的个数
| cols | 字节数组，数组长度 = int((width + 7) / 8) | 每个列是否被使用，每个位代表一个字段
| rows | 0行或多行数据 | 对于一次修改多行的语句，该事件会有多行数据

<h6/>
<h5 align="center">表 4: Body for Rows_event</h5>

代码中的注释和实际实现上有所不同，如上表格以代码实现为准。我们以上面的插入语句为例，分析 binlog 文件中的 Write_rows_log_event 内容:

```
root@ubuntu-bionic:/data/data# mysqlbinlog --start-position=931647020 --hexdump binlog.000001
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 156
#201101  5:58:30 server id 1  end_log_pos 125 CRC32 0x52873723
# Position  Timestamp   Type   Master ID        Size      Master Pos    Flags
# 0000009c 86 4e 9e 5f   0f   01 00 00 00   79 00 00 00   7d 00 00 00   00 00
# 000000af 04 00 38 2e 30 2e 32 32  00 00 00 00 00 00 00 00 |..8.0.22........|
# 000000bf 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
# 000000cf 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
# 000000df 00 00 00 00 86 4e 9e 5f  13 00 0d 00 08 00 00 00 |.....N..........|
# 000000ef 00 04 00 04 00 00 00 61  00 04 1a 08 00 00 00 08 |.......a........|
# 000000ff 08 08 02 00 00 00 0a 0a  0a 2a 2a 00 12 34 00 0a |.............4..|
# 0000010f 28 01 23 37 87 52                                |...7.R|
# 	Start: binlog v 4, server v 8.0.22 created 201101  5:58:30 at startup
ROLLBACK/*!*/;
BINLOG '
hk6eXw8BAAAAeQAAAH0AAAAAAAQAOC4wLjIyAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAACGTp5fEwANAAgAAAAABAAEAAAAYQAEGggAAAAICAgCAAAACgoKKioAEjQA
CigBIzeHUg==
'/*!*/;
# at 931647020
#201107 14:12:16 server id 1  end_log_pos 931647066 CRC32 0xe0258212
# Position  Timestamp   Type   Master ID        Size      Master Pos    Flags
# 3787ce2c 40 ab a6 5f   1e   01 00 00 00   2e 00 00 00   5a ce 87 37   00 00
# 3787ce3f 8c 00 00 00 00 00 01 00  02 00 03 ff 04 01 00 00 |................|
# 3787ce4f 00 05 61 70 70 6c 65 12  82 25 e0                |..apple....|
# 	Write_rows: table id 140 flags: STMT_END_F

BINLOG '
QKumXx4BAAAALgAAAFrOhzcAAIwAAAAAAAEAAgAD/wQBAAAABWFwcGxlEoIl4A==
'/*!*/;
```

- table_id: 8c 00 00 00 00 00 => 140
- flags: 01 00 => STMT_END_F
- extra_row_info: 02 00 => EXTRA_ROW_INFO_HEADER_LENGTH
- width: 03 => 三列
- cols: ff
- rows: 04 01 00 00 00 05 61 70 70 6c 65 => id = 1; name = apple
- crc: 12 82 25 e0

Write_rows_log_event/Delete_rows_log_event 与上面的分析类似，此不赘述。

#### Event Lifetime

一个时间在构造的时候通过传递特定的参数来填充其数据布局中的各字段，然后在 flush event 阶段调用 Log_event::write 将数据写到 binlog 文件中。

使用 gdb 来看 Rows_log_event 构造函数的调用栈:

```
(gdb) bt
#0  Rows_log_event::Rows_log_event (this=0x7f41180f8750,
    __vtt_parm=0x55be8d466dd8 <VTT for Write_rows_log_event+8>, thd_arg=0x7f4118001270,
    tbl_arg=0x7f4118157bf0, tid=..., cols=0x7f4118157d60, using_trans=true,
    event_type=binary_log::WRITE_ROWS_EVENT, extra_row_ndb_info=0x0, __in_chrg=<optimized out>)
    at /opt/vagrant/mysql-server/sql/log_event.cc:7654
#1  0x000055be89dce96a in Write_rows_log_event::Write_rows_log_event (this=0x7f41180f8750,
    thd_arg=0x7f4118001270, tbl_arg=0x7f4118157bf0, tid_arg=..., is_transactional=true,
    extra_row_ndb_info=0x0, __in_chrg=<optimized out>, __vtt_parm=<optimized out>)
    at /opt/vagrant/mysql-server/sql/log_event.cc:11638
#2  0x000055be89d9ef96 in THD::binlog_prepare_pending_rows_event<Write_rows_log_event> (
    this=0x7f4118001270, table=0x7f4118157bf0, serv_id=1, needed=11, is_transactional=true,
    extra_row_info=0x0, source_part_id=2147483647) at /opt/vagrant/mysql-server/sql/binlog.cc:10776
#3  0x000055be89d92783 in THD::binlog_write_row (this=0x7f4118001270, table=0x7f4118157bf0,
    is_trans=true, record=0x7f4118098138 "\377\001", extra_row_info=0x0)
    at /opt/vagrant/mysql-server/sql/binlog.cc:11049
#4  0x000055be89dcea53 in Write_rows_log_event::binlog_row_logging_function (thd_arg=0x7f4118001270,
    table=0x7f4118157bf0, is_transactional=true, before_record=0x0,
    after_record=0x7f4118098138 "\377\001") at /opt/vagrant/mysql-server/sql/log_event.cc:11646
#5  0x000055be88e8c753 in binlog_log_row (table=0x7f4118157bf0, before_record=0x0,
    after_record=0x7f4118098138 "\377\001",
    log_func=0x55be89dcea18 <Write_rows_log_event::binlog_row_logging_function(THD*, TABLE*, bool, unsigned char const*, unsigned char const*)>) at /opt/vagrant/mysql-server/sql/handler.cc:7725
#6  0x000055be88e8d0dc in handler::ha_write_row (this=0x7f4118158fe8, buf=0x7f4118098138 "\377\001")
    at /opt/vagrant/mysql-server/sql/handler.cc:7825
#7  0x000055be891b785c in write_record (thd=0x7f4118001270, table=0x7f4118157bf0, info=0x7f419478a650,
    update=0x7f419478a6d0) at /opt/vagrant/mysql-server/sql/sql_insert.cc:2148
#8  0x000055be891b2ff4 in Sql_cmd_insert_values::execute_inner (this=0x7f41180f0ce0, thd=0x7f4118001270)
    at /opt/vagrant/mysql-server/sql/sql_insert.cc:633
#9  0x000055be88b56296 in Sql_cmd_dml::execute (this=0x7f41180f0ce0, thd=0x7f4118001270)
    at /opt/vagrant/mysql-server/sql/sql_select.cc:725
#10 0x000055be88ad4f7f in mysql_execute_command (thd=0x7f4118001270, first_level=true)
    at /opt/vagrant/mysql-server/sql/sql_parse.cc:3357
```

再看 `Log_event::write` 的调用栈:

```
(gdb) bt
#0  Log_event::write (this=0x7f41180f8750, ostream=0x7f41180f85a8)
    at /opt/vagrant/mysql-server/sql/log_event.h:905
#1  0x000055be89d9c55f in binary_event_serialize<Log_event> (ev=0x7f41180f8750, ostream=0x7f41180f85a8)
    at /opt/vagrant/mysql-server/sql/log_event.h:4375
#2  0x000055be89d742b8 in binlog_cache_data::write_event (this=0x7f41180f8550, ev=0x7f41180f8750)
    at /opt/vagrant/mysql-server/sql/binlog.cc:1389
#3  0x000055be89d862c7 in MYSQL_BIN_LOG::flush_and_set_pending_rows_event (
    this=0x55be8d8c6680 <mysql_bin_log>, thd=0x7f4118001270, event=0x0, is_transactional=true)
    at /opt/vagrant/mysql-server/sql/binlog.cc:6866
#4  0x000055be89d93426 in THD::binlog_flush_pending_rows_event (this=0x7f4118001270, stmt_end=true,
    is_transactional=true) at /opt/vagrant/mysql-server/sql/binlog.cc:11254
#5  0x000055be89d93d8a in THD::binlog_query (this=0x7f4118001270, qtype=THD::ROW_QUERY_TYPE,
    query_arg=0x7f41180eeff8 "INSERT INTO t VALUES(1, 'apple', NULL)", query_len=38, is_trans=true,
    direct=false, suppress_use=false, errcode=0) at /opt/vagrant/mysql-server/sql/binlog.cc:11466
#6  0x000055be891b32e9 in Sql_cmd_insert_values::execute_inner (this=0x7f41180f0ce0, thd=0x7f4118001270)
    at /opt/vagrant/mysql-server/sql/sql_insert.cc:706
#7  0x000055be88b56296 in Sql_cmd_dml::execute (this=0x7f41180f0ce0, thd=0x7f4118001270)
    at /opt/vagrant/mysql-server/sql/sql_select.cc:725
#8  0x000055be88ad4f7f in mysql_execute_command (thd=0x7f4118001270, first_level=true)
    at /opt/vagrant/mysql-server/sql/sql_parse.cc:3357
```

从调用栈可以看出在 handler::ha_write_row 阶段只是完成 Write_rows_log_event 的生成（对于一条语句修改多行的会将事件保存在 binlog_cache_data 的 m_pending 字段以便将后面的数据添加到这个事件中），具体的刷盘操作由 `MYSQL_BIN_LOG::flush_and_set_pending_rows_event` 完成。这里的逻辑有些复杂，留给后面的文章来具体分析这块的逻辑。

**题外话:**

mysqlbinlog 在使用 --start-position 参数时 FDE 的显示地址有误，笔者已提 [bug #101450][bug101450] 及相关修改给 MySQL 社区。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [mysqlbinlog Row Event Display][mysqlbinlog-row-events]<br>
</span>

[binlog-event-format]: /2020/11/01/binlog-event-format.html
[write-only-fg]: /assets/2020/sysbench-write-only-with-binlog-on.svg
[row]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/rows_event.h#L951
[Rows_log_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/sql/log_event.h#L2697
[Table_map_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/rows_event.h#L521
[Table_map_log_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/sql/log_event.h#L2388
[net_store_length]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/mysys/pack.cc#L128
[enum_field_types]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/include/field_types.h#L52
[Table_table_map_event_column_types]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/rows_event.h#L192
[Table_table_map_event_optional_metadata]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/rows_event.h#L407
[Rows_log_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/sql/log_event.h#L2697
[Rows_event]: https://github.com/mysql/mysql-server/blob/mysql-cluster-8.0.22/libbinlogevents/include/rows_event.h#L873
[mysqlbinlog-row-events]: https://dev.mysql.com/doc/refman/8.0/en/mysqlbinlog-row-events.html
[bug101450]: https://bugs.mysql.com/bug.php?id=101450
