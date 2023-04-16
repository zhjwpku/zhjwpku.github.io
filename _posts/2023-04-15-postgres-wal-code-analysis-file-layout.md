---
layout: post
title: PostgreSQL WAL 代码解析 —— File Layout
date: 2023-04-15 00:00:00 +0800
tags:
- postgres
- wal
- xlog
---

WAL (Write Ahead Log) 是 PG 保证 ACID 中 Atomicity 和 Durability 特性的子模块，任何修改必须先写入 stable storage 的日志后才能将结果返回给客户端，在系统失效重启后，通过回放日志以保证不丢失数据（数据库常用的 STEAL + NO-FORCE 模式）。

*注: 本文介绍 WAL 文件格式相关的内容，源码 Postgres 16devel (commit 6ff2e8cdd410f70057cfa6259ad395c1119aeb32)*

首先了解一些 WAL 的基本概念 💡

#### LSN 与 WAL 文件映射关系

Postgres 10 将保存 WAL 日志的目录从 `pg_xlog` 改为 `pg_wal`，并将用户可设置的 GUC 进行了类似的重命名，不过源码保留 xlog 术语，因此看代码的时候可以认为 **xlog == wal**。

**src/include/access/xlogdefs.h** 中定义了 WAL 的三个相关类型:

```C++
typedef unit64 XLogRecPtr;      // 指向 WAL 中的位置，即 LSN，64 位避免溢出

typedef uint64 XLogSegNo;       // 日志文件的 sequence number

typedef uint32 TimeLineID;      // 用于识别不同数据库历史，暂停/重启(包括 crash-and-recovery)不改变该值，但是在做 point-in-time recovery 后需要赋予一个新的 TLI
```

XLogRecPtr 定义为 64 位无符号数意味着逻辑上 WAL 是一个 16 ExaByte 的文件，为方便管理，Postgres 将 LSN 的高 32 位称为 `xlogid`，低 32 位称为 `xrecoff`，即逻辑上将 WAL 拆分成了 4G 个 4 GigaByte 的逻辑文件。但 4 GigaByte 依然是一个比较大的文件对象，Postgres 进一步将其切分成 16MB（initdb 时可用 `--wal-segsize` 修改 Segment 大小）一个的 Segment。

**src/include/pg_config_manual.h**
```C
/*
 * This is the default value for wal_segment_size to be used when initdb is run
 * without the --wal-segsize option.  It must be a valid segment size.
 */
#define DEFAULT_XLOG_SEG_SIZE	(16*1024*1024)
```

**src/backend/access/transam/xlog.c**
```C
int	wal_segment_size = DEFAULT_XLOG_SEG_SIZE;
```

Segment 文件的名字由三部分组成，如下图所示:

![XLog File Name](/assets/2023/wal_filename.jpg)

可以看出，三部分各自都能够表达 32 位的范围，但 Segment 文件本身具有一定的大小，segmentId 通常不会很大，比如对于默认 16MB 的 Segment 大小，上面的 segmentId 的取值范围为 [0, 255]，即每个 XLogId 对应 256 个 Segment。**(segmentId * wal_segment_size) + 文件内的偏移 = xrecoff**。

**src/include/access/xlog_internal.h**
```C
#define XLogSegmentsPerXLogId(wal_segsz_bytes)	\
	(UINT64CONST(0x100000000) / (wal_segsz_bytes))

static inline void
XLogFileNameById(char *fname, TimeLineID tli, uint32 log, uint32 seg)
{
	snprintf(fname, MAXFNAMELEN, "%08X%08X%08X", tli, log, seg);
}
```

根据 Segment 的序号及 Segment 大小，很容易得出该 Segment 的文件名:

```C
/*
 * Generate a WAL segment file name.  Do not use this function in a helper
 * function allocating the result generated.
 */
static inline void
XLogFileName(char *fname, TimeLineID tli, XLogSegNo logSegNo, int wal_segsz_bytes)
{
	snprintf(fname, MAXFNAMELEN, "%08X%08X%08X", tli,
			 (uint32) (logSegNo / XLogSegmentsPerXLogId(wal_segsz_bytes)),
			 (uint32) (logSegNo % XLogSegmentsPerXLogId(wal_segsz_bytes)));
}
```

给出一个 LSN 可以将其转为 **xlogid/xrecoff** 字符串格式，然后在 psql 调用 `pg_walfile_name` 既可得到它对应的 Segment 文件名。

```C
/*
 * Handy macro for printing XLogRecPtr in conventional format, e.g.,
 *
 * printf("%X/%X", LSN_FORMAT_ARGS(lsn));
 */
#define LSN_FORMAT_ARGS(lsn) (AssertVariableIsOfTypeMacro((lsn), XLogRecPtr), (uint32) ((lsn) >> 32)), ((uint32) (lsn))
```

如 *LSN1 = 4311744512* 及 *LSN2 = 4311744513* 可通过如下操作获取到其所在的 Segment 文件名。

```SQL
postgres=# \set x 4311744512
postgres=# SELECT concat(to_hex(:x >> 32), '/', to_hex(:x & 0xFFFFFFFF)) as locationpoint;
 locationpoint 
---------------
 1/1000000
(1 row)

postgres=# SELECT pg_walfile_name('1/1000000');
     pg_walfile_name      
--------------------------
 000000010000000100000000
(1 row)

postgres=# SELECT pg_walfile_name('1/1000001');
     pg_walfile_name      
--------------------------
 000000010000000100000001
(1 row)
```

可以看出，如上的 LSN1 和 LSN2 处于不同的文件。读者可以用如上方法测试下 *LSN3 = 8589934591* 和 *LSN4 = 8589934592* 所在的文件，看看有什么发现 🧐。Postgres 还提供一个函数 `pg_walfile_name_offset` 用于查询 LSN 所在的文件及其在文件中的 offset。

initdb 创建数据集簇的时候创建第一个 Segment 文件 **000000010000000000000001**，即 (timelineId = 1, xlogId = 0, segmentId = 1)，代码逻辑见 `BootStrapXLOG`。这样定义的好处是可以使用 0 来代表非法的 LSN。

```C
/*
 * Zero is used indicate an invalid pointer. Bootstrap skips the first possible
 * WAL segment, initializing the first WAL page at WAL segment size, so no XLOG
 * record can begin at zero.
 */
#define InvalidXLogRecPtr	0
#define XLogRecPtrIsInvalid(r)	((r) == InvalidXLogRecPtr)
```

#### WAL 文件布局

通过上节的介绍，我们知道了 WAL 是一个一个固定大小的 Segment 文件，本节我们来看下在一个 Segment 中，WAL 记录是如何布局的。

WAL Segment 将空间划分为 8KB（可通过 configure --with-wal-blocksize 修改，合法值为 1,2,4,8,16,32,64，单位 KB）的 XLogPage，一个默认的 16 MB 的 Segment 包含 2048 个 XLogPage，如下图:

![XLogPage](/assets/2023/xlog_pages.jpg)

其中每个 Segment 第一个页比较特殊，其页头相对于其它页多出了 16 个字节，长页头 `XLogLongPageHeaderData` 包含短页头 `XLogPageHeaderData`:

**src/include/access/xlog_internal.h**
```C
typedef struct XLogPageHeaderData
{
	uint16		xlp_magic;	/* magic value for correctness checks */
	uint16		xlp_info;	/* flag bits, see below */
	TimeLineID	xlp_tli;	/* TimeLineID of first record on page */
	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */

	/*
	 * When there is not enough space on current page for whole record, we
	 * continue on the next page.  xlp_rem_len is the number of bytes
	 * remaining from a previous page; it tracks xl_tot_len in the initial
	 * header.  Note that the continuation data isn't necessarily aligned.
	 */
	uint32		xlp_rem_len;	/* total len of remaining data for record */
} XLogPageHeaderData;

typedef XLogPageHeaderData *XLogPageHeader;

/*
 * When the XLP_LONG_HEADER flag is set, we store additional fields in the
 * page header.  (This is ordinarily done just in the first page of an
 * XLOG file.)	The additional fields serve to identify the file accurately.
 */
typedef struct XLogLongPageHeaderData
{
	XLogPageHeaderData std;			/* standard header fields */
	uint64		xlp_sysid;		/* system identifier from pg_control */
	uint32		xlp_seg_size;		/* just as a cross-check */
	uint32		xlp_xlog_blcksz;	/* just as a cross-check */
} XLogLongPageHeaderData;

typedef XLogLongPageHeaderData *XLogLongPageHeader;
```

由于存在 Alignment，短页头结构体大小为 24 字节，长页头结构体大小为 40 字节，我们用 hexdump 看下 initdb 生成的第一个 Segment 文件第一个页头的内容（hexdump 的显示为低地址在前，高地址在后）:

```shell
➜  pg_wal hexdump -C -n 40 000000010000000000000001
00000000  13 d1 02 00 01 00 00 00  00 00 00 01 00 00 00 00  |.�..............|
00000010  00 00 00 00 00 00 00 00  46 e0 d3 df cd 55 36 64  |........F����U6d|
00000020  00 00 00 01 00 20 00 00                           |..... ..|
00000028
```

- [ 0, 1] magic，0xd113，对应 XLOG_PAGE_MAGIC
- [ 2, 3] flag bits, 0x0002，对应 XLP_LONG_HEADER，说明这是一个长页头，即 Segment 中的第一个页
- [ 4, 7] timelineId, 0x00000001，即 timelineId = 1，对应 BootstrapTimeLineID
- [ 8,15] 该页的 LSN, 0x0000000001000000，结合之前的分析，可以得出该值对应 segmentId = 1，匹配文件名
- [16,19] 0x00000000, remaining data for record
- [20,23] 0x0 alignment padding
- [24,31] 0x643655cddfd3e046 通过时间和 pid 生成的一个系统标识符
- [32,35] 0x01000000, Segment 大小，16M，对应 wal_segment_size
- [36,39] 0x00002000, XLog Page 大小，8K，对应 XLOG_BLCKSZ

再看第二个页头的内容:

```shell
➜  pg_wal hexdump -s 8192 -C -n 24 000000010000000000000001
00002000  13 d1 05 00 01 00 00 00  00 20 00 01 00 00 00 00  |.�....... ......|
00002010  29 00 00 00 00 00 00 00                           |).......|
00002018
```

- [ 0, 1] magic, 0xd113 
- [ 2, 3] flag bits, 0x0005，对应 XLP_FIRST_IS_CONTRECORD \| XLP_BKP_REMOVABLE，说明上一个页最后一个 record 跨页了
- [ 4, 7] timelineId，同第一个页相同
- [ 8,15] 0x0000000001002000，该页所在的 segmentId = 1，偏移为 8K，即第二个页
- [16,19] 0x00000029, 上个页最后一个 record 遗留的数据长度

可以看出，之所以第一个页使用长页头，可用于读取 Segment 文件时校验 wal_segment_size 和 XLOG_BLCKSZ 是否兼容。

下面再来看 WAL record 的格式。一个 XLOG record 包含通用的 Header 部分和与其相关的数据部分。

**src/include/access/xlogrecord.h**
```C
/*
 * The overall layout of an XLOG record is:
 *		Fixed-size header (XLogRecord struct)
 *		XLogRecordBlockHeader struct
 *		XLogRecordBlockHeader struct
 *		...
 *		XLogRecordDataHeader[Short|Long] struct
 *		block data
 *		block data
 *		...
 *		main data
 */
typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */
	TransactionId xl_xid;			/* xact id */
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
	uint8		xl_info;		/* flag bits, see below */
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */

} XLogRecord;
```

XLogRecord 总是以 `MAXALIGN` (64位机器上为 8 字节) 边界对齐，但其内部字段没有这样的限制。`xl_info` 的低四位被用于标记特殊修改方式（XLR_SPECIAL_REL_UPDATE）或一致性检测（XLR_CHECK_CONSISTENCY），高四位被 resource manager 用于标记不同的记录类型。`xl_rmid` 则记录了生成该记录的 resource manager，该字段 8 字节，因此最多支持 128 个 resource manager，**src/include/access/rmgrlist.h** 中罗列了所有的 RM。在故障恢复的时候，根据 xl_rmid 和 xl_info 中的信息找到对应的操作来回放操作。

> Postgres 最初只支持内置的 Resource Manager，Citus 在开发基于 [Table Access Method API](https://www.postgresql.org/docs/current/tableam.html) 的 [Citus Columnar](https://github.com/citusdata/citus/tree/main/src/backend/columnar) 时，为了支持逻辑 insert/update/delete WAL 记录，向社区贡献了 [Custom WAL Resource Manager](https://wiki.postgresql.org/wiki/CustomWALResourceManagers) 特性，使得 table AM 支持自定义的 redo、decode、WAL format、crash recovery 等能力。详细讨论见 [Extensible Rmgr for Table AMs](https://www.postgresql.org/message-id/ed1fb2e22d15d3563ae0eb610f7b61bb15999c0a.camel%40j-davis.com)。

XLogRecord 是一条日志的通用头部分，之后的 `xl_tot_len - SizeOfXLogRecord` 保存了 0 或多个 **XLogRecordBlockHeader** 及 1 个**XLogRecordDataHeader[Short\|Long]**，每个 Header 都以 uint8 大小 id 字段开始，并且都有其一一对应的数据部分。

![The Internals of PostgreSQL Fig. 9.9. Common XLOG record format.](/assets/2023/fig-9-09.png)

*上图引用自: The Internals of PostgreSQL Fig. 9.9. Common XLOG record format.*

值得注意的是，**XLogRecordBlockHeader** 结构并非定长，当 fork_flags 设置了 `BKPBLOCK_HAS_IMAGE`，data_length 后会附加 XLogRecordBlockImageHeader 结构（如果 backup block 内有空洞或启用了压缩算法，则在该结构后还会跟一个 XLogRecordBlockCompressHeader 结构）；如果未设置 `BKPBLOCK_SAME_REL`，后面会再追加一个 RelFileLocator 结构。

**src/include/access/xlogrecord.h**
```C
/*
 * Header info for block data appended to an XLOG record.
 *
 * 'data_length' is the length of the rmgr-specific payload data associated
 * with this block. It does not include the possible full page image, nor
 * XLogRecordBlockHeader struct itself.
 *
 * Note that we don't attempt to align the XLogRecordBlockHeader struct!
 * So, the struct must be copied to aligned local storage before use.
 */
typedef struct XLogRecordBlockHeader
{
	uint8		id;		/* block reference ID */
	uint8		fork_flags;	/* fork within the relation, and flags */
	uint16		data_length;	/* number of payload bytes (not including page image) */

	/* If BKPBLOCK_HAS_IMAGE, an XLogRecordBlockImageHeader struct follows */
	/* If BKPBLOCK_SAME_REL is not set, a RelFileLocator follows */
	/* BlockNumber follows */
} XLogRecordBlockHeader;

/*
 * The fork number fits in the lower 4 bits in the fork_flags field. The upper
 * bits are used for flags.
 */
#define BKPBLOCK_FORK_MASK	0x0F
#define BKPBLOCK_FLAG_MASK	0xF0
#define BKPBLOCK_HAS_IMAGE	0x10	/* block data is an XLogRecordBlockImage */
#define BKPBLOCK_HAS_DATA	0x20
#define BKPBLOCK_WILL_INIT	0x40	/* redo will re-init the page */
#define BKPBLOCK_SAME_REL	0x80	/* RelFileLocator omitted, same as previous */
```

由于 XLogRecordDataHeaderShort 和 XLogRecordDataHeaderLong 在一条 WAL 记录中只会有一个，其 id 字段被设置为固定值。

**src/include/access/xlogrecord.h**
```C
/*
 * XLogRecordDataHeaderShort/Long are used for the "main data" portion of
 * the record. If the length of the data is less than 256 bytes, the short
 * form is used, with a single byte to hold the length. Otherwise the long
 * form is used.
 */
typedef struct XLogRecordDataHeaderShort
{
	uint8		id;		/* XLR_BLOCK_ID_DATA_SHORT */
	uint8		data_length;	/* number of payload bytes */
} XLogRecordDataHeaderShort;

typedef struct XLogRecordDataHeaderLong
{
	uint8		id;		/* XLR_BLOCK_ID_DATA_LONG */
	/* followed by uint32 data_length, unaligned */
} XLogRecordDataHeaderLong;

#define XLR_MAX_BLOCK_ID		32

#define XLR_BLOCK_ID_DATA_SHORT		255
#define XLR_BLOCK_ID_DATA_LONG		254
#define XLR_BLOCK_ID_ORIGIN		253
#define XLR_BLOCK_ID_TOPLEVEL_XID	252
```

注意到通用头 `XLogRecord` 结构中并没有指定 XLogRecordBlockHeader 的个数，读取 xlog record 时可根据头结构中 id 字段的取值，识别 XLogRecordDataHeader[Short\|Long]，用于确定是否已经解析到最后一个头。下图展示了 XLog 记录的三个例子:

![The Internals of PostgreSQL Fig. 9.10. Examples of XLOG records.](/assets/2023/fig-9-10.png)

*上图引用自: The Internals of PostgreSQL Fig. 9.10. Examples of XLOG records.*

- a) Backup block 的 main data 部分依生成该记录的命令而异，如上图 Insert 操作生成的 Backup block，其 main data 部分是 xl_head_insert，如果是 Update 操作，则 main data 部分是 xl_heap_updated。
- b) Non-Backup block 插入操作在 `XLogRecordBlockHeader` 中记录了用于定位修改的 block 的 RelFileLocator、BlockNumber 和 ForkNumber（fork_flags 字段） 等信息，其对应的数据部分 block data 保存了所插入的数据。这使得 main data 部分记录的 xl_heap_insert 非常简单，包含插入数据在页内的 offset 以及可见性信息。
- c) CheckPoint 记录不包含 `XLogRecordBlockHeader`，因为没有相关的 block。

下面我们使用 hexdump 来看一个 WAL record 的内容，先打印 XLogRecord 部分，长度 24 字节（跳过第一个页面 XLogLongPageHeaderData 的 40 字节）:

```shell
➜  pg_wal hexdump -s 40 -C -n 24 000000010000000000000001
00000028  72 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |r...............|
00000038  00 00 00 00 20 48 9c 0d                           |.... H..|
00000040
```

- [40-43] 0x00000072，xl_tot_len = 114
- [44-47] xid = 0
- [48-55] prev lsn = 0/00000000
- [56] xl_info, 高四位 0x00，对应 XLOG_CHECKPOINT_SHUTDOWN
- [57] xl_rmid: 0, RM_XLOG_ID 即 XLOG 资源管理器
- [58-59] padding
- [60-63] crc

读取 XLogRecord 的第一次字节，查看是哪个 header:

```shell
➜  pg_wal hexdump -s 64 -C -n 1 000000010000000000000001 
00000040  ff                                                |�|
00000041
```

ff 值为 255，对应 XLR_BLOCK_ID_DATA_SHORT，因此对应 XLogRecordDataHeaderShort，之后的一个字节:

```shell
➜  pg_wal hexdump -s 65 -C -n 1 000000010000000000000001
00000041  58                                                |X|
00000042
```

即 main data 部分的数据长度为 88 字节，具体内容本文不展开介绍了，验证一下该记录的长度:

```text
   XLogRecord + XLogRecordDataHeaderShort + main data
       24     +           2               +     88    = 114
```

然后我们用 pg_waldump 工具查看一下第一条 xlog 的内容，根据页头和第一条记录的长度，我们可以算出第一条记录结束的 LSN，使用如下命令打印第一条记录，发现和上面用 hexdump 分析的结果一致。

```shell
➜  pg_wal pg_waldump -e '0/100009a' 000000010000000000000001
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/01000028, prev 0/00000000, desc: CHECKPOINT_SHUTDOWN redo 0/1000028; tli 1; prev tli 1; fpw true; xid 0:3; oid 10000; multi 1; offset 0; oldest xid 3 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 0; shutdown
```

hexdump 可以在字节级别查看 WAL 页面内容，但是 pg_waldump 使用起来更加方便，如下面两个例子，一个通过起始 lsn 和 limit 查看下个记录的内容，另一个通过 -x 打印同一个事务产生的 wal 日志:

```shell
➜  pg_wal pg_waldump -s '0/10000a0' -n 1 000000010000000000000001
rmgr: XLOG        len (rec/tot):     30/    30, tx:          1, lsn: 0/010000A0, prev 0/01000028, desc: NEXTOID 18192

➜  pg_wal pg_waldump -x 1 -n 5 000000010000000000000001
rmgr: XLOG        len (rec/tot):     30/    30, tx:          1, lsn: 0/010000A0, prev 0/01000028, desc: NEXTOID 18192
rmgr: XLOG        len (rec/tot):     49/   137, tx:          1, lsn: 0/010000C0, prev 0/010000A0, desc: FPI , blkref #0: rel 1663/1/6117 blk 0 FPW
rmgr: XLOG        len (rec/tot):     49/   137, tx:          1, lsn: 0/01000150, prev 0/010000C0, desc: FPI , blkref #0: rel 1664/0/6115 blk 0 FPW
rmgr: XLOG        len (rec/tot):     49/   137, tx:          1, lsn: 0/010001E0, prev 0/01000150, desc: FPI , blkref #0: rel 1664/0/6114 blk 0 FPW
rmgr: XLOG        len (rec/tot):     49/   137, tx:          1, lsn: 0/01000270, prev 0/010001E0, desc: FPI , blkref #0: rel 1663/1/6116 blk 0 FPW
```

上面的 FPI 是 Full Page Image 的简称，即对应一个 Backup block 记录。除了 pg_waldump，还可以使用 `pg_walinspect` 插件来查看在 lsn 后的第一个 wal record 或两个 lsn 之间所有的 wal record:

```SQL
postgres=# select * from pg_get_wal_record_info('0/01000028');
-[ RECORD 1 ]----+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
start_lsn        | 0/1000028
end_lsn          | 0/10000A0
prev_lsn         | 0/0
xid              | 0
resource_manager | XLOG
record_type      | CHECKPOINT_SHUTDOWN
record_length    | 114
main_data_length | 88
fpi_length       | 0
description      | redo 0/1000028; tli 1; prev tli 1; fpw true; xid 0:3; oid 10000; multi 1; offset 0; oldest xid 3 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 0; shutdown
block_ref        |

postgres=# select * from pg_get_wal_records_info('0/01000029', '0/01000300');
-[ RECORD 1 ]----+---------------------------------------------------------------------------------
start_lsn        | 0/10000A0
end_lsn          | 0/10000C0
prev_lsn         | 0/1000028
xid              | 1
resource_manager | XLOG
record_type      | NEXTOID
record_length    | 30
main_data_length | 4
fpi_length       | 0
description      | 18192
block_ref        | 
-[ RECORD 2 ]----+---------------------------------------------------------------------------------
start_lsn        | 0/10000C0
end_lsn          | 0/1000150
prev_lsn         | 0/10000A0
xid              | 1
resource_manager | XLOG
record_type      | FPI
record_length    | 137
main_data_length | 0
fpi_length       | 88
description      | 
block_ref        | blkref #0: rel 1663/1/6117 fork main blk 0 (FPW); hole: offset: 72, length: 8104
-[ RECORD 3 ]----+---------------------------------------------------------------------------------
start_lsn        | 0/1000150
end_lsn          | 0/10001E0
prev_lsn         | 0/10000C0
xid              | 1
resource_manager | XLOG
record_type      | FPI
record_length    | 137
main_data_length | 0
fpi_length       | 88
description      | 
block_ref        | blkref #0: rel 1664/0/6115 fork main blk 0 (FPW); hole: offset: 72, length: 8104
-[ RECORD 4 ]----+---------------------------------------------------------------------------------
start_lsn        | 0/10001E0
end_lsn          | 0/1000270
prev_lsn         | 0/1000150
xid              | 1
resource_manager | XLOG
record_type      | FPI
record_length    | 137
main_data_length | 0
fpi_length       | 88
description      | 
block_ref        | blkref #0: rel 1664/0/6114 fork main blk 0 (FPW); hole: offset: 72, length: 8104
-[ RECORD 5 ]----+---------------------------------------------------------------------------------
start_lsn        | 0/1000270
end_lsn          | 0/1000300
prev_lsn         | 0/10001E0
xid              | 1
resource_manager | XLOG
record_type      | FPI
record_length    | 137
main_data_length | 0
fpi_length       | 88
description      | 
block_ref        | blkref #0: rel 1663/1/6116 fork main blk 0 (FPW); hole: offset: 72, length: 8104

```

#### 小结

本文仅介绍了 postgres 中 wal 文件的命名规范及 wal 文件的布局，并未深入 wal 与其它子模块的交互流程，包括不同 resource manager 是如何写入 wal 文件及 crash recovery 如何将 wal 恢复到 buffer manager 等内容。在今后的文章里，我会对这部分代码进行解析。

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [The Internals of PostgreSQL, Chapter 9 Write Ahead Logging](http://www.interdb.jp/pg/pgsql09.html).<br>
2 [pg_walinspect](https://www.postgresql.org/docs/current/pgwalinspect.html)<br>
</span>

[url_name]: url
