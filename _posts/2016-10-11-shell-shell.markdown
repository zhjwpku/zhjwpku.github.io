---
layout: post
title: Shell Scripts
date: 2016-10-11 18:00:00 +0800
tags:
- shell
---

哇！发现了好多不懂的Shell命令，记录一下...

**awk**

```
# 以 [ 或 ] 作为分隔符
awk -F'[][]'        # '[[]]' 不行，因为会解析成 Bracket Expression

# 统计某一字段出现的频率(后面一种方法效率更高)
awk -F'[][]' '{ if ($10 != "") print $10}' file_to_be_handled.txt | sort | uniq -c | sort -nr
awk -F'[][]' '{ if ($10 != "") tot[$10]++} END { for (i in tot) print tot[i], i }' file_to_be_handled.txt | sort -nr

# 如果使用内置的 sorting, 效率可能会更高
# 参考 https://www.gnu.org/software/gawk/manual/html_node/Controlling-Scanning.html
```

**. command**

{% highlight shell %}
#!/bin/bash

# 重启一个子shell，在子shell中执行脚本，执行完毕后退出并回到父shell
./another_shell.sh

# 在当前shell进程中执行脚本，两者使用同一个上下文
. ./another_shell.sh

# 同. ./another_shell.sh
source ./another_shell.sh
{% endhighlight %}

**expr**

```shell
#!/bin/bash

# 将 1 + 2 的算数值赋给 a
a=$(expr 1 + 2)
echo $a         # 输出 3
b=$(expr $a - 1)
echo $b         # 输出 2
c=$(expr $a \* $b)
echo $c         # 输出 6
d=$(expr $c / $a)
echo $d         # 输出 2
```

**set -e & set -o**

{% highlight shell %}
#!/bin/bash

# 一旦脚本中有命令的返回值为非0，脚本立即退出
set -e

# Same as -e
set -o errexit

# Same as -E. If set, any trap on ERR is inherited by shell functions, command substitutions, and commands executed in a subshell environment.
set -o errtrace

# 设置了set -o pipefail，返回从右往左第一个非零返回值，即ls的返回值1
set -o pipefail
ls ./a.txt |echo "hi" >/dev/null
echo $?
{% endhighlight %}

**set -x & set +x**

```shell
#!/bin/bash

# 执行指令后，会先显示该指令及所下的参数
set -x

# 取消显示
set +x
```

**command -v**

{% highlight shell %}
#!/bin/bash

# 是否存在http命令
if ! command -v http &>/dev/null ; then
  echo "httpie not installed, 'apt-get install httpie', 'yum install httpie' or 'brew install httpie'"
  exit 1
fi
{% endhighlight %}

**hexdump/hd**

```
root@531f5d0c9f9f:/opt/csapp3e# hexdump -x README.md
0000000    6854    7369    6420    7269    6320    6e6f    6174    6e69
0000010    2073    796d    7320    6c6f    7475    6f69    736e    7420
0000020    206f    6874    2065    616c    7362    000a
000002b
root@531f5d0c9f9f:/opt/csapp3e# hd -x README.md
00000000  54 68 69 73 20 64 69 72  20 63 6f 6e 74 61 69 6e  |This dir contain|
0000000    6854    7369    6420    7269    6320    6e6f    6174    6e69
00000010  73 20 6d 79 20 73 6f 6c  75 74 69 6f 6e 73 20 74  |s my solutions t|
0000010    2073    796d    7320    6c6f    7475    6f69    736e    7420
00000020  6f 20 74 68 65 20 6c 61  62 73 0a                 |o the labs.|
0000020    206f    6874    2065    616c    7362    000a
000002b
```

**将目录下的某种类型的所有文件以空格分开输出**

```
# 目录类型以 / 结尾
$ ls -F | grep -E '/$' | xargs
# 软链接类型以 @ 结尾
$ ls -F | grep -E '@$' | xargs
# 文件类型
$ ls -F | grep -E '[^/@]$' | xargs

## 其它详见 man ls
```

**查看ASCII码**

```
# 不用再去网上搜了，随便找台Linux就可以看了
man ascii
```

**查看 shell 的条件语句写法**

```
man test
```

**查看 errno**

```
# not work on Mac
man errno
```

**查看块设备 UUID**

```
blkid
```

**查看网络端口**

```
netstat -tlnp
```

**特殊变量**

{% highlight shell %}
#!/bin/bash

# 当前进程的pid
echo $$

# Shell最后运行的后台进程PID
echo $!

# 当前脚本名
echo $0

# 传递给脚本第n个参数，n为一个具体的数字
echo $n

# 传递给脚本的参数个数
echo $#

# 传递给脚本的所有参数，被双引号包含时，会把所有参数作为一个整体
echo $*

# 传递给脚本的所有参数，不管是否被双引号包含，都以分开的形式传递
echo $@

# 上个命令的退出状态
echo $?

{% endhighlight %}

**${} / ## / %%**

```shell
#!/bin/bash

file = /path/to/my.file.txt

# 删掉第一个'/'及其左边的字符串，结果为`path/to/my.file.txt`
echo ${file#*/}

# 删掉最后一个'/'及其左边的字符串，结果为`my.file.txt`
echo ${file##*/}

# 删掉第一个'.'及其左边的字符串，结果为`file.txt`
echo ${file#*.}

# 删掉最后一个'.'及其左边的字符串，结果为`txt`
echo ${file##*.}

# 删掉最后一个'/'及其右边的字符串，结果为`path/to`
echo ${file%/*}

# 删掉第一个'/'及其右边的字符串，结果为空
echo ${file%%/*}

# 删掉最后一个'.'及其右边的字符串，结果为`/path/to/my.file`
echo ${file%.*}

# 删掉第一个'.'及其右边的字符串，结果为`/path/to/my`
echo ${file%%.*}

# 将第一个path替换为dir，结果为`/dir/to/my.file.txt`
echo $(file/path/dir)

# 将所有的path替换为dir，纯字符串替换
echo ${file//path/dir}

# 如果$file没有设定，则使用my.file.txt作为返回值，$file为空值不会处理
echo ${file-my.file.txt}

# 如果$file没有设定或为空时，则使用my.file.txt作为返回值
echo ${file:-my.file.txt}

# 如果$file没有设定，则使用my.file.txt作为返回值，$file为空值不会处理，同时为$file赋值
echo ${file=my.file.txt}

# 如果$file没有设定或为空时，则使用my.file.txt作为返回值，同时为$file赋值
echo ${file:=my.file.txt}

# 如果$file没有设定，则将my.file.txt输出至STDERR，$file为空值不会处理，同时为$file赋值
echo ${file?my.file.txt}

# 如果$file没有设定或为空时，则将my.file.txt输出至STDERR，同时为$file赋值
echo ${file:?my.file.txt}
```

**获取脚本绝对路径**

```shell
#!/bin/bash

# 将 pwd 的输出赋给 shell_dir
shell_dir=$(cd $(dirname $0); pwd)

# 如果是软链接，readlink 打印软链接的内容，否则不返回值
# GNU/Linux 下的 readlink -f 获取 $0 参数解析后的全路径文件名
# Mac 上 readlink 无 -f 选项
# dirname 返回文件或目录所在的目录
shell_dir=$(dirname $(readlink -f $0))

# GNU/Linux 下的 realpath 返回 $0 解析后的全路径文件名
# Mac 上无此命令
shell_dir=$(dirname $(realpath $0))
```

**获取进程下的所有线程信息**

```shell
# 获取进程 pid 中创建的所有线程信息
$ ps -T -p <pid>
$ top -H -p <pid>
```

**获取进程信息的某一个字段**

```shell
# 获取进程 pid 的启动时间和启动命令(mac 上是comm)
$ ps -o lstart,cmd -p <pid>
# 通过进程名称来获取特定进程的输出(bash 是 /proc/<pid>/comm 中的内容，该命令不适用 macOS)
$ ps -o lstart,pid -C bash
```

**spawn/expect/send**

questions.sh 脚本
```shell
#!/bin/bash

echo "Hello, who are you?"

read $REPLY

echo "Can I ask you some questions?"

read $REPLY

echo "What is your favorite topic?"

read $REPLY
```

expect 脚本
```shell
#!/usr/bin/expect -f

set timeout 900  # 设置交互超时时间 6 分钟

spawn ./questions.sh

expect "Hello, who are you?\r"

send -- "Im Adam\r"     # 发送的消息要带 \r

expect "Can I ask you some questions?\r"

send -- "Sure\r"

expect "What is your favorite topic?\r"

send -- "Technology\r"

expect eof
```

**查看当前使用的shell类型及如何更换**

```
# 当前使用的 shell 类型
echo $0
echo $SHELL

# 查看系统支持的 shell 类型
cat /etc/shells

# 更换 shell
chsh -s /bin/zsh
```

**iostat**

该命令是通过在一个时间间隔内把 /proc/diskstats 中的内容读出来进行计算，得出磁盘的当前各项性能指标。

```shell
# 每隔一秒统计一次磁盘性能，第一次结果不准确，忽略
[vagrant@localhost ~]$ iostat -x 1
Linux 3.10.0-693.5.2.el7.x86_64 (localhost.localdomain)     12/30/2018  _x86_64_(1 CPU)

avg-cpu:  %user   %nice %system %iowait  %st    eal   %idle
           0.00    0.00    0.04    0.00    0.00   99.95

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.01    0.15    0.06     4.14     1.76    56.92     0.00    3.42    2.25    6.38   0.77   0.02
dm-0              0.00     0.00    0.09    0.06     3.86     1.70    74.60     0.00    4.97    3.41    7.59   1.04   0.02
dm-1              0.00     0.00    0.00    0.00     0.06     0.00    47.40     0.00    2.27    2.27    0.00   1.89   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          0.00    0.00    0.98    0.00    0.00   99.02

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

**dstat**

dstat 可用于替换 vmstat, iostat 及 ifstat

```shell
# dstat -all --disk-util
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system-- ---load-avg--- ---load-avg--- vda--vdb-
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw | 1m   5m  15m | 1m   5m  15m |util:util
  4   1  94   0   0   0|  58M   99M|   0     0 |   0     0 |  26k   20k|59.2 61.3 58.8|59.2 61.3 58.8|0.25:15.7
  2  22  75   1   0   0| 671M  550M| 420M  720M|   0     0 | 150k   28k|59.2 61.3 58.8|59.2 61.3 58.8|   0: 100
  3  21  75   1   0   0| 678M  599M| 465M  746M|   0     0 | 156k   34k|59.2 61.3 58.8|59.2 61.3 58.8|   0: 100
  4  20  75   1   0   0| 639M  646M| 498M  726M|   0     0 | 156k   33k|62.6 62.0 59.0|62.6 62.0 59.0|   0: 100
  4  21  75   1   0   0| 741M  670M| 510M  762M|   0     0 | 170k   38k|62.6 62.0 59.0|62.6 62.0 59.0|   0: 100
  4  20  75   1   0   0| 728M  684M| 766M  775M|   0     0 | 172k   39k|62.6 62.0 59.0|62.6 62.0 59.0|0.10:99.8
  4  19  76   1   0   0| 698M  750M| 741M  762M|   0     0 | 172k   39k|62.6 62.0 59.0|62.6 62.0 59.0|   0: 100
  5  19  75   1   0   0| 739M  636M| 829M  832M|   0     0 | 184k   45k|62.6 62.0 59.0|62.6 62.0 59.0|   0: 100
  4  18  76   1   0   0| 726M  680M| 769M  784M|   0     0 | 176k   42k|65.3 62.5 59.2|65.3 62.5 59.2|   0: 100
  4  20  76   1   0   0| 675M  755M| 721M  708M|   0     0 | 171k   38k|65.3 62.5 59.2|65.3 62.5 59.2|   0:99.9
  4  20  75   1   0   0| 710M  703M| 767M  750M|   0     0 | 175k   41k|65.3 62.5 59.2|65.3 62.5 59.2|0.10: 100
```

**mpstat**

用于查看 CPU 的时间消耗百分比

```shell
root@bytelligent0:~# mpstat -P ALL 2 1
Linux 5.4.0-80-generic (bytelligent0)  08/13/21         _x86_64_     (8 CPU)

12:08:39     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
12:08:41     all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
12:08:41       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
```

**smem**

[smem](https://www.selenic.com/smem/) is a tool that can give numerous reports on memory usage on Linux systems.

**查看哪些进程使用了 swap**

```
for file in /proc/*/status
do 
  awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' $file 
done
```

**查看网络中断绑核**

```shell
cat /proc/interrupts | grep PCI-MSI-edge | grep -E 'output|input' | awk -F':' '{print $1}'| while read n; do cat /proc/irq/$n/smp_affinity_list; done
```

**macos 查看 Listen 端口**

```shell
# netstat
netstat -anvp tcp | awk 'NR<3 || /LISTEN/'

# lsof
sudo lsof -PiTCP -sTCP:LISTEN
```

**查看运营商分配的公网IP**

```shell
curl cip.cc
curl myipip.net
```

**在不知道 root 密码的情况下登录 root 账号**

当然前提是你必须有一个具有 sudo 权限的账号，然后执行如下命令:

```shell
sudo bash -c bash
```

**yt-dlp && ffmpeg**

```shell
# download bestvideo and bestaudio
yt-dlp -f 'bv,ba' <videid>

# https://www.reddit.com/r/ffmpeg/comments/100nhxj/how_to_merge_audio_file_m4a_with_video_file_mp4/
# merge audio file (M4a) with video file (MP4) using ffmpeg
ffmpeg -i video_file -i audio_file -c:v copy -c:a aac out_file.mp4
```

<h4>Tips</h4>

- shell 命令前加反斜线可以避免使用 alias 命令，如 `\ls`

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Is there a way to see details of all the threads that a process has in Linux?][ref1]<br>
2 [How to view threads of a process on Linux][ref2]<br>
3 [Expect command and how to automate shell scripts like magic][ref3]<br>
4 [容易被误读的IOSTAT][ref4]<br>
5 [Using Bracket Expressions][ref5]<br>
6 [backslash at the beginning of a command][ref6]<br>
</span>

[ref1]: https://unix.stackexchange.com/questions/892/is-there-a-way-to-see-details-of-all-the-threads-that-a-process-has-in-linux
[ref2]: http://ask.xmodulo.com/view-threads-process-linux.html
[ref3]: https://likegeeks.com/expect-command/
[ref4]: http://linuxperf.com/?p=156
[ref5]: https://www.gnu.org/software/gawk/manual/html_node/Bracket-Expressions.html#Bracket-Expressions
[ref6]: https://serverfault.com/questions/480271/backslash-at-the-beginning-of-a-command
