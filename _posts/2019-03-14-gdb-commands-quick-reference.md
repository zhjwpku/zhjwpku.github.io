---
layout: post
title: "GDB 命令快速参考"
date: 2019-03-14 22:00:00 +0800
tags:
- "C/C++"
---

本文是 [Summary of GDB commands for x86-64 Systems][gdbnotes-x86-64] 的人肉翻译 📝。并在 *Reference* 处列出了一些 gdb 的在线文档。

**启动**

```
gdb
gdb <file>
```

**运行 & 停止**

```
run                                 运行程序
run 1 2 3                           以 1 2 3 作为参数运行程序
kill                                杀掉当前运行程序
quit/q                              停止 gdb
Ctrl-d                              停止 gdb
```

*注：Ctrl-C 不会从 gdb 退出，但是会停止当前运行的 gdb 命令*

**断点**

```
break sum                           在函数 sum 入口处设置断点
break *0x80483c3                    在地址 0x80483c3 处设置断点
delete 1                            删除断点 1
disable 1                           关闭断点 1 (gdb 会为你创建的每个断点编号)
enable 1                            打开断点 1
delete                              删除所有断点
clear sum                           删除在函数 sum 入口处所有的断点
```

**执行**

```
stepi/si                            执行一条指令
stepi 4                             执行四条指令
nexti                               跟 stepi 相似，但是执行函数时不会停止
step/s                              执行一条 C 语句                    // step into
next/n                              执行一条 C 语句，遇到函数不会进入     // step over
finish/fin                          继续执行直到当前栈返回的下一条语句    // step out
continue                            恢复程序执行直到下一个断点
until 3                             继续执行程序直到断点3
finish                              恢复执行直到当前函数返回
call sum(1, 2)                      调用 sum(1, 2) 并打印返回值
```

**查看代码**

```
disas                               反汇编当前函数
disas sum                           反汇编函数 sum
disas 0x80483b7                     反汇编地址 0x80483b7 附近的函数
disas 0x80483b7, 0x80483c7           反汇编指定地址范围内的代码

print /x $rip                       以16进制打印程序计数器 (PC)
print /d $rip                       以10进制打印 PC
print /t $rip                       以2进制打印 PC
```

**查看数据**

```
print /d $rax                       以10进制打印 $rax 中的内容
print /x $rax                       以16进制打印 $rax 中的内容
print /t $rax                       以2进制打印 $rax 中的内容
print /d (int)$rax                  以10进制打印 $rax 低32位中的内容，例如当低32位存储的是 0xffffffff，你会看到
                                    (gdb) print $rax
                                    $1 = 4294967295
                                    (gdb) print (int)$rax
                                    $2 = -1

print 0x100                         打印 0x100 的十进制表示
print /x 555                        打印 555 的十六进制表示
print /x ($rsp+8)                   打印 $rsp 寄存器中值 + 8 的16进制
print *(int *) 0xbffff890           打印地址 0xbffff890 处的 int 值
print *(int *) ($rsp+8)             打印地址 $rsp + 8 地址处的 int 值
print (char *) 0xbffff890           打印 0xbffff890 处的字符串

x/w 0xbffff890                      查看地址 0xbffff890 起始的字(4字节)
x/w $rsp                            查看 $rsp 中地址起始的字
x/wd $rsp                           查看 $rsp 中地址起始的字，以十进制打印
x/2w $rsp                           查看 $rsp 中地址起始的两个字
x/2wd $rsp                          查看 $rsp 中地址起始的两个字，并以十进制打印
x/g $rsp                            查看 $rsp 中地址起始的 8 字节长度内容
x/gd $rsp                           查看 $rsp 中地址起始的 8 字节长度内容，以十进制打印
x/a $rsp                            查看 $rsp 中的地址。Print as offset from previous global symbol.
x/s 0xbffff890                      查看地址 0xbffff890 处的字符串
x/20b sum                           查看函数 sum 的前 20 个操作码字节
x/10i                               查看函数 sum 的前 10 个指令
```

*注：x 命令的通用格式为 x/[NUM][SIZE][FORMAT]，其中*
*NUM  = 要展示对象的个数*
*SIZE = 每个对象的大小（b=byte, h=half-word, w=word, g=giant/quad-word）*
*FORMAT = 如何展示每个对象（d=decimal, x=hex, o=octal）*
*如果不指定 SIZE 或 FORMAT，将使用默认值或最近使用的 print 或 x 命令的值*

**其它有用的信息**

```
backtrace                           打印当前地址和函数调用栈
where                               打印当前地址和函数调用栈

info program                        打印当前程序的状态
info functions                      打印程序中的函数
info stack                          打印函数调用栈
info frame                          打印当前栈帧的信息
info registers                      打印寄存器及其内容
info all-registers                  get all of the registers on the particular CPU you are using
info breakpoints                    打印用户可设置的断点的状态

display /FMT EXPR                   每次 GDB 停止的时候，以 FMT 格式打印 EXPR
undisplay                           关闭 display 模式

help                                获取 gdb 的帮助信息
```

**batch 模式**

当不想进入交互模式去执行 GDB 命令并在执行完后退出时，比如打印进程全局变量，可以使用 batch 模式。

```
gdb -batch -ex "set pagination 0" -ex "p *WalRcv" -p 11551
gdb -batch -ex "set pagination 0" -ex "set variable WalRcv->reserveLSN = 939524096" -p 39395
```

**trace into child process when fork**

```
set follow-fork-mode child
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
0 A [backup](/assets/pdf/gdbnotes-x86-64.pdf) of Summary of GDB commands for x86-64 Systems.<br>
1 [Guide to Faster, Less Frustrating Debugging](http://heather.cs.ucdavis.edu/~matloff/UnixAndC/CLanguage/Debug.html) <br>
2 [GDB Documentation](https://www.gnu.org/software/gdb/documentation/)<br>
3 [Debugging with GDB](https://sourceware.org/gdb/current/onlinedocs/gdb/index.html) and its [PDF version](/assets/pdf/debugging_with_gdb.pdf)<br>
4 [gdb Cheatsheet](http://cs.brown.edu/courses/csci0330/docs/guides/gdb.pdf) from Brown University's [CS0330](http://cs.brown.edu/courses/csci0330/)<br>
5 [x64 Cheat Sheet](http://cs.brown.edu/courses/csci0330/docs/guides/x64_cheatsheet.pdf) from Brown University's CS0330 <br>
6 [5.2 Continuing and Stepping](https://sourceware.org/gdb/onlinedocs/gdb/Continuing-and-Stepping.html)<br>
7 [The Art of Debugging with GDB, DDD, and Eclipse](/assets/pdf/books/The.Art.of.Debugging.with.GDB.DDD.and.Eclipse.pdf)<br>
8 [GDB Tutorial: Some Cool Tips to Debug C/C++ Code](https://www.techbeamers.com/how-to-use-gdb-top-debugging-tips/)<br>
9 [CppCon 2015: Greg Law "Give me 15 minutes & I'll change your view of GDB"](https://www.youtube.com/watch?v=PorfLSr3DDI)<br>
10 [CppCon 2016: Greg Law "GDB - A Lot More Than You Knew"](https://www.youtube.com/watch?v=-n9Fkq1e6sg)<br>
11 [CppCon 2018: Simon Brand "How C++ Debuggers Work"](https://www.youtube.com/watch?v=0DDrseUomfU)<br>
12 [GDB to LLDB command map](https://lldb.llvm.org/use/map.html)<br>
13 [Debugging C/C++ with LLDB Tutorial](https://www.youtube.com/watch?v=2GV0K9Y2MKA)<br>
</span>

[gdbnotes-x86-64]: http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf
