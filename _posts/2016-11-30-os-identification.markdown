---
layout: post
title: Shell脚本判断OS版本
date: 2016-11-30 22:00:00 +0800
tags:
- uname
- os-release
- lsb_release
---

Shell脚本中经常需要对操作系统的版本类型类型进行判断。这里介绍几种常用的方法，如有遗漏请包涵 :)

<h4>uname</h4>

uname ( short for unix name ) 用来打印系统信息，在shell脚本中可以这样使用：

{% highlight shell %}
if [ x`uname`x = xFreeBSDx ]; then
    sudo pkg install -yq \
        devel/git \
        ...
else
    ...
{% endhighlight %}

<h4>/etc/os-release</h4>

在Linux发行版中都会有`/etc/os-release`文件，可以通过`source`命令将文件中的K/V值引入到上下文中。

CentOS 7:
{% highlight shell %}
$ cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
{% endhighlight %}

Ubuntu 16.04:
{% highlight shell %}
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="16.04.1 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.1 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
UBUNTU_CODENAME=xenial
{% endhighlight %}

在Shell脚本中这样使用：

{% highlight shell %}
#!/bin/bash

source /etc/os-release
case $ID in
debian|ubuntu|devuan)
    sudo apt-get install lsb-release
    ;;
centos|fedora|rhel)
    yumdnf="yum"
    if test "$(echo "$VERSION_ID >= 22" | bc)" -ne 0; then
        yumdnf="dnf"
    fi
    sudo $yumdnf install -y redhat-lsb-core
    ;;
*)
    exit 1
    ;;
esac
{% endhighlight %}

<h4>lsb_release</h4>

在上一节的例子中已经将`lsb_release` ( Linux Standard Base ) 工具安装到系统中，使用方法：

Ubuntu 16.04
{% highlight shell %}
$ lsb_release -sc
xenial
{% endhighlight %}

CentOS 7
{% highlight shell %}
$ lsb_release -si
CentOS
$ lsb_release -rs
7.1.1503
$ lsb_release -rs | cut -f1 -d.
7
$ lsb_release -rs | cut -f2 -d.
1
$ lsb_release -rs | cut -f3 -d.
1503
{% endhighlight %}
