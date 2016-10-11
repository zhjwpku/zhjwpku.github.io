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

**command -v**

{% highlight shell %}
#!/bin/bash

# 是否存在http命令
if ! command -v http &>/dev/null ; then
  echo "httpie not installed, 'apt-get install httpie', 'yum install httpie' or 'brew install httpie'"
  exit 1
fi

{% endhighlight %}

