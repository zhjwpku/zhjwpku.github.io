---
layout: post
title: Awesome PostgreSQL Learning Resources
date: 2022-07-12 22:00:00 +0800
tags:
- postgres
- gpdb
---

[PostgreSQL](https://www.postgresql.org/) 学习资料，包括但不限于文章、书籍、教程、视频。

<h4>Links for developers</h4>

- [So, you want to be a developer?](https://wiki.postgresql.org/wiki/So,_you_want_to_be_a_developer%3F) by Selena Deckelmann
- [PostgreSQL Developer FAQ](https://wiki.postgresql.org/wiki/Developer_FAQ)
- [PostgreSQL Outstanding Features](https://wiki.postgresql.org/wiki/Todo)
- [PostgreSQL Backend Flowchart](https://www.postgresql.org/developer/backend/)
- [Introduction to Hacking PostgreSQL](http://www.neilconway.org/talks/hacking/)
- [SQL-92](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)
- [SQL-99 Complete, Really](https://crate.io/docs/sql-99/en/latest/)
- [Submitting a Patch](https://wiki.postgresql.org/wiki/Submitting_a_Patch)
- [Reviewing a Patch](https://wiki.postgresql.org/wiki/Reviewing_a_Patch)
- [Review of Patch Reviewing](https://www.pgcon.org/2011/schedule/events/368.en.html) by Stephen Frost, PGCon 2011
- [Hacking the Query Planner](https://www.pgcon.org/2011/schedule/attachments/188_Planner%20talk.pdf) by Tom Lane, PGCon 2011
- [PGXN](https://pgxn.org/)  PostgreSQL Extension Network
- [HOT Understanding this important update optimization](https://www.slideshare.net/GrantMcAlister/hot-understanding-this-important-update-optimization)
- [Packaging Related Objects into an Extension](https://www.postgresql.org/docs/current/extend-extensions.html)
- [Background Worker Processes](https://www.postgresql.org/docs/current/bgworker.html)
- [Deal with Corruption](https://wiki.postgresql.org/wiki/Corruption)
- [Slow Query Questions](https://wiki.postgresql.org/wiki/Slow_Query_Questions)
- [Using EXPLAIN](https://wiki.postgresql.org/wiki/Using_EXPLAIN)
- [Foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)
- [Lock Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring)

<h4>Blogs</h4>

- [Planet](https://planet.postgresql.org/)
- [momjian.us](https://momjian.us/)
- [depesz.com](https://www.depesz.com/) by Hubert Lubaczewski
- [Robert Haas](https://rhaas.blogspot.com/)'s blogspot
- [PostgreSQL and Databases in general](http://amitkapila16.blogspot.com/) Amit Kapila's blogspot
- [What PostgreSQL has over other open source SQL databases: Part I](https://www.compose.com/articles/what-postgresql-has-over-other-open-source-sql-databases/)
- [What PostgreSQL has over other open source SQL databases: Part II](https://www.compose.com/articles/what-postgresql-has-over-other-open-source-sql-databases-part-ii/)
- [Why use Postgres](https://www.craigkerstiens.com/2017/04/30/why-postgres-five-years-later/) by Craig Kerstiens
- [Indexes in PostgreSQL](https://habr.com/en/company/postgrespro/blog/441962/) by Egor Rogov
  - [1 Introduction](https://postgrespro.com/blog/pgsql/3994098)
  - [2 Interface](https://postgrespro.com/blog/pgsql/4161264)
  - [3 Hash](https://postgrespro.com/blog/pgsql/4161321)
  - [4 Btree](https://postgrespro.com/blog/pgsql/4161516)
  - [5 GiST](https://postgrespro.com/blog/pgsql/4175817)
  - [6 SP-GiST](https://postgrespro.com/blog/pgsql/4220639)
  - [7 GIN](https://postgrespro.com/blog/pgsql/4261647)
  - [8 RUM](https://postgrespro.com/blog/pgsql/4262305)
  - [9 BRIN](https://postgrespro.com/blog/pgsql/5967830)
  - [10 Bloom](https://postgrespro.com/blog/pgsql/5967832)
- [MVCC in PostgreSQL](https://habr.com/en/company/postgrespro/blog/467437/) by Egor Rogov
  - [1. Isolation](https://habr.com/en/company/postgrespro/blog/467437/)
  - [2. Forks, files, pages](https://habr.com/en/company/postgrespro/blog/469087/)
  - [3. Row Versions](https://habr.com/en/company/postgrespro/blog/477648/)
  - [4. Snapshots](https://habr.com/en/company/postgrespro/blog/479512/)
  - [5. In-page vacuum and HOT updates](https://habr.com/en/company/postgrespro/blog/483768/)
  - [6. Vacuum](https://habr.com/en/company/postgrespro/blog/484106/)
  - [7. Autovacuum](https://habr.com/en/company/postgrespro/blog/486104/)
  - [8. Freezing](https://habr.com/en/company/postgrespro/blog/487590/)
- [Tuple internals: exposing, exploring and explaining](https://pgconf.ru/media/2016/05/13/tuple-internals.pdf) by Nikolay Shaplov, 2016
- [Understanding of Bloat and VACUUM in PostgreSQL](https://www.percona.com/blog/2018/08/06/basic-understanding-bloat-vacuum-postgresql-mvcc/) by Avinash Vallarapu, 2018
- [Tuning Autovacuum in PostgreSQL and Autovacuum Internals](https://www.percona.com/blog/2018/08/10/tuning-autovacuum-in-postgresql-and-autovacuum-internals/) by Avinash Vallarapu, 2018
- [Tuning PostgreSQL Database Parameters to Optimize Performance](https://www.percona.com/blog/2018/08/31/tuning-postgresql-database-parameters-to-optimize-performance/) by Ibrar Ahmed, 2018
- [Tune Linux Kernel Parameters For PostgreSQL Optimization](https://www.percona.com/blog/2018/08/29/tune-linux-kernel-parameters-for-postgresql-optimization/) by Ibrar Ahmed, 2018
- [MVCC and VACUUM](http://rhaas.blogspot.com/2017/12/mvcc-and-vacuum.html) by Robert Haas, 2017
- [VACUUM FULL doesn't mean "VACUUM, but better"](http://rhaas.blogspot.com/2014/03/vacuum-full-doesnt-mean-vacuum-but.html) by Robert Haas, 2014
- [DO or UNDO - there is no VACUUM](http://rhaas.blogspot.com/2018/01/do-or-undo-there-is-no-vacuum.html) by Robert Haas, 2018
- [Different Approaches for MVCC used in well known Databases](http://amitkapila16.blogspot.com/2015/03/different-approaches-for-mvcc-used-in.html) by Amit Kapila, 2015
- [Why Uber Engineering Switched from Postgres to MySQL](https://www.uber.com/blog/postgres-to-mysql-migration/) 2016
- [On Uber’s Choice of Databases](https://use-the-index-luke.com/blog/2016-07-29/on-ubers-choice-of-databases) by Markus Winand, 2016


<h4>Books</h4>

- [PostgreSQL: Introduction and Concepts](https://momjian.us/main/writings/pgsql/aw_pgsql_book/0.html) by Bruce Momjian, 2001, 介绍 PG 基本用法及一些概念
- [PostgreSQL: Up and Running, 3rd Edition][postgres_up_and_running_3rd] by Regina O. Obe, Leo S. Hsu (Nov, 2017), 介绍 PG 基本使用方法
- [Mastering PostgreSQL 11 Expert techniques to build scalable, reliable, and fault-tolerant database applications][mastering_pg_11] by Hans-Jurgen Schonig, 2018
- [The Internals of PostgreSQL][postgres_internals] by Hironobu SUZUKI
  - [Chapter 1. Database Cluster, Databases and Tables](http://www.interdb.jp/pg/pgsql01.html)
  - [Chapter 2. Process and Memory Architecture](http://www.interdb.jp/pg/pgsql02.html)
  - [Chapter 3. Query Processing](http://www.interdb.jp/pg/pgsql03.html)
  - [Chapter 4. Foreign Data Wrappers (FDW)](http://www.interdb.jp/pg/pgsql04.html)
  - [Chapter 5. Concurrency Control](http://www.interdb.jp/pg/pgsql05.html)
  - [Chapter 6. VACUUM Processing](http://www.interdb.jp/pg/pgsql06.html)
  - [Chapter 7. Heap Only Tuple (HOT) and Index-Only Scans](http://www.interdb.jp/pg/pgsql07.html)
  - [Chapter 8. Buffer Manager](http://www.interdb.jp/pg/pgsql08.html)
  - [Chapter 9. Write Ahead Logging (WAL)](http://www.interdb.jp/pg/pgsql09.html)
  - [Chapter 10. Base Backup and Point-In-Time Recovery (PITR)](https://www.interdb.jp/pg/pgsql10.html)
  - [Chapter 11. Streaming Replication](https://www.interdb.jp/pg/pgsql11.html)

<h4>Papers</h4>

- [Enhancement of the ANSI SQL Implementation of PostgreSQL](https://www.ic.unicamp.br/~celio/livrobd/postgres/ansi_sql_implementation_postgresql.pdf) by Stefan Simkovics

<h4>Transaction</h4>

- [Serializable Snapshot Isolation (SSI) in PostgreSQL](https://wiki.postgresql.org/wiki/SSI)
- [Information about the SSI implementation for the SERIALIZABLE transaction isolation level in PostgreSQL](https://wiki.postgresql.org/wiki/Serializable)

<h4>Tutorials</h4>

- [2-Day Introduction to PostgreSQL 11](https://postgrespro.com/education/courses/2dINTRO)

<h4>Videos</h4>

- [PostgreSQL Internals Through Pictures](https://www.youtube.com/watch?v=JFh22atXTRQ) by Bruce Momjian, 2015
- [Inside PostgreSQL Shared Memory](https://www.youtube.com/watch?v=BNDjonm7s7I) by Bruce Momjian, 2015
- [How does PostgreSQL actully work](https://www.youtube.com/watch?v=OeKbL55OyL0) by Peter Eisentraut, 2016
- [PostgreSQL Top Ten Features](https://www.youtube.com/watch?v=KK0YPraAYTo) by Simon Riggs, 2014
- [Understanding Autovacuum](https://www.youtube.com/watch?v=GqrBp0gyNHs) by Dan Robinson, 2016
- [Hacking on PostgreSQL](https://www.youtube.com/watch?v=-OZEPExwZEw) by Stephen Frost, 2013
- [Debugging your PL/pgSQL code](https://www.youtube.com/watch?v=pOb-7JZQoW4) by Jim Mlodgenski, 2016
- [Zen of Postgres: How to be a happy hacker](https://www.youtube.com/watch?v=yFDyM29tB6k) by Andrew Dunstan, 2012
- [MVCC Unmasked](https://www.youtube.com/watch?v=sq_aO34SWZc) by Bruce Momjian, OSCon 2011
- [Understanding Advanced Datatypes in PostgreSQL](https://www.youtube.com/watch?v=wzKWMF-kWGc)
- [Unlocking the Postgres Lock Manager](https://www.youtube.com/watch?v=s3ee0nuDDqs) / [Slides](https://momjian.us/main/writings/pgsql/locking.pdf) by Bruce Momjian, 2012
- [Index Internals](https://www.youtube.com/watch?v=W6B8-srOsrs) by Heikki Linnakangas, 2016
- [PostgreSQL Partitioning](https://www.youtube.com/watch?v=yp_8QSWOcTI) by Robert Treat, PGCon 2018
- [Learning to Hack on Postgres Planner](https://www.youtube.com/watch?v=j7UPVU5UCV4) by Melanie Plageman, PGCon 2019
- [An introduction to Memory Contexts](https://www.youtube.com/watch?v=tP2pHbKz2R0) by Chris Travers, PGCon 2019
- [Hacking the Query Planner, Again](https://www.youtube.com/watch?v=wTg02tniO2A) by Richard Guo, PGCon 2020
- [PostgreSQL 空闲空间管理](https://www.bilibili.com/video/BV1934y1i7pk) by Wang Xiaoran, PGConf.Asia 2021
- [Why Uber Engineering Switched from Postgres to MySQL](https://www.youtube.com/watch?v=_E43l5EbNI4) by Hussein Nasser, 2020
- [A PostgreSQL Response to Uber](https://www.youtube.com/watch?v=5dIbB5GIqAo) [slides](https://thebuild.com/presentations/uber-perconalive-2017.pdf) by Christophe Pettus, 2017

<h4>Greenplum specific</h4>

- [Greenplum性能优化之路 --（一）分区表](https://cloud.tencent.com/developer/article/1374067)
- [Greenplum性能优化之路 --（二）存储格式](https://cloud.tencent.com/developer/article/1393372)
- [Greenplum性能优化之路 --（三）ANALYZE](https://cloud.tencent.com/developer/article/1685910)
- [Greenplum 分布式数据库内核揭秘(上篇)](https://cn.greenplum.org/greenplum-distributed-database-kernel-1/)
- [Greenplum 分布式数据库内核揭秘(下篇)](https://cn.greenplum.org/greenplum-distributed-database-kernel-2/)
- [Greenplum Storage Considerations](https://www.youtube.com/watch?v=6xRiERh-x24)
- [新一代数据分析及实时数仓平台 Greenplum 介绍](https://www.bilibili.com/video/BV1Zr4y127Zk)
- [Greenplum —— 一个用于分析、机器学习和AI的开源大规模并行数据库平台](https://www.bilibili.com/video/BV1v5411u7VE)
- [Brin Index 在 Append Only Table 中的实现](https://www.bilibili.com/video/BV12V411a7A5)
- [Brin Index 在 Greenplum 7 中的理论与实践](https://www.bilibili.com/video/BV1YU4y187ia)
- [当谈起 Greenplum 7 时，我们在谈什么？](https://www.bilibili.com/video/BV1WV411n7Zh)
- [Greenplum 中的多阶段聚集实现](https://www.bilibili.com/video/BV1ey4y1H7eb)
- [Greenplum 的分布式锁及其相关的一切](https://www.bilibili.com/video/BV1vZ4y1G7fy)
- [《深入浅出 Greenplum 内核》系列视频](https://space.bilibili.com/489184136/channel/seriesdetail?sid=888852)
  - [新一代大数据分析平台 Greenplum 架构解读](https://www.bilibili.com/video/BV1Sf4y1U7QP)
  - [开源分布式数据库 Greenplum 并行执行引擎揭秘](https://www.bilibili.com/video/BV1Si4y1474L)
  - [开源大数据分析平台 Greenplum 查询优化揭秘](https://www.bilibili.com/video/BV1J5411Y7yu)
  - [Greenplum 内核揭秘之B树索引](https://www.bilibili.com/video/BV1164y1F7XP)
  - [Greenplum 内核揭秘之 MVCC 并发控制](https://www.bilibili.com/video/BV1yT4y1w7Fn)
  - [Greenplum 内核揭秘之排序算法](https://www.bilibili.com/video/BV17f4y1D76h)
  - [Greenplum 分布式事务和两阶段提交协议](https://www.bilibili.com/video/BV1et4y1e7RF)
  - [揭秘 Greenplum 存储引擎之Heap表](https://www.bilibili.com/video/BV1fK4y1j7jJ)
  - [Greenplum 高可用理论与实践](https://www.bilibili.com/video/BV1Sz4y167cP)
  - [揭秘 Greenplum 恢复（Recovery）系统](https://www.bilibili.com/video/BV1Ft4y1B74e)
- [Parallel Postgres: What's Coming in Greenplum 7](https://www.youtube.com/watch?v=-8RbbETBs4E)
- [Greenplum 分布式数据库内核揭秘](https://www.bilibili.com/video/BV1sP4y1T7wo)
- [Greenplum 中的资源管理策略](https://www.bilibili.com/video/BV1uZ4y167Sh)

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
[1] [Awesome Postgres](https://github.com/dhamaniasad/awesome-postgres)<br>
</span>

[postgres_up_and_running_3rd]: https://www.oreilly.com/library/view/postgresql-up-and/9781491963401/
[postgres_internals]: http://www.interdb.jp/pg/
[mastering_pg_11]: https://www.amazon.com/Mastering-PostgreSQL-techniques-fault-tolerant-applications/dp/1789537819
