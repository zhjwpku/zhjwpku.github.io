---
layout: post
title: 在 Mac 上编译安装并运行 PostgreSQL
date: 2022-07-02 00:00:00 +0800
tags:
- postgres
---

记录在 MacOS 上编译、安装、运行、调试 PostgreSQL 的步骤。

**下载代码**

```shell
git clone https://github.com/postgres/postgres.git
cd postgres
```

**编译安装**

在 mac 上通常是要定位一些代码问题，因此把 debug 打开，默认将程序安装到 /usr/local/pgsql，可通过 `./configure --prefix=<path>` 修改安装路径。

```shell
CFLAGS="-Og -fno-inline-functions -fno-omit-frame-pointer" ./configure --enable-debug --without-icu --enable-tap-tests
make -j8
sudo make install
```

**初始化数据库并运行**

1. 创建一个当前用户拥有的目录，并在该目录初始化数据库
```shell
mkdir -p ~/pgdata
/usr/local/pgsql/bin/initdb -D ~/pgdata
```

2. 启动数据库
```shell
/usr/local/pgsql/bin/pg_ctl -D ~/pgdata -l ~/pgdata/logfile start
```

3. 停止数据库
```shell
/usr/local/pgsql/bin/pg_ctl -D ~/pgdata stop
```

**调试**

1. 调试 postmaster
```shell
lldb -p `head -n1 ~/pgdata/postmaster.pid`
```

2. 调试 postgres
```shell
lldb -p <postgres_pid>
```

3. 忽略 SIGUSR1 信号的处理:
```shell
(lldb) pro hand -p true -s false SIGUSR1
```

**building postgres with meson**

社区在 2022/9/22 合入了 `meson based build system` 用于替代 Autoconf，详见 ref [2]。

```
mkdir -p build && cd build
meson setup -Ddebug=true . ..
meson compile
sudo meson install
```

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [GDB to LLDB command map](https://lldb.llvm.org/use/map.html)<br>
2 [building postgres with meson - v13](https://www.postgresql.org/message-id/20220915022626.5xx3ccgkzpkqw5mq@awork3.anarazel.de)<br>
3 [Postgres Meson Wiki](https://wiki.postgresql.org/wiki/Meson)<br>
4 [GCC command options for debugging – -Og and -g3](http://kristerw.blogspot.com/2018/02/gcc-command-options-for-debugging-og.html)<br>
5 [Unlock Your Arsenals: GDB Debugging Essentials with PostgreSQL](https://www.techbuddies.io/2023/10/30/unlock-your-arsenals-gdb-debugging-essentials-with-postgresql/)
</span>

