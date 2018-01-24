---
layout: post
title: Java 程序分析工具
date: 2018-01-23 22:00:00 +0800
tags:
- java
---

由于运行了一堆 Java 微服务，导致开发环境越来越慢，在扩展了一次内存之后，问题依然没有解决。`free -h` 和 `vmstate -s` 可以用来查看操作系统内存使用情况，但对于 Java 程序，`jps`、`jmap`、`jinfo`、`jstat` 等工具能让你进行更细粒度的分析。

**jps**

该命令用于显示运行的 JVM 的入口函数：

```
[root@dev-appserver ~]# jps -l
5799 com.star.epg.EpgApplication
7880 com.star.strategy.DvbStrategyApplication
6540 com.star.op.OperationsApplication
2893 com.star.smssender.SMSServiceApplication
9773 com.star.payment.PaymentApplication
10190 sun.tools.jps.Jps
7727 com.star.cms.start.Start
9615 com.star.pup.PupApplication
6672 com.star.bms.BmsServiceApplication
9201 com.star.mobilewallet.MobileWalletApplication
9012 com.star.order.OrderApplication
10132 com.star.channel.ChannelApplication
2710 com.star.css.CssApplication
9496 com.star.boss.proxy.Main
2462 org.apache.catalina.startup.Bootstrap
```

**jmap**

该命令用于显示共享对象内存映射或堆(heap)详细信息:

```
[root@dev-appserver ~]# jmap -heap 5799
Attaching to process ID 5799, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.131-b11

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2051014656 (1956.0MB)
   NewSize                  = 42991616 (41.0MB)
   MaxNewSize               = 683671552 (652.0MB)
   OldSize                  = 87031808 (83.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 634388480 (605.0MB)
   used     = 129431144 (123.4351577758789MB)
   free     = 504957336 (481.5648422241211MB)
   20.402505417500645% used
From Space:
   capacity = 25165824 (24.0MB)
   used     = 11320032 (10.795623779296875MB)
   free     = 13845792 (13.204376220703125MB)
   44.98176574707031% used
To Space:
   capacity = 24117248 (23.0MB)
   used     = 0 (0.0MB)
   free     = 24117248 (23.0MB)
   0.0% used
PS Old Generation
   capacity = 87031808 (83.0MB)
   used     = 38775616 (36.97930908203125MB)
   free     = 48256192 (46.02069091796875MB)
   44.55338443618223% used

30769 interned Strings occupying 3598960 bytes.

[root@dev-appserver ~]# jmap -histo 5799
 num     #instances         #bytes  class name
----------------------------------------------
   1:        387415       55601848  [C
   2:         51496       31275592  [I
   3:        117228       23027728  [B
   4:        186882        7475280  java.util.HashMap$KeyIterator
   5:        291963        7007112  java.lang.String
   6:        165687        5301984  java.util.HashMap$Node
   7:         75385        4221560  java.util.concurrent.ConcurrentHashMap$KeyIterator
   8:         35510        3124880  java.lang.reflect.Method
   9:         49229        3112688  [Ljava.util.HashMap$Node;
  10:        119850        2934864  [Z
  11:         64694        2869648  [Ljava.lang.Object;
  12:        118929        2854296  java.text.ParsePosition
  13:        108477        2603448  java.lang.Long
  14:         44959        2158032  java.util.HashMap
  15:         33602        1612896  java.nio.HeapByteBuffer
  16:         38012        1520480  java.util.LinkedHashMap$Entry
  17:         37508        1500320  java.util.TreeMap$Entry
  18:         13291        1479224  java.lang.Class
  19:         42219        1351008  java.util.concurrent.ConcurrentHashMap$Node
  20:         46482        1115568  java.util.concurrent.ConcurrentHashMap$MapEntry
  21:         30908         989056  java.util.concurrent.locks.AbstractQueuedSynchronizer$Node
  22:         46273         944512  [Ljava.lang.Class;
  23:         15408         862848  java.util.concurrent.ConcurrentHashMap$EntryIterator
  24:         15108         846048  java.util.LinkedHashMap
  25:         32349         776376  java.util.ArrayList
  26:         15579         747792  java.nio.HeapCharBuffer
  27:         18389         735560  java.util.HashMap$EntryIterator
  28:         22068         710928  [Ljava.lang.String;
  29:         20307         649824  java.lang.ref.WeakReference
  30:         15543         621720  java.util.HashMap$ValueIterator
```

**jinfo**

该命令用于 JVM 运行参数：

```
[root@dev-appserver ~]# jinfo -flags 6540
Attaching to process ID 6540, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.131-b11
Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=1073741824 -XX:MaxNewSize=357564416 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Xms256m -Xmx1024m
```

**jstat**

该命令用于监控 JVM 统计信息：

```
[root@dev-appserver ~]# jstat -gcutil 6540 1000
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 46.49   0.00  52.64  46.14  95.90  93.65    120   64.400     3    0.281   64.681
 46.49   0.00  52.64  46.14  95.90  93.65    120   64.400     3    0.281   64.681
 46.49   0.00  52.64  46.14  95.90  93.65    120   64.400     3    0.281   64.681
```

*未完待补充*

