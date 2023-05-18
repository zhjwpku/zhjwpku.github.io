---
layout: post
title: GPDB 差异化修复原理简介
date: 2023-05-16 08:00:00 +0800
tags:
- gpdb
- recovery
---

Greenplum [PR15146](https://github.com/greenplum-db/gpdb/pull/15146) 增加了一种新的 mirror 节点修复方式 —— **差异化修复**（Differential recovery），介于增量修复（Incremental recovery）和全量修复（Full recovery）之间，当数据量庞大且由于 WAL 缺失不能进行增量修复时，差异化修复提供了一种快速修复的能力。

#### ☆ GPDB 节点修复原理简介

GPDB 的 segment 节点修复以 python 脚本 `gprecoverseg` 为入口，通过读取 Coordinator 节点的元数据，来获取需要修复（seg.isSegmentDown()）节点的三元组（表示为 RecoveryTriplet）：

- failed: 当前 down 的 segment，即要修复的节点
- live:   当前 up 的节点，作为修复的 source
- failover: 节点修复的 dest，如果该值为 None，则在 failed 节点原地修复

通过将如上三元组数组构造为一个 `GpMirrorListToBuild`，之后调用该对象的 `recover_mirrors` 函数进行修复。

该函数首先通过 `build_recover_info` 为每个需要修复的 segment 生成一个 `RecoverInfo` 对象，保存在一个 { hostname, RecoverInfo list } 的字典结构中。

```python
def build_recovery_info(mirrors_to_build):
    """
    This function is used to format recovery information to send to each segment host

    @param mirrors_to_build:  list of mirrors that need recovery

    @return A dictionary with the following format:

            Key   =   <host name>
            Value =   list of RecoveryInfos - one RecoveryInfo per segment on that host
    """
    timestamp = datetime.datetime.today().strftime('%Y%m%d_%H%M%S')

    recovery_info_by_host = defaultdict(list)
    for to_recover in mirrors_to_build:

        source_segment = to_recover.getLiveSegment()
        target_segment = to_recover.getFailoverSegment() or to_recover.getFailedSegment()

        process_name = 'pg_basebackup' if to_recover.isFullSynchronization() else 'pg_rewind'
        progress_file = '{}/{}.{}.dbid{}.out'.format(gplog.get_logger_dir(), process_name, timestamp,
                                                     target_segment.getSegmentDbId())

        hostname = target_segment.getSegmentHostName()

        recovery_info_by_host[hostname].append(RecoveryInfo(
            target_segment.getSegmentDataDirectory(), target_segment.getSegmentPort(),
            target_segment.getSegmentDbId(), source_segment.getSegmentHostName(),
            source_segment.getSegmentPort(), to_recover.isFullSynchronization(),
            progress_file))
    return recovery_info_by_host
```

然后调用 `_run_setup_recovery` 和 `_run_recovery` 分别去要修复的节点执行 `gpsegsetuprecovery.py` 和 `gpsegrecovery.py`。

`gpsegsetuprecovery.py` 在目标节点上根据宕机节点的修复类型（full/incremental）做不同的前置工作:

- 如果是全量修复，执行 `ValidationForFullRecovery` 检测目标端的目录是否满足需求（如果非 overwrite 需要保证目标目录为空）
- 如果是增量修复，执行 `SetupForIncrementalRecovery` 去源端执行 `CHECKPOINT` 命令（执行 CHECKPOINT 是为了保证如果 source 端刚被 promote，其 control file 能反应最新的 timeline information，详见 [pg_rewind](https://www.postgresql.org/docs/15/app-pgrewind.html)）并将目的端的 *postmaster.pid* 删除。

`gpsegrecovery.py` 在目标节点上根据修复类型执行 `pg_basebackup` 或 `pg_rewind`，各命令使用的参数如下:

**PgBaseBackup**

| 参数 | 含义
|-|-
| -c fast | Sets checkpoint mode to fast
| -D target_datadir | 目标目录
| -h source_host | 源端目录
| -p source_port | 源端 port
| --create-slot  | 进行 backup 之间创建 --slot 指定的 replication slot
| --slot         | 要创建的 replcation slot 名称
| --wal-method stream | basebackup 忽略 WAL 文件的传输，而是由另外一个进程在 basebackup 的同时流式传输 WAL 文件
| --no-verify-checksums | 忽略 checksums
| --write-recovery-conf | 生成 Recovery Config，PG 12 之前为 recovery.conf，之后为 postgresql.auto.conf
| --force-overwrite | gpdb 引入的一个选项，在做 pg_basebackup 之前不需要目的端目录为空，而是在执行的过程中将需要从源端拷贝的目录或文件删除
| --target-gp-dbid  | gpdb 引入的选项，为了适配 Greenplum 用户定义的 tablespaces
| -E ./db_dumps -E ./promote | 忽略的目录
| --progress        | 显示进度百分比
| --verbose         | 显示更多的日志信息

<br>
**PgRewind**

| 参数 | 含义
|-|-
| --write-recovery-conf | 生成 Recovery Config，PG 12 之前为 recovery.conf，之后为 postgresql.auto.conf
| --slot=internal_wal_replication_slot | 写入 Recovery Config 的 replication slot
| --source-server=CONNSTR   | 源端数据库
| --target-pgdata=DIRECTORY | 目的端数据目录
| --progress | 显示更多的日志信息

<br>
值得注意的是，无论增量修复还是全量修复后，静态的实例状态未必处于一致的状态，都需要启动实例进入 recovery mode 回放日志来达到一致的状态。主要原因有两个:

1. 有些文件在拷贝过之后在源端有修改，这就使得数据目录不是一个完整的快照状态
2. 读取的文件可能不是完整的数据页（由于 pg 的 PAGE_SIZE 和 OS 的 PAGE_SIZE 不同），需要源端开启 full_page_writes 选项保证正确修复

关于 postgres 全量修复和增量修复更详细的解读可参考 [pg_basebackup 代码浅析](/2023/04/19/pg-basebackup-code-analysis.html) 和 [pg_rewind 代码浅析](/2023/04/25/pg-rewind-code-analysis.html)。

#### ☆ 差异化修复

gpdb 作为分析型数据库通常实例的数据量较大，在宕机之后如果能增量修复具有显著的性能优势，但有时由于 WAL 日志缺失会导致增量修复的失败，且如果 pg_rewind 中断了，不建议进行重试，只能使用更耗时的全量修复。在全量修复过程中如果网络抖动，极有可能在同步了 90% 数据的时候修复失败，只能重新执行全量修复，费力劳心。

因此社区提出了使用 `rsync` 进行增量修复的方案，在数据量大且修改较少的情况下，rsync 的算法（[Rolling checksum & Checksum searching](https://rsync.samba.org/tech_report/tech_report.html)）能够极大减少数据传输量，且支持断点续传。并且 rsync 已经是 gpdb 本地升级失败回退工作的依赖工具，无需额外安装。

![rsync algorithm](/assets/2023/rsync_algorithm.jpg)

**差异化修复算法**

差异化修复的逻辑和 pg_basebackup 非常相似，其算法步骤:

1. On source: Skip differential recovery if the segment is already in backup
2. On source: drop the replication slot if exists
3. On source: start backup: pg_start_backup('differential_backup')
4. On source: create a replication slot
5. On target: sync pg_data of the mirror with its primary using rsync
6. On target: sync table space created outside the data directory
7. On source: stop backup: pg_stop_backup()
8. On target: write configuration file: pg_basebackup -D <backup_dir> --write-conf-files-only
9. On target: sync wal and pg_control file
10. On target: update port in the configuration
11. On target: start segment
12. On Source: if any exception is raised during steps 2 to 11 the program should stop the backup

pg_basebackup 使用两个进程对数据和 WAL 分别拉取，在发送 `BASE_BACKUP` 命令之后、fork LogStreamer 之前创建了 physical replication slot，上述的步骤 3-8 对应 pg_basebackup 的 `BaseBackup` 逻辑，步骤 9 中的 **sync wal** 对应 pg_basebackup 的 `LogStreamerMain` 逻辑，pg_basebackup 使用 `perform_base_backup` 发送完所有 tablespaces 和 DataDir 中的数据后再发送 pg_control 文件，差异化修复则是在发送完 WAL 文件后再同步 pg_control，不影响正确性。步骤 10-11 是 gpsegrecovery 不同修复类型共有的启动逻辑。

代码的主要逻辑是在 `gpsegrecovery.py` 添加了一个 **DifferentialRecovery** 类，该类的 `run` 方法实现了上述算法：

```python
def run(self):
    self.logger.info("Running differential recovery with progress output temporarily in {}".format(
        self.recovery_info.progress_file))
    self.error_type = RecoveryErrorType.DIFFERENTIAL_ERROR

    """ Drop replication slot 'internal_wal_replication_slot' """
    if self.replication_slot.slot_exists() and not self.replication_slot.drop_slot():
        raise Exception("Failed to drop replication slot")

    """ start backup with label differential_backup """
    self.pg_start_backup()

    try:
        if not self.replication_slot.create_slot():
            raise Exception("Failed to create replication slot")

        """ rsync pg_data and tablespace directories including all the WAL files """
        self.sync_pg_data()

        """ rsync tablespace directories which are out of pg-data-directory """
        self.sync_tablespaces()

    finally:
        # Backup is completed, now run pg_stop_backup which will also remove backup_label file from
        # primary data_dir
        if is_seg_in_backup_mode(self.recovery_info.source_hostname, self.recovery_info.source_port):
            self.pg_stop_backup()


    """ Write the postresql.auto.conf and internal.auto.conf files """
    self.write_conf_files()

    """ sync pg_wal directory and pg_control file just before starting the segment """
    self.sync_wals_and_control_file()

    self.logger.info(
        "Successfully ran differential recovery for dbid {}".format(self.recovery_info.target_segment_dbid))

    """ Updating port number on conf after recovery """
    self.error_type = RecoveryErrorType.UPDATE_ERROR
    update_port_in_conf(self.recovery_info, self.logger)

    self.error_type = RecoveryErrorType.START_ERROR
    start_segment(self.recovery_info, self.logger, self.era)
```

其中:

- `sync_pg_data`、`sync_tablespaces`、`sync_wals_and_control_file` 用 **Rsync** 传输数据
- `write_conf_files` 使用 `PgBaseBackup` 新增的 writeconffilesonly 选项生成 `recover.conf` 文件
- `update_port_in_conf`、`start_segment` 是同步之后的重启逻辑，与增量修复和全量修复相同

**Rsync** 使用的参数:

| 参数 | 含义
|-|-
| -r | 递归同步目录下的目录和文件
| -v | 显示更多信息
| -a | archive mode; same as -rlptgoD
| -c | 使用 checksum 进行文件对比而 mod-time 和 size
| --progress | 显示进度
| --delete | 删除目标端多于的文件
| --exclude | 忽略的文件或目录，如 pg_log
| SRC | 源端目录
| DEST | 目的端目录

<br>

#### ☆ 使用场景

增量修复失败可使用差异化修复，相较于全量修复速度更快，且差异化修复可多次运行。全量修复则用于新增节点或在另一台机器上重搭。
