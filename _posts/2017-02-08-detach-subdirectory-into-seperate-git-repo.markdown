---
layout: post
title: 分离子目录到单独的 Git Repo
date: 2017-02-08 20:00:00 +0800
tags:
- git
- subtree
---

微服务框架盛行的时代，开发人员希望将各模块解耦，有时会涉及代码仓库的拆分，如将一个大仓库中的某个子目录作为一个新的独立仓库拆分出去。

有两种比较直观的方式：

1. 把子目录移出代码仓库，在移出的子目录里 git init 创建新仓库 （缺点：新仓库不保留文件历史）
2. 复制代码仓库，删除子目录以外的其他文件 （缺点： 其他的目录文件还存在于仓库中，git checkout 可查看，而且仓库巨大）

以上两种方式不可取，使用 `git subtree` 命令能够简化这样的操作。

假设现有目录仓库如下：

{% highlight shell %}
tree ~/Code/node-browser-compat

node-browser-compat
├── ArrayBuffer
├── Audio
├── Blob
├── FormData
├── atob
├── btoa
├── location
└── navigator
{% endhighlight %}

我们把`btoa`目录分离成一个独立的 git repo

{% highlight shell %}
# git subtree split -P <prefix> [OPTIONS] [<commit>]
# 合成btoa子目录树的历史记录，并将历史记录存放在btoa-only分支
pushd ~/Code/node-browser-compat/
git subtree split -P btoa -b btoa-only
popd
{% endhighlight %}

上述操作完成后，分支 btoa-only 中只存在 btoa 子目录树中的文件和历史记录。

创建一个新的仓库，并将 btoa-only 分支的代码 pull 到新仓库中。

{% highlight shell %}
mkdir ~/Code/btoa/
pushd ~/Code/btoa/
git init
git pull ~/Code/node-browser-compat btoa-only
{% endhighlight %}

可以将仓库推到 github 或私有仓库中了，步骤略。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Detach (move) subdirectory into separate Git repository][url_name]
</span>

[url_name]: http://stackoverflow.com/questions/359424/detach-move-subdirectory-into-separate-git-repository/17864475#17864475
