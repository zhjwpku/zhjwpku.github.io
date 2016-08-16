---
layout: post
title: git bisect
date:   2016-02-18 17:50:00 +0800
categories: others
tags:
- git
---

今天在看openssl源码的时候，发现最新pull下来的代码把ssl/ssl_algs.c这个文件删除了，该文件有函数
`SSL_library_init`的定义，于是我想知道为什么删除该文件，刚开始用笨办法——倒着查看每个patch，
但太慢了，于是想到了`git bisect`，虽然之前知道这个工具，但一直没用过，现在终于有用武之地了。

{% highlight ruby %}
//开始二分查找
$git bisect start  

//找了一个包含ssl/ssl_algs.c文件的tag——`OpenSSL_1_1_0-pre2`，`bd31d02`是它commit号的前几位(只要不冲突就行)。
$git bisect good bd31d02

//接下来会自动切换到中间的一个状态，如果包含所需要的文件，就输入：
$git bisect good
 
//否则输入：
$git bisect bad
 
//如此重复直到定位到是在哪个commit将文件删除的，记下commit号，然后回复到代码库原始的状态：
$git bisect reset

//查看patch的详细信息
$git show `commit-num`
{% endhighlight %}
