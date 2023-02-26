---
layout: post
title: Greenplum FTS 服务代码解析
date: 2023-02-25 08:00:00 +0800
categories: category
tags:
- gpdb
- fts
---

FTS (Fault Tolerance Service) 是 Greenplum 提供的用于检测 Segment 节点故障的服务，通过在 Coordinator 维护一个 ftsprobe BackgroundWorker，定期发送探活消息来检测 Segment 的健康状态，并根据不同响应做进一步决策，来保证 GPDB 集群的高可用。需要注意的是，FTS 并不能解决 Coordinator 的高可用问题。

FTS 功能实现主要在以下文件中：

```
src/backend/fts/fts.c                       bgworker 的主函数和启动逻辑
src/backend/fts/ftsprobe.c                  探测的逻辑
src/backend/fts/ftsmessagehandler.c         Segment 接收 fts 消息并进行处理的逻辑
src/backend/replication/gp_replication.c    主备同步状态相关逻辑
src/backend/cdb/cdbfts.c                    维护 FtsProbeInfo 信息，并提供触发 probe 的内部函数
```

#### FTS 的不同角色

在 fts.c 文件中定义了两个初始值为 false 的 bool 变量：`am_ftsprobe` 和 `am_ftshandler`，FTS bgworker 在 启动函数 `FtsProbeMain` 中将 `am_ftsprobe` 置为 true，ftsprobe 进程连接 Segment 时，Segment 在 fork 出的 backend 在 `ProcessStartupPacket` 中将 am_ftshandler 置为 true。因此 FTS 模块主要涉及两个角色 —— ftsprobe 和 ftshandler。

![fts roles](/assets/2023/fts_roles.svg)

#### 👉🏻 ftsprobe

ftsprobe 的逻辑在 fts.c、ftsprobe.c 及 cdbfts.c 中实现，它作为一个标准的 BackgroundWorker，hardcode 在 Postgres 的 `PMAuxProcList` 结构中:

```c
static BackgroundWorker PMAuxProcList[MaxPMAuxProc] =
{
    {"ftsprobe process", "ftsprobe process",
     BGWORKER_SHMEM_ACCESS | BGWORKER_BACKEND_DATABASE_CONNECTION,
     BgWorkerStart_DtxRecovering, /* no need to wait dtx recovery */
     0, /* restart immediately if ftsprobe exits with non-zero code */
     "postgres", "FtsProbeMain", 0, {0}, 0,
     FtsProbeStartRule},

    // ...
}
```

Postmaster 根据 `FtsProbeStartRule` 的逻辑（Gp_role == GP_ROLE_DISPATCH，即只在 Coordinator 中启动）判断是否需要启动该进程，并且当该进程因为某种原因退出之后，Postmaster 主循环中会将 ftsprobe 进程重启拉起。

`FtsProbeMain` 中调用了一个无限循环函数 `FtsLoop`:

```c
static void FtsLoop()
{
    while (true)
    {
        SpinLockAcquire(&ftsProbeInfo->lock);
        ftsProbeInfo->start_count++;                   // 记录 fts 触发次数
        SpinLockRelease(&ftsProbeInfo->lock);

        cdbs = readCdbComponentInfoAndUpdateStatus();  // 从 gp_segment_configuration 系统表读取集群 segment 元数据
        
        oldContext = MemoryContextSwitchTo(probeContext);
        updated_probe_state = FtsWalRepMessageSegments(cdbs);    // 每一轮 probe 的主逻辑
        MemoryContextSwitchTo(oldContext);

        /* Bump the version if configuration was updated. */
        if (updated_probe_state)
        {
            StartTransactionCommand();
            writeGpSegConfigToFTSFiles();              // 将改动后的 segment 元数据写入 gpsegconfig_dump 文件
            CommitTransactionCommand();

            ftsProbeInfo->status_version++;
        }

        /* free current components info and free ip addr caches */    
        cdbcomponent_destroyCdbComponents();           // 释放 cdbs 占用的内存空间

        SpinLockAcquire(&ftsProbeInfo->lock);
        ftsProbeInfo->done_count = ftsProbeInfo->start_count;  // 标记该次 probe 完成
        SpinLockRelease(&ftsProbeInfo->lock);

        /* check if we need to sleep before starting next iteration */
        elapsed = time(NULL) - probe_start_time;
        timeout = elapsed >= gp_fts_probe_interval ? 0 : 
                gp_fts_probe_interval - elapsed;       // 计算下一次 probe 之前需要睡眠的时间

        if (probe_requested)
        {
            timeout = 0;    // 如果在 probe 的过程中有外部请求重新进行 probe，那么不需进入睡眠
        }

        rc = WaitLatch(&MyProc->procLatch,                       // ftsprobe 进程进入睡眠
               WL_LATCH_SET | WL_TIMEOUT | WL_POSTMASTER_DEATH,  // 可以被外部请求或 Postmaster 退出唤醒
               timeout * 1000L,
               WAIT_EVENT_FTS_PROBE_MAIN);

        ResetLatch(&MyProc->procLatch);

        /* emergency bailout if postmaster has died */
        if (rc & WL_POSTMASTER_DEATH)
            proc_exit(1);
    }

    return;
}
```

除了定时触发 FTS probe（通过 gp_fts_probe_interval GUC 控制，默认为 60s），还可以通过 UDF `gp_request_fts_probe_scan()` 或通过调用内部函数 `FtsNotifyProber()` 来触发 probe 并同步等待 probe 过程的完成（例如在创建 gang 失败后）。`gp_request_fts_probe_scan()` 也是通过调用 `FtsNotifyProber()` 实现的。

```c
void FtsNotifyProber(void)
{
    Assert(Gp_role == GP_ROLE_DISPATCH);
    int32            initial_started;
    int32            started;
    int32            done;

    if (am_ftsprobe)    // 如果是 ftsprobe bgworker，直接返回
        return;

    SpinLockAcquire(&ftsProbeInfo->lock);
    initial_started = ftsProbeInfo->start_count;    // 记录当前的 probe 计数
    SpinLockRelease(&ftsProbeInfo->lock);

    SendPostmasterSignal(PMSIGNAL_WAKEN_FTS);        // 给 postmaster 发信号

    /* Wait for a new fts probe to start. */
    for (;;)
    {
        SpinLockAcquire(&ftsProbeInfo->lock);
        started = ftsProbeInfo->start_count;
        SpinLockRelease(&ftsProbeInfo->lock);

        if (started != initial_started)            // 等待新一轮 probe 开始
            break;

        CHECK_FOR_INTERRUPTS();
        pg_usleep(50000);
    }

    /* Wait until current probe in progress is completed */
    for (;;)
    {
        SpinLockAcquire(&ftsProbeInfo->lock);
        done = ftsProbeInfo->done_count;
        SpinLockRelease(&ftsProbeInfo->lock);

        if (done - started >= 0)            // 等待新一轮 probe 结束
            break;

        CHECK_FOR_INTERRUPTS();
        pg_usleep(50000);
    }

}
```

Postmaster 在接收到子进程发送的 `PMSIGNAL_WAKEN_FTS` 信号之后，给 ftsprobe 进程发送 `SIGINT` 信号。ftsprobe 处理该信号的逻辑：

```c
/* SIGINT: set flag to indicate a FTS scan is requested */
static void sigIntHandler(SIGNAL_ARGS)
{
    probe_requested = true;       // 标记有新的 probe 请求，用于快速进入下一次 probe

    if (MyProc)
        SetLatch(MyLatch);        // 唤醒睡眠的 ftsprobe 进程，进入下一次 probe
}
```

`FtsWalRepMessageSegments` 作为单次 probe 的入口，通过向各个 Segment 的 primary 节点发送消息来获取所有 Segment 的健康状态。

```c
bool FtsWalRepMessageSegments(CdbComponentDatabases *cdbs)
{
    bool is_updated = false;
    fts_context context;

    FtsWalRepInitProbeContext(cdbs, &context);    // 根据从 gp_segment_configuration 读取的元数据初始化 fts_context
    InitPollFds(cdbs->total_segments);            // 分配 poll 多路复用结构

    while (!allDone(&context) && FtsIsActive())   // 需要确认所有 segment 的状态之后（即状态为 FTS_RESPONSE_PROCESSED）才认为一次 probe 完成
    {
        ftsConnect(&context);       // 与每个 Segment 的 primary 建立连接
        ftsPoll(&context);          // 调用 poll 检查是否有 I/O 请求
        ftsSend(&context);          // 发送 fts message
        ftsReceive(&context);       // 接收 fts handler 发送回来的消息
        processRetry(&context);     // 检查是否需要对默写 segment 进行重试
        is_updated |= processResponse(&context);    // 处理消息
    }
    int i;
    if (!FtsIsActive())
    {
        for (i = 0; i < context.num_pairs; i++)
        {
            if (context.perSegInfos[i].conn)
            {
                PQfinish(context.perSegInfos[i].conn);
                context.perSegInfos[i].conn = NULL;
            }
        }
    }

    pfree(context.perSegInfos);
    pfree(PollFds);
    return is_updated;
}
```

`FtsWalRepInitProbeContext` 将从 gp_segment_configuration 系统表中读取的元数据按照 Segment primary-mirror pair 的形式保存在 `fts_segment_info` 中，并将初始状态置为 `FTS_PROBE_SEGMENT`， 如果某个 Segment 没有的 mirror 节点，则不需要进行探测。

while 循环中的六步对维护了所有 Segment 的状态变化，单个 Segment 的 probe 状态机可以总结为下图：

![fts state machine](/assets/2023/fts.svg)

可以看出，这些状态可以分为三组，每组都有四个状态。SUCCESS 代表成功从 Segment 获得了响应，Failed 则代表不能获得响应，失败 `gp_fts_probe_retries` 次之后，进入状态机的下一个状态。ftsprobe 进程可能随时退出，为了避免由于 `partial operation` 造成不可恢复的状态，FTS 先更新系统表后才做 promote 或 sync off 等操作，详情见 [Stability guarantee](https://github.com/greenplum-db/gpdb/wiki/Master-auto-failover#stability-guarantee)。

在探测到 Segment 状态有变化之后，ftsprobe 在事务中更新 `gp_configuration_history` 和 `gp_segment_configuration` 两张系统表。

#### 👉🏻 ftshandler

ftshandler 的逻辑在 ftsmessagehandler.c 和 gp_replication.c 中实现。Segment 在收到 ftsprobe 发来的消息后，通过 `HandleFtsMessage` 来处理消息并返回处理后的结果。

```c
void HandleFtsMessage(const char* query_string)
{
    int dbid;
    int contid;
    char message_type[FTS_MSG_MAX_LEN];
    int error_level;

    if (sscanf(query_string, FTS_MSG_FORMAT,
               message_type, &dbid, &contid) != 3)    // 解析 fts 消息类型
    {
        ereport(ERROR,
                (errmsg("received invalid FTS query: %s", query_string)));
    }

    if (dbid != GpIdentity.dbid)
        ereport(error_level,
                (errmsg("message type: %s received dbid:%d doesn't match this segments configured dbid:%d",
                        message_type, dbid, GpIdentity.dbid)));

    if (contid != GpIdentity.segindex)
        ereport(error_level,
                (errmsg("message type: %s received contentid:%d doesn't match this segments configured contentid:%d",
                        message_type, contid, GpIdentity.segindex)));

    if (strncmp(query_string, FTS_MSG_PROBE,
                strlen(FTS_MSG_PROBE)) == 0)
        HandleFtsWalRepProbe();                            // 处理 probe 消息
    else if (strncmp(query_string, FTS_MSG_SYNCREP_OFF,
                     strlen(FTS_MSG_SYNCREP_OFF)) == 0)
        HandleFtsWalRepSyncRepOff();                       // 处理 sync off 消息
    else if (strncmp(query_string, FTS_MSG_PROMOTE,
                     strlen(FTS_MSG_PROMOTE)) == 0)
        HandleFtsWalRepPromote();                          // 处理 promote 消息
    else
        ereport(ERROR,
                (errmsg("received unknown FTS query: %s", query_string)));
}

```

- `HandleFtsWalRepProbe` 通过调用 `GetMirrorStatus` 来查看相应的 `WalSnd` 结构信息进而获取 mirror 的状态；
- `HandleFtsWalRepSyncRepOff` 调用 `UnsetSyncStandbysDefined` 将 `synchronous_standby_names` 设置为 `''`，并调用 `GetMirrorStatus` 将 mirror 状态添加到响应消息中；
- `HandleFtsWalRepPromote` 调用 `UnsetSyncStandbysDefined` 将 `synchronous_standby_names` 设置为 `''`，然后调用 `CreateReplicationSlotOnPromote` 创建 replication slot，最后调用 `SignalPromote` 将备提升为主。

gp_replication.c 在共享内存中维护了一个全局变量 `FTSRepStatusCtl` 来记录 mirror 是否一直处于 crash 重启的状态，用于判断 mirror 状态为 down 时是否需要重试, 其结构如下：

```c
/*
 * Each GPDB primary-mirror pair has a FTSReplicationStatus in shared memory.
 *
 * Mainly used to track replication process for FTS purpose.
 *
 * This struct is protected by its 'mutex' spinlock field. The walsender
 * and FTS probe process will access this struct.
 */
typedef struct FTSReplicationStatus
{
    NameData    name;             /* The slot's identifier, ie. the replicaton application name */
    slock_t     mutex;            /* lock, on same cacheline as effective_xmin */
    bool        in_use;           /* is this slot defined */

    /*
     * For GPDB FTS purpose, if the the primary, mirror replication keeps crash
     * continuously and attempt to create replication connection too many times,
     * FTS should mark the mirror down.
     * If the connection established, clear the attempt count to 0.
     * See more details in FTSGetReplicationDisconnectTime.
     */
    uint32      con_attempt_count;

    /*
     * Records time, either during initialization or due to disconnection.
     * This helps to detect time passed since mirror didn't connect.
     */
    pg_time_t   replica_disconnected_at;
} FTSReplicationStatus;

typedef struct FTSReplicationStatusCtlData
{
    /*
     * This array should be declared [FLEXIBLE_ARRAY_MEMBER], but for some
     * reason you can't do that in an otherwise-empty struct.
     */
    FTSReplicationStatus   replications[1];
} FTSReplicationStatusCtlData;

extern FTSReplicationStatusCtlData *FTSRepStatusCtl;
```

由于当前 GPDB 只能支持单个 mirror，这里只维护了一个 `FTSReplicationStatus` 结构。walsender 在状态变换时通过调用 `FTSReplicationStatusUpdateForWalState` 或 `FTSReplicationStatusMarkDisconnectForReplication` 来更新上面的结构。

当 mirror 重启之后，状态重新回到 In Sync 状态，Probe 操作会将 primary 的 `synchronous_standby_names` 置为 `*`。

#### Summary

FTS 机制极大提高了 GPDB 集群的可用性，实现简单且容易理解。但它不能用于 Coordinator 的故障检测，且不能将节点自动拉起，在主备切换之后，即使原来的主恢复了，依然不能自动恢复到主备同步的状态，需要手动调用 `gpsegrecover` 工具来调整集群状态。当集群处于不可用状态时，通过查看 `gp_configuration_history` 可以发现一些集群变更记录，辅助问题定位。

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [Master auto failover](https://github.com/greenplum-db/gpdb/wiki/Master-auto-failover).<br>
</span>

