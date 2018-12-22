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
