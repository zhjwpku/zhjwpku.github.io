---
layout: post
title: Gprof 实践
date: 2020-07-07 22:00:00 +0800
tags:
- gprof
- profiling
---

[Profiling][profiling] 是一种可以有效度量软件程序性能的方法，常用的技术包括 event-based, statistical, instrumented, and simulation methods 等。Linux 上常用的 Profiling 工具有 [Gprof][gprof], [perf][perf], [Valgrind][valgrind], [OProfile][oprofile], Google 的 [gperftools][gperftools] 等，这其中我用过 Gprof 和 Valgrind，但我后面会更关注 perf，本文简单介绍 Gprof 的使用方法。

Gprof 的使用大致可以分为三步，我们以一个 JPEG 图片压缩库 [Brunsli][brunsli] 为例来看下 Gprof 的使用方法。

实验在一个 `v.memory = 1024; v.cpus = 2` 的虚拟机上进行:

```
vagrant@ubuntu-xenial:~$ git clone --depth=1 https://github.com/google/brunsli.git
vagrant@ubuntu-xenial:~$ cd brunsli
vagrant@ubuntu-xenial:~/brunsli$ git submodule update --init --recursive
```

**1. 添加编译/链接参数 `-pg`**

编译添加 `-pg` 是为了在对象文件中生成额外的 profiling 信息；而链接添加 `-pg` 是为了链接 gcrt0.o 而非 crt0.o，从而能在 main 函数之前调用 profiling 相关的初始化函数。我们在修改 brunsli 的 Makefile:

```
CFLAGS += -O2 -std=c++11 -ffunction-sections -pg
LDFLAGS += -Wl,-gc-sections -pg
```

编译:
```
vagrant@ubuntu-xenial:~/brunsli$ make
```

**2. 运行程序**

由于第 1 步已经将 profiling 需要的信息写在了可执行文件中，因此该步骤跟正常执行一个程序完全一样:

```
vagrant@ubuntu-xenial:~/brunsli$ bin/cbrunsli docs/WIEGO_ACCRA_8008_FULLY_RELEASED-scaled.p.jpg
```

执行完之后会生成一个 gmon.out 的文件，这里边包含了后面用 `gprof` 进行查看分析的所有 profiling 信息。

**3. 使用 gprof 进行分析**

gprof 展示的结果分两部分: *[The Flat Profile][flat]* 和 *[The Call Graph][graph]*，具体说明见手册。

```
vagrant@ubuntu-xenial:~/brunsli$ gprof bin/cbrunsli
```

**Bonus**

使用 [gprof2dot][gprof2dot] 可以将 gprof 的 Call Graph 生成直观的图片:

```
vagrant@ubuntu-xenial:~/brunsli$ pip install gprof2dot
vagrant@ubuntu-xenial:~/brunsli$ sudo apt-get install graphviz
vagrant@ubuntu-xenial:~/brunsli$ gprof bin/cbrunsli | gprof2dot | dot -Tpng -o output.png
```

**Pros&Cons**

Pros:

- GCC 原生支持，不需要额外安装其它库，简单易用
- call graph 能清晰展现出程序的调用关系

Cons:

- 修改代码需要重新编译，对于大型项目来说意味着很耗时间
- 采样率 100ps，略低
- 由于往 gmon.out 写 profiling 数据是通过 atexit() 调用了 mcleanup，因此在程序异常退出的情况下得不到 profiling 数据

*注: 后面笔者会重点关注 perf*

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [CPU Profiling Tools on Linux](http://euccas.github.io/blog/20170827/cpu-profiling-tools-on-linux.html)<br>
2 [GNU gprof](https://sourceware.org/binutils/docs/gprof/)<br>
3 [GCC Program Instrumentation Options](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html#Instrumentation-Options)<br>
</span>

[gprof]: https://en.wikipedia.org/wiki/Gprof
[profiling]: https://en.wikipedia.org/wiki/Profiling_(computer_programming)
[valgrind]: https://valgrind.org/
[perf]: https://perf.wiki.kernel.org/index.php/Main_Page
[oprofile]: https://en.wikipedia.org/wiki/OProfile
[gperftools]: https://github.com/gperftools/gperftools
[brunsli]: https://github.com/google/brunsli
[flat]: https://sourceware.org/binutils/docs/gprof/Flat-Profile.html
[graph]: https://sourceware.org/binutils/docs/gprof/Call-Graph.html
[gprof2dot]: https://github.com/jrfonseca/gprof2dot