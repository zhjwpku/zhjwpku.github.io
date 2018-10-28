---
layout: post
title: Dapper and Blkin
date: 2018-10-28 15:00:00 +0800
tags:
- paper
---

[Dapper](/assets/201810/dapper.pdf) 是 Google 生产环境下的分布式跟踪系统，其设计之初参考了 Magpie 和 X-Trace 等分布式系统的理念，具有低损耗、应用透明的、大范围部署等特点。本文介绍 Dapper 的基本原理及其一个 C++ 开源实现 [Blkin][blkin]。

<h4>Dapper 原理</h4>

**span**

span 对应分布式系统中的一个进程或微服务（相互之间通过RPC进行通信），每个 span 结构中应包含进程或微服务的标识（通常是服务名 name），RPC请求中携带的 parent_span_id，及当前进程或微服务生成的 span_id。

**annotation**

annotation 对应每一条 Trace Log 中有用的信息，一般为事件或Key-Value对，常见的事件包含：

    const char* const CLIENT_SEND = "cs";
    const char* const CLIENT_RECV = "cr";
    const char* const SERVER_SEND = "ss";
    const char* const SERVER_RECV = "sr";
    const char* const WIRE_SEND = "ws";
    const char* const WIRE_RECV = "wr";
    const char* const CLIENT_SEND_FRAGMENT = "csf";
    const char* const CLIENT_RECV_FRAGMENT = "crf";
    const char* const SERVER_SEND_FRAGMENT = "ssf";
    const char* const SERVER_RECV_FRAGMENT = "srf";

应用程序关心的其它信息可以记录为 Key-Value 类型日志。

**trace id**

通过 span 和 annotation 可以将分布式中不同的进程或服务串在一起，因为每条 Trace Log 中记录了 parent_span_id 和 span_id，通过它们之间的关系可以将一次请求所有的 RPC 调用组合成一棵树。但是仅仅通过 span_id 之间的关系来分析，在日志量巨大的情况下是非常耗时的，因此使用一个统一的 trace id (通常是64位随机数) 来标识一次请求，分布式系统对外的一次服务所有的 RPC 请求都对应一个 trace
id，提高了日志的检索效率。

**Sampling**

trace id 另外一个用途是用来采样，将 trace id 哈希成一个 0 ≤ z ≤ 1 的值，同时设置一个全局的系数，当 z 小于这个系数时将 Trace Log 进行记录，否则滤掉。这就保证了分布式系统对外提供的每一次服务对应的不同 RPC 请求要么全部被记录，要么全部被过滤掉。

<h4>Blkin</h4>

确切来讲，blkin 自己并不能算是 Dapper 的开源实现，比如它没有收集日记的工具，而是借助 [lttng][lttng] 来将 Trace 写入日志文件并进行收集。

其跟 Dapper 相关的结构体列举如下:

```
 /**
  * @struct blkin_endpoint
  * Information about an endpoint of our instrumented application where
  * annotations take place
  */
 struct blkin_endpoint {
     const char *ip;
     int16_t port;
     const char *name;
 };

 /**
  * @struct blkin_trace_info
  * The information exchanged between different layers offering the needed
  * trace semantics
  */
 struct blkin_trace_info {
     int64_t trace_id;
     int64_t span_id;
     int64_t parent_span_id;
 };

  /**
  * @struct blkin_trace
  * Struct used to define the context in which an annotation happens
  */
 struct blkin_trace {
     const char *name;
     struct blkin_trace_info info;
     const struct blkin_endpoint *endpoint;
 };

 /**
  * @typedef blkin_annotation_type
  * There are 2 kinds of annotation key-val and timestamp
  */
 typedef enum {
     ANNOT_STRING = 0,
     ANNOT_INTEGER,
     ANNOT_TIMESTAMP
 } blkin_annotation_type;

 /**
  * @struct blkin_annotation
  * Struct carrying information about an annotation. This information can either
  * be key-val or that a specific event happened
  */
 struct blkin_annotation {
     blkin_annotation_type type;
     const char *key;
     union {
     const char *val_str;
     int64_t val_int;
     };
     const struct blkin_endpoint *endpoint;
 };
```

Ceph 使用了 blkin 来做分布式跟踪，略。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [zipkinCore][zipkinCore]
</span>

[blkin]: https://github.com/ceph/blkin
[zipkinCore]: https://github.com/openzipkin/zipkin-api/blob/master/thrift/zipkinCore.thrift
[lttng]: https://lttng.org/docs/v2.10/
