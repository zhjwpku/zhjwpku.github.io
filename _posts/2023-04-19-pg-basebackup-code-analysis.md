---
layout: post
title: pg_basebackup 代码浅析
date: 2023-04-19 00:00:00 +0800
tags:
- postgres
- pg_basebackup
- wal
---

[pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) 是 PostgreSQL 中基于流复制的在线备份工具，它通过使用与流复制相同的协议，从一个正在运行的 PostgreSQL 服务器复制数据目录并创建一个新的基线备份，并将其传输到指定的目录或远程服务器。

*注: 本文分析基于源码 Postgres 16devel (commit 6ff2e8cdd410f70057cfa6259ad395c1119aeb32)*

pg_basebackup 是一个客户端进程，它使用 replication mode 连接到 PostgreSQL 服务端，服务端 fork 一个 walsender 进程与客户端进行交互。因此涉及 pg_basebackup 的代码有两部分:

```text
## 客户端代码
src/bin/pg_basebackup/receivelog.c
src/bin/pg_basebackup/streamutil.c
src/bin/pg_basebackup/walmethods.c
src/bin/pg_basebackup/pg_basebackup.c         <- main 函数入口
src/bin/pg_basebackup/bbstreamer_file.c
src/bin/pg_basebackup/bbstreamer_gzip.c
src/bin/pg_basebackup/bbstreamer_inject.c
src/bin/pg_basebackup/bbstreamer_lz4.c
src/bin/pg_basebackup/bbstreamer_tar.c
src/bin/pg_basebackup/bbstreamer_zstc

## 服务端代码
src/backend/backup/*
```

我们通过一个最简单的备份操作来分析一下客户端和服务端的交互过程:

```shell
pg_basebackup --pgdata=/tmp/pg_backup --format=t --compress=9 --verbose
```

忽略 C/S 之间建链的过程，上述命令 pg_basebackup 和 walsender 的交互如下图展示:

![pg_basebackup](/assets/2023/pg_basebackup.jpg)

上述 pg_basebackup 操作发给 walsender 的命令格式如下:

```text
BASE_BACKUP ( LABEL 'pg_basebackup base backup', PROGRESS, WAIT 0, TABLESPACE_MAP, MANIFEST 'yes', TARGET 'client')
```

walsender 在 `exec_replication_command` 中使用语法解析器把字符串转化为 `BaseBackupCmd`，然后在 `SendBaseBackup` 中调用 `parse_basebackup_options` 将 `BaseBackupCmd` 结构中的 options 列表解析到 `basebackup_options` 结构中。

```gdb
* thread #1, queue = 'com.apple.main-thread', stop reason = step in
    frame #0: 0x0000000101446e25 postgres`SendBaseBackup(cmd=0x00007fbd99013af0) at basebackup.c:968:30 [opt]
   965  {
   966          basebackup_options opt;
   967          bbsink     *sink;
-> 968          SessionBackupState status = get_backup_status();
   969 
   970          if (status == SESSION_BACKUP_RUNNING)
   971                  ereport(ERROR,
(lldb) p *cmd
(BaseBackupCmd) $5 = {
  type = T_BaseBackupCmd
  options = 0x00007fbd99013010
}

(lldb) p opt
(basebackup_options) $8 = {
  label = 0x00007fbd99012ba8 "pg_basebackup base backup"
  progress = true
  fastcheckpoint = false
  nowait = true
  includewal = false
  maxrate = 0
  sendtblspcmapfile = true
  send_to_client = true
  use_copytblspc = false
  target_handle = NULL
  manifest = MANIFEST_OPTION_YES
  compression = PG_COMPRESSION_NONE
  compression_specification = {
    algorithm = PG_COMPRESSION_NONE
    options = 0
    level = 0
    workers = 0
    long_distance = false
    parse_error = 0x0000000000000000
  }
  manifest_checksum_type = CHECKSUM_TYPE_CRC32C
}
```

根据 basebackup_options 创建了一个 'copystream' basebackup sink，然后调用 `perform_base_backup` 去执行 base backup 的操作。`perform_base_backup` 的关键调用栈如下：

```C
perform_base_backup
  do_pg_backup_start                // 获取 backup 起始状态，包括 start lsn、start tli 等信息
  bbsink_begin_backup               // 通知客户端
    bbsink_copystream_begin_backup
      SendXlogRecPtrResult          // Tell client the backup start location.
      SendTablespaceList            // Send client a list of tablespaces.
      pq_puttextmessage('C', "SELECT"); // Send a CommandComplete message
      SendCopyOutResponse           // Begin COPY stream.
  foreach(lc, state.tablespaces)    // Send off our tablespaces one by one
    bbsink_begin_archive(sink, "base.tar");
    bbsink_copystream_begin_archive // Send a CopyData message announcing the beginning of a new archive.
      pq_beginmessage(&buf, 'd');   // CopyData
      pq_sendbyte(&buf, 'n');       // New archive
      pq_sendstring(&buf, archive_name);
    sendDir(sink, ".", 1, false, state.tablespaces,
            sendtblspclinks, &manifest, NULL);  // send the bulk of the files...
      sendFile
        basebackup_read_file
        bbsink_archive_contents     // Archive the data we just read
          bbsink_copystream_archive_contents    // Send a CopyData message containing a chunk of archive content.
    bbsink_end_archive              // OK, that's the end of the archive.
      bbsink_copystream_end_archive
  do_pg_backup_stop                 // 结束 base backup
  SendBackupManifest                // 发送 manifest
    bbsink_begin_manifest           // Send the backup manifest.
      bbsink_copystream_begin_manifest
    bbsink_manifest_contents        // Process the manifest contents.
      bbsink_copystream_manifest_contents
    bbsink_end_manifest             // Finish the backup manifest.
      bbsink_copystream_end_manifest
  bbsink_end_backup                 // Finish a backup.
    bbsink_copystream_end_backup
bbsink_cleanup                      // Release resources before destruction.
  bbsink_copystream_cleanup
```

由于我们没指定 --wal-method，默认为 stream 模式，即在执行 base backup 的同时（base backup 忽略 pg_wal 目录中的文件），根据返回的 `xlogstart` 和 `starttli` fork 一个子进程去同步 wal 日志（逻辑见函数 StartLogStreamer），这样能够保证 wal 日志的完整性。如果指定了 --wal-method=fetch，则是在 base backup 的过程中备份 pg_wal 目录中的文件，但这可能会造成备份过程中 wal 日志被删除。

pg_baseback 主进程在接收完 base 备份之后，会收到一个 xlogend 的 LSN，主进程通过管道将 xlogend 发给 logstreamer 进程，xlogstream 在接收到 xlogend 位点的 wal 后退出，父进程 waitpid 等待其退出，做一些收尾工作，整个流程退出。

备份完成后目标目录下的内容如下:

```shell
➜  pg_backup ls
backup_manifest base.tar.gz     pg_wal.tar.gz
```

如果指定备份 format=p，则备份后的目录内容为:

```shell
➜  pg_backup ls       
PG_VERSION           global               pg_hba.conf          pg_notify            pg_stat              pg_twophase          postgresql.conf
backup_label         logfile              pg_ident.conf        pg_replslot          pg_stat_tmp          pg_wal
backup_manifest      pg_commit_ts         pg_logical           pg_serial            pg_subtrans          pg_xact
base                 pg_dynshmem          pg_multixact         pg_snapshots         pg_tblspc            postgresql.auto.conf
➜  pg_backup ls pg_wal
000000010000000000000013 archive_status
```

**pg_basebackup 一个使用的前提是需要在 primary 节点打开 `full_page_writes` 选项**，因为备份数据时可能会 dump 到一个不完整的数据页，需要 WAL 记录的 full page 去修复数据页。

> We must do full-page WAL writes during an on-line backup even if not doing so at other times, because it's quite possible for the backup dump to obtain a "torn" (partially written) copy of a database page if it reads the page concurrently with our write to the same page.
>
> This can be fixed as long as the first write to the page in the WAL sequence is a full-page write.

#### 小结

本文对 pg_basebackup 的代码逻辑进行了粗略的解析，读者可以通过本文的分析了解 pg_basebackup 基本的工作原理。其它参数的含义需要读者自行阅读源码去了解。另外 C/S 两端 copy stream 数据传输的过程笔者也并未深入，感兴趣的读者请自行阅读代码。🧐

