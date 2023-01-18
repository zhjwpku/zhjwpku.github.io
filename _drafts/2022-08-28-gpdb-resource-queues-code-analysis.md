---
layout: post
title: Greenplum 资源队列（Resource Queue）源码分析
date: 2022-08-28 00:00:00 +0800
tags:
- gpdb
---

Greenplum 提供两种资源管理的方式 — `Resource Queues` 和 `Resource Groups`。本文对 `Resource Queue` 的实现原理及其主要源码进行分析。

<h4>简介</h4>

为了避免一个 QUERY 耗尽所有资源及 QUERY 之间的资源竞争，Greenplum 使用 `Resource Queue` 来实现资源的管理。通过创建 `Resource Queue` 并将用户（user-level roles）指定给它，来达到资源隔离的目的。

`Resource Queue` 实现了三个纬度的资源隔离:

- 并发控制（可以通过 ACTIVE_STATEMENTS 或 MAX_COST 来限制）
- 内存（通过 MEMORY_LIMIT 限制）
- CPU（通过 PRIORITY 限制）

其中单个 QUERY 的内存限制的计算与并发度控制的方式相关，当使用 ACTIVE_STATEMENTS 设置并发度时，每个查询能够分配的内存为:

```
MEMORY_LIMIT / ACTIVE_STATEMENTS
```

当时用 MAX_COST 设置并发度时，每个查询能够使用的内存为:

```
MEMORY_LIMIT * (query_cost / MAX_COST)
```

<h4>系统表</h4>

`Resource Queue` 相关的系统表有三个:

**1. src/include/catalog/pg_resourcetype.h**

`pg_resourcetype` 中预定义了六种资源类型（src/include/catalog/pg_resourcetype.dat），其中前四种是 `pg_resqueue` 后四列的映射。

```c
/* ----------------
 *		pg_resourcetype definition.  cpp turns this into
 *		typedef struct FormData_pg_resourcetype
 * ----------------
 */

CATALOG(pg_resourcetype,6059,ResourceTypeRelationId) BKI_SHARED_RELATION
{
	Oid			oid;				/* oid */
	NameData	resname;			/* name of resource type  */
	int16		restypid;			/* resource type id  */
	bool		resrequired;		/* if required, user must specify during CREATE */
	bool		reshasdefault;		/* create a default entry for optional type */
	bool		reshasdisable;		/* whether the type can be removed or shut off */
#ifdef CATALOG_VARLEN
	text		resdefaultsetting;	/* default resource setting  */
	text		resdisabledsetting;	/* value that turns it off  */
#endif
} FormData_pg_resourcetype;

#define PG_RESRCTYPE_ACTIVE_STATEMENTS	1	/* rsqcountlimit: count  */
#define PG_RESRCTYPE_MAX_COST			2	/* rsqcostlimit: max_cost */
#define PG_RESRCTYPE_MIN_COST			3	/* rsqignorecostlimit: min_cost */
#define PG_RESRCTYPE_COST_OVERCOMMIT	4	/* rsqovercommit: cost_overcommit*/
/* start of "pg_resourcetype" entries... */
#define PG_RESRCTYPE_PRIORITY			5	/* backoff.c: priority queue */
#define PG_RESRCTYPE_MEMORY_LIMIT		6	/* memquota.c: memory quota */
```

```sql
postgres=# select * from pg_resourcetype;
 oid  |      resname      | restypid | resrequired | reshasdefault | reshasdisable | resdefaultsetting | resdisabledsetting 
------+-------------------+----------+-------------+---------------+---------------+-------------------+--------------------
 6454 | active_statements |        1 | f           | t             | t             | -1                | -1
 6455 | max_cost          |        2 | f           | t             | t             | -1                | -1
 6456 | min_cost          |        3 | f           | t             | t             | -1                | 0
 6457 | cost_overcommit   |        4 | f           | t             | t             | -1                | -1
 6458 | priority          |        5 | f           | t             | f             | medium            | 
 6459 | memory_limit      |        6 | f           | t             | t             | -1                | -1
(6 rows)
```

**2. src/include/catalog/pg_resqueue.h**

`pg_resqueue` 保存 `Resource Queue` 的名称及并发控制相关的资源类型。

```c
/* ----------------
 *		pg_resqueue definition.  cpp turns this into
 *		typedef struct FormData_pg_resqueue
 * ----------------
 */

CATALOG(pg_resqueue,6026,ResQueueRelationId) BKI_SHARED_RELATION
{
	Oid			oid;				/* oid */
	NameData	rsqname;			/* name of resource queue */
	float4		rsqcountlimit;		/* max active count limit */
	float4		rsqcostlimit;		/* max cost limit */
	bool		rsqovercommit;		/* allow overcommit on suitable  limits */
	float4		rsqignorecostlimit;	/* ignore queries with cost less than */
} FormData_pg_resqueue;
```

```sql
postgres=# select * from pg_resqueue;
 oid  |  rsqname   | rsqcountlimit | rsqcostlimit | rsqovercommit | rsqignorecostlimit 
------+------------+---------------+--------------+---------------+--------------------
 6055 | pg_default |            10 |           -1 | f             |                  0
(1 row)
```

**3. src/include/catalog/pg_resqueuecapability.h**

`pg_resqueuecapability` 用于保存 `Resource Queue` 对不同资源类型的限制，其设计支持增加不同的资源类型，但目前只保存了内存和 CPU 的资源限制（即 restypeid = 5 或 6）。

```c
/* ----------------
 *		pg_resqueuecapability definition.  cpp turns this into
 *		typedef struct FormData_pg_resqueuecapability
 * ----------------
 */

CATALOG(pg_resqueuecapability,6060,ResQueueCapabilityRelationId) BKI_SHARED_RELATION
{
	Oid		resqueueid;	/* OID of the queue with this capability  */
	int16	restypid;	/* resource type id (key to pg_resourcetype)  */
#ifdef CATALOG_VARLEN			/* variable-length fields start here */
	text	ressetting;	/* resource setting (opaque type)  */
#endif
} FormData_pg_resqueuecapability;
```

```sql
postgres=# select * from pg_resqueuecapability;
 resqueueid | restypid | ressetting 
------------+----------+------------
       6055 |        5 | medium
       6055 |        6 | -1
(2 rows)
```

另外 `src/include/catalog/pg_authid.h` 中新增 `rolresqueue` 字段，用于表示该角色归属的 `Resource Queue`。

<h4>创建资源队列</h4>

操作资源队列的代码在 `src/backend/commands/queue.c`，其中 `CreateQueue(CreateQueueStmt *stmt)` 用于创建 `Resource Queue`，其主要逻辑:

1. 权限检查 - only superuser can create queues
2. 从 statement node tree 提取创建 res queue 的选项
3. 检查 ACTIVE_STATEMENTS 和 MAX_COST 至少有一个被指定
4. 检查 resource queue name 不能为 'none'，('none' is used to signify no queue in ALTER ROLE)
5. 检查 resource queue name 没有重名
6. 插入 pg_resqueue
7. 插入 pg_resourcetype，通过调用 AlterResqueueCapabilityEntry 实现
8. 如果为 GP_ROLE_DISPATCH，调用 ResCreateQueue 创建 resource queue 的内存结构
9. 如果为 GP_ROLE_DISPATCH，调用 CdbDispatchUtilityStatement 将命令发送给各个 GP_ROLE_EXECUTE 节点

该文件还定义了 `AlterQueue`、`DropQueue` 等函数，这里不再赘述。

<h4>Scheduler</h4>



<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [Using Resource Queues](https://docs.vmware.com/en/VMware-Tanzu-Greenplum/6/greenplum-database/GUID-admin_guide-workload_mgmt.html)<br>
</span>

