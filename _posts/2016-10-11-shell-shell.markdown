---
layout: post
title: Shell Scripts
date: 2016-10-11 18:00:00 +0800
tags:
- shell
---

哇！发现了好多不懂的Shell命令，记录一下...

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

{% highlight shell %}
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
{% endhighlight %}

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


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Is there a way to see details of all the threads that a process has in Linux?
][ref1]<br>
2 [How to view threads of a process on Linux][ref2]<br>
3 [Expect command and how to automate shell scripts like magic][ref3]<br>
</span>

[ref1]: https://unix.stackexchange.com/questions/892/is-there-a-way-to-see-details-of-all-the-threads-that-a-process-has-in-linux
[ref2]: http://ask.xmodulo.com/view-threads-process-linux.html
[ref3]: https://likegeeks.com/expect-command/
