---
layout: post
title: "Core Dumps"
date: 2019-03-21 20:00:00 +0800
tags:
- "C/C++"
---

操作系统可以配置为当某个程序因进程终止信号（如 segmentation fault）而崩溃时，生成 core dump 文件。这个核心转储文件包含进程终止时的内存快照。可以使用 gdb 加载该文件，查看程序崩溃时的状态，这在调试中非常有用。

默认情况下，Linux 操作系统上的程序在崩溃时不会产生 core dump 文件。将 *~/.bashrc* 文件中的 `ulimit -c 0` 改为 `ulimit -c unlimited`，并运行 `source ~/.bashrc` 来使其立即生效，可以保证当程序崩溃时会产生 core dump 文件。

**core dump 文件写在哪?**

```
sysctl kernel.core_pattern  # 查看 core pattern
sysctl -w kernel.core_pattern=/tmp/core-%e.%p.%h.%t # 更改 core pattern
```

**调试 core dump**

通过如下命令即可查看程序崩溃时的各个栈帧的寄存器状态。

```
gdb <executable file name> <core dump file name>
```

**bt 看不见栈**

有时候在加载了可执行文件和core文件之后，bt 看到的栈信息可能是一堆问号，很有可能是二进制依赖的动态链接库状态不对，可以使用 `info sharedlibrary` 看需的依赖库，然后把所有依赖库按照原来的路径结构放到一个目录下，比如 */fakeroot*，然后在 gdb 中用 `set sysroot /fakeroot`，之后再加载一下 core 文件（`core-file core-name`），就可以看到栈信息了。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [C Programming Tools](http://cs.brown.edu/courses/csci0330/docs/guides/tools.pdf) from Brown University's [CS0330](http://cs.brown.edu/courses/csci0330/) <br>
2 [How to get a core dump for a segfault on Linux](https://jvns.ca/blog/2018/04/28/debugging-a-segfault-on-linux/) <br>
</span>

