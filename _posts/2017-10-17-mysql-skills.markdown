---
layout: post
title: MySQL实用技巧
date: 2017-10-17 18:30:00 +0800
tags:
- mysql
---

笔者对 MySQL 从来心存敬畏，一直期望成为一名懂一丢丢 DBA 的研发😏，在此记录开发中与 MySQL 相关的点滴。

<h4>字段中带空格的值的查询方法</h4>

有这样一组数据（通过 `select * from user where nickName like "%marz%";` 获取）:

| id | age | sex | nickName
|:-|:-|:-|:-
| 1 | 20 | 0 | marzxwell
| 2 | 30 | 0 | Marzuki Manuel
| 3 | 40 | 1 | Marz Kimz
| 4 | 50 | 1 | Marzuqah Amuni

<br>
现在有一个需求，当查询条件为 `MarzukiManuel` 或 `Marzuki Manuel` 都要求把第二条数据查出。由于内容的不确定，需要一种通用的查询方式：

```
select * from user where trim(replace(nickName,' ','')) like trim(replace('%Marzuki Manuel%',' ',''));
select * from user where trim(replace(nickName,' ','')) like trim(replace('%MarzukiManuel%',' ',''));
```

如此不论传进来的值中是否带有空格，都能够获取所需的结果。


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [mysql字段中带空格的值的查询方法](http://www.liyangweb.com/mysql/142.html)
</span>
