---
layout: post
title: Xshell alternative for Mac
date: 2017-03-12 13:00:00 +0800
tags:
- storm
- tmux
---

Xshell 绝对称得上 Windows 平台最让人留恋的工具之一。笔者最近从 *Win7 + Ubuntu 虚拟机* 切换到了 Mac 工作平台 (开发环境的配置见 [Mac OS X Setup Guide][ref1])，在寻找像 Xshell 这样的工具时遇到了一些麻烦，好在最终找到了，并且体验不亚于 Xshell。 => [Storm][storm] + [Tmux][tmux]。

<h4>Storm</h4>

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

**Alias**

给子命令一个别名能让 storm 用起来更加方便。

{% highlight shell %}
→ ~ $ mkdir -p ~/.stormssh
# 创建配置文件并添加相应的别名
→ ~ $ vim ~/.stormssh/config
# 如下为config文件中的内容
{
    "aliases": {
        "list": ["ls", "ll"],
        "delete": ["rm"]
    }
}
{% endhighlight %}

<h4>Tmux</h4>

**安装**

{% highlight shell %}
→ ~ $ brew install tmux
{% endhighlight %}

**基本使用方法**

{% highlight shell %}
→ ~ $ tmux  # 进入Tmux模式

# 键入Ctrl+b后松开，c 新建一个Tab
# 键入Ctrl+b后松开，, 重命名Tab页
# 键入Ctrl+b后松开，n 移动到下一个Tab页
# 键入Ctrl+b后松开，p 移动到上一个Tab页
# 键入Ctrl+b后松开，% 将 Window 垂直切分
# 键入Ctrl+b后松开，" 将 Window 水平切分
# 键入Ctrl+b后松开，? 显示帮助菜单
# 键入Ctrl+b后松开，方向键在各窗口中切换光标
# 键入Ctrl+b后松开，空格键重新排列多窗口的布局
# 键入Ctrl+b后松开, d 或 :detach 将当前Session分离

→ ~ $ tmux ls   # 列出当前所有Session

→ ~ $ tmux attach -t 0 # 将Session 0恢复
{% endhighlight %}

Tmux 跟 Screen 很相似，它们的命令对比可以参考 [tmux & screen cheat-sheet][ref2]。

[storm]: https://github.com/emre/storm
[tmux]: https://github.com/tmux/tmux
[ref1]: http://sourabhbajaj.com/mac-setup/
[ref2]: http://www.dayid.org/comp/tm.html
