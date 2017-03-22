---
layout: post
title: 调整 Java Heap Memory 大小
date: 2017-03-22 11:00:00 +0800
tags:
- java
---

安装 [ZooKeeper][zookeeper] 时有一步要求调整 Java Heap 大小，比如 4GB 内存的机器应该把最大 heap size 调整为 3GB, 从而避免swapping（swapping 会严重影响 ZooKeeper 性能）。

**查看当前环境最大 Heap Size**

{% highlight shell %}
[vagrant@zookeeper1 ~]$ java -XX:+PrintFlagsFinal -version | grep -iE 'Heapsize|permsize|threadStackSize'
Picked up _JAVA_OPTIONS: -Xmx3072m
     intx CompilerThreadStackSize               = 0                             {pd product}
    uintx ErgoHeapSizeLimit                     = 0                             {product}
    uintx HeapSizePerGCThread                   = 87241520                      {product}
    uintx InitialHeapSize                      := 62914560                      {product}
    uintx LargePageHeapSizeThreshold            = 134217728                     {product}
    uintx MaxHeapSize                          := 3221225472                    {product}
     intx ThreadStackSize                       = 1024                          {pd product}
     intx VMThreadStackSize                     = 1024                          {pd product}
openjdk version "1.8.0_121"
OpenJDK Runtime Environment (build 1.8.0_121-b13)
OpenJDK 64-Bit Server VM (build 25.121-b13, mixed mode)
{% endhighlight %}

InitialHeapSize = 62914560 Byte = 60M
MaxHeapSize = 3221225472 Byte = 3GB

**运行时指定**

{% highlight shell %}
# 指定最大HeapSize为6G，使用64-bit JVM
$ java -Xmx6144M -d64 ...
{% endhighlight %}

**设置环境变量**

{% highlight shell %}
# 指定InitialHeapSize为512M，MaxHeapSize为3072M
$ export _JAVA_OPTIONS="-Xms512m -Xmx3072m"
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Increase heap size in java][ref1]<br>
2 [how to increase java heap memory permanently?][ref2]<br>
3 [Find out your Java heap memory size][ref3]
</span>

[zookeeper]: http://zookeeper.apache.org/doc/trunk/zookeeperAdmin.html
[ref1]: http://stackoverflow.com/questions/1565388/increase-heap-size-in-java
[ref2]: http://stackoverflow.com/questions/11578123/how-to-increase-java-heap-memory-permanently 
[ref3]: https://www.mkyong.com/java/find-out-your-java-heap-memory-size
