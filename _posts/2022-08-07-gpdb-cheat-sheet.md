---
layout: post
title: Greenplum 常用命令
date: 2022-08-07 23:00:00 +0800
tags:
- postgres
- gpdb
---

记录 Greenplum 常用的调优、运维、定位问题相关的命令。

<h4>Utilities</h4>

**gpstate**

```shell
# List the port numbers used throughout the Greenplum Database system.
gpstate -p

# Displays detailed status information for the Greenplum Database system.
gpstate -s

# Display the Greenplum Database software version information for each instance.
gpstate -i

# Display details of any current gpexpand.
gpstate -x

# Display details of the standby coordinator host if configured.
gpstate -f
```

**gpconfig**

set, unset, or view configuration parameters from the postgresql.conf files of all instances (master, segments, and mirrors) in your Greenplum Database system.

```shell
# 查看集群各实例配置
gpconfig -s <param_name> 

# 设置集群各实例配置
gpconfig -c <param_name> -v <value>
```

**gpstop**

```shell
# Reload the postgresql.conf and pg_hba.conf files after making configuration changes but do not shutdown the Greenplum Database array
gpstop -u

# Stop all segment instances and then restart the system
gpstop -r

# Stop a Greenplum Database system in smart mode without prompting the user for confirmation
gpstop -a
```

**gprecoverseg**

```shell
# incremental recovery
gprecoverseg -a

# full recovery
gprecoverseg -a -F

# rebalance segments
gprecoverseg -ar
```

**连接 segment 实例**

```shell
# GP7
PGOPTIONS="-c gp_role=utility" psql -p 7002 -d postgres

# GP6
PGOPTIONS="-c gp_session_role=utility" psql -p 4000 -d db01
```

<h4>Extensions</h4>

**pageinspect**

```sql
-- load pageinspect
CREATE EXTENSION pageinspect;

-- check page layout
SELECT lower, upper, special, pagesize FROM page_header(get_raw_page('accounts',0));

-- check page items
SELECT * FROM heap_page_items(get_raw_page('accounts',0)) \gx

-- check infomask
SELECT '(0,'||lp||')' AS ctid,
       CASE lp_flags
         WHEN 0 THEN 'unused'
         WHEN 1 THEN 'normal'
         WHEN 2 THEN 'redirect to '||lp_off
         WHEN 3 THEN 'dead'
       END AS state,
       t_xmin as xmin,
       t_xmax as xmax,
       (t_infomask & 256) > 0  AS xmin_commited,
       (t_infomask & 512) > 0  AS xmin_aborted,
       (t_infomask & 1024) > 0 AS xmax_commited,
       (t_infomask & 2048) > 0 AS xmax_aborted,
       t_ctid
FROM heap_page_items(get_raw_page('accounts',0)) \gx

CREATE FUNCTION heap_page(relname text, pageno integer)
RETURNS TABLE(ctid tid, state text, xmin text, xmax text, hhu text, hot text, t_ctid tid)
AS $$
SELECT (pageno,lp)::text::tid AS ctid,
       CASE lp_flags
         WHEN 0 THEN 'unused'
         WHEN 1 THEN 'normal'
         WHEN 2 THEN 'redirect to '||lp_off
         WHEN 3 THEN 'dead'
       END AS state,
       t_xmin || CASE
         WHEN (t_infomask & 256+512) = 256+512 THEN ' (f)'
         WHEN (t_infomask & 256) > 0 THEN ' (c)'
         WHEN (t_infomask & 512) > 0 THEN ' (a)'
         ELSE ''
       END AS xmin,
       t_xmax || CASE
         WHEN (t_infomask & 1024) > 0 THEN ' (c)'
         WHEN (t_infomask & 2048) > 0 THEN ' (a)'
         ELSE ''
       END AS xmax,
       CASE WHEN (t_infomask2 & 16384) > 0 THEN 't' END AS hhu,
       CASE WHEN (t_infomask2 & 32768) > 0 THEN 't' END AS hot,
       t_ctid
FROM heap_page_items(get_raw_page(relname,pageno))
ORDER BY lp;
$$ LANGUAGE SQL;

SELECT * FROM heap_page('accounts',0);

CREATE FUNCTION index_page(relname text, pageno integer)
RETURNS TABLE(itemoffset smallint, ctid tid)
AS $$
SELECT itemoffset,
       ctid
FROM bt_page_items(relname,pageno);
$$ LANGUAGE SQL;
```

**pgstattuple**

```sql
-- load pgstattuple
CREATE EXTENSION pgstattuple;

-- `tuple_percent` shows the percentage of useful data
SELECT * FROM pgstattuple('vac') \gx

-- `avg_leaf_density` shows the percentage of useful information (in leaf pages)
SELECT * FROM pgstatindex('vac_s') \gx
```

<h4>SQL</h4>

```sql
-- show the user who is logged in as.
SELECT current_user;
\conninfo

-- 显示 segment 角色，master 上运行会显示 unknown, primary 上会显示 p;
show gp_preferred_role;

-- 显示默认存储参数
postgres=# show gp_default_storage_options;
           gp_default_storage_options
-------------------------------------------------
 blocksize=32768,compresstype=none,checksum=true
(1 row)

-- 查看表 accounts 对应的文件路径
postgres=# SELECT pg_relation_filepath('accounts');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)

postgres=# SELECT oid FROM pg_database WHERE datname = 'postgres';
 oid
-----
   5
(1 row)

postgres=# SELECT relfilenode FROM pg_class WHERE relname = 'accounts';
 relfilenode
-------------
       16389
(1 row)

-- List the databases in the server and show their names, owners, character set encodings, and access privileges.
\l[+]

-- 查看当前连接数据库
SELECT current_database();

-- 查看实例 block_size
SELECT current_setting('block_size');

-- 查看表的 oid，由于 oid 是隐藏列，所以需要显示指定要查询该列
SELECT oid, * FROM pg_class WHERE relname = 'lineitem';

-- 在各个 segment 节点查询
SELECT random() FROM gp_dist_random('gp_id');
SELECT * FROM gp_dist_random('lineitem') LIMIT 10;

-- 查看表是否倾斜
SELECT gp_segment_id, count(*) FROM lineitem GROUP BY gp_segment_id;

-- 查看表大小
SELECT pg_size_pretty(pg_relation_size('lineitem'));

-- 查看表和索引的总大小
SELECT pg_size_pretty(pg_total_relation_size('lineitem'));

-- 查看一个 schema 中的数据量
SELECT schemaname, round(sum(pg_total_relation_size(schemaname||'.'||tablename))/1024/1024) "Size_MB"
  FROM pg_tables 
  WHERE schemaname='public' 
  GROUP BY 1;

-- 查看数据库的数据量
SELECT pg_size_pretty(pg_database_size('postgres'));
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;

-- 查看 gp segment 和 mirror 的配置及节点状态
SELECT * FROM gp_segment_configuration \gx
SELECT * FROM gp_segment_configuration WHERE mode <> 's' OR status <> 'u';

-- 查看节点状态变更历史
SELECT * FROM gp_configuration_history;

-- 查看 postmaster 启动时间
SELECT pg_postmaster_start_time();

-- 杀 session
SELECT pg_terminate_backend(pid);
```

<h4>GUC</h4>

**gp_select_invisible**

```sql
-- 开启该变量可查看其它事务未提交的记录以及删除未被 vacuum 的记录
set gp_select_invisible=true;

-- 查看 catalog 表是否膨胀
set gp_select_invisible=false;
select count(*) from pg_attribute;
set gp_select_invisible=true;
select count(*) from pg_attribute;
-- 令 n 为后者与前者的比值，如果 n > 10 则认为是严重 bloat，n 在 [4, 10] 之间则认为是中等 bloat
```

**misc**

```sql
-- 打开如下选项，在日志中会打印 plan tree
set debug_print_plan to on;

-- Print debug information for Gang allocating
set gp_log_gang='debug';
```


<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [gpconfig doc](https://docs.vmware.com/en/VMware-Tanzu-Greenplum/6/greenplum-database/GUID-utility_guide-ref-gpconfig.html)<br>
2 [Show database bloat](https://wiki.postgresql.org/wiki/Show_database_bloat)<br>
3 [Greenplum通过gp_dist_random('gp_id') 在所有节点调用某个函数](https://developer.aliyun.com/article/7593)<br>
4 [Useful Greenplum SQLs](http://www.openkb.info/2014/05/useful-greenplum-sqls.html)<br>
</span>

