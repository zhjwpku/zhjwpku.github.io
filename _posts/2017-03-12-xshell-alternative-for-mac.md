---
layout: post
title: Xshell alternative for Mac
date: 2017-03-12 13:00:00 +0800
tags:
- storm
---

Xshell 绝对称得上 Windows 平台最让人留恋的工具之一。笔者最近从 *Win7 + Ubuntu 虚拟机* 切换到了 Mac 工作平台，在寻找像 Xshell 这样的工具时遇到了一些麻烦，好在最终找到了，并且体验不亚于 Xshell。 => [Storm][storm]。

**安装**

{% highlight shell %}
→ ~ $ brew install stormssh
{% endhighlight %}

**添加 AWS EC2**

{% highlight shell %}
→ ~ $ mkdir -p ~/.ssh/keys
# move the ec2 public key (say, app.pem) to ~/.ssh/keys/
→ ~ $ chmod 600 app.pem
→ ~ $ mv app.pem ~/.ssh/keys/
# Add entry to storm
→ ~ $ storm add aws-app ubuntu@54.171.111.110 --id_file=/Users/zhjwpku/.ssh/keys/app.pem
success  aws-app added to your ssh config. you can connect it by typing "ssh aws-app".
# List the entry
→ ~ $ storm list
 Listing entries:

    aws-app -> ubuntu@54.171.111.110:22
	[custom options] identityfile="/Users/zhjwpku/.ssh/keys/app.pem"
{% endhighlight %}

**添加局域网Host**

{% highlight shell %}
→ ~ $ storm add ubuntu root@10.0.63.179
success  ubuntu added to your ssh config. you can connect it by typing "ssh ubuntu".
# 设置免密码登录
→ ~ $ ssh-copy-id ubuntu
{% endhighlight %}

如此这样将自己管理或使用的机器都以 entry 的形式添加到 Storm 中，在需要的时候使用 `list` 或 `search` 查看机器，然后用 *ssh host-entry* 便就可以直接登录到远程机器。

[storm]: https://github.com/emre/storm
