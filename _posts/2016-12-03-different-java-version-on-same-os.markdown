---
layout: post
title: 同一系统安装多个Java版本
date: 2016-12-03 22:00:00 +0800
tags:
- java
- alternatives
---

毫无疑问，在一个操作系统上可以安装多个不同的Java版本，在命令行运行`yum install java<tab><tab>`便知一二。如果只是运行java程序，安装`java-x.x.x-openjdk.x86_64`就可以了，而开发人员还需安装`java-x.x.x-openjdk-devel.x86_64`以提供`javac`等编译环境。

![yuminstalljava](/assets/201612/yuminstalljava.png)

安装之后会在`/etc/alternatives/`目录下创建相应的软链接。

![lsjre](/assets/201612/lsjre.png)

这些软链接通过`alternetives` (update-alternatives) 来管理，例如使用该命令更改默认的java运行时环境：

![alternativesconfigjava](/assets/201612/alternativesconfigjava.png)

`alternatives`是一个用来维护默认命令的符号链接的常用工具，常用命令有`--list`、`--install`、`--display`，`--auto`，`--config`等。

简单介绍下`--install`的使用方法，假设我们现在想创建 openjdk-1.8.0 的执行连接，可执行如下命令：

{% highlight shell %}
$ sudo alternatives --install /usr/bin/java8 java8 /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.el7_2.x86_64/bin/java 3
                         |          |          |                              |                                          |
                        安装     链接位置      名字                       执行文件路径                                  优先级
{% endhighlight %}

这样就可以在命令行使用`java8`来运行java程序了。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [How to install java on CentOS and Fedora][ref]
</span>

[ref]: https://www.digitalocean.com/community/tutorials/how-to-install-java-on-centos-and-fedora
