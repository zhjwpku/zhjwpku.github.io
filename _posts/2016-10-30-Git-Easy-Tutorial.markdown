---
layout: post
title: Git Easy Tutorial
date: 2016-10-30 20:00:00 +0800
tags:
- git
---

本文记录了我自己使用Git的习惯，并列出了一些优秀的Git教程。

<h4>安装</h4>

[Getting Started Installing Git][installgit]

<h4>配置</h4>

git的配置分为系统（System）、全局（Global）和本地（Local）三个级别，对应存放在系统不同目录的三个`config`文件。可以通过`git config --system/global/local`相关命令进行增删查改，也可通过编辑配置文件。通常最常用的是修改Global级别的配置，Linux存放在~/.gitconfig文件中。

常用的配置命令：

{% highlight shell %}
git config --global user.email zhjwpku@gmail.com
git config --global user.name "Zhao Junwang"
git config --global core.editor vim 
git config --global credential.helper store
git config --global push.default simple

# use less pager by default
git config --global --replace-all core.pager "less -+S"

git config --global --color.ui auto

# alias
git config --global alias.dt difftool
git config --global alias.mt "mergetool -y"
git config --global alias.br "branch -vv"
git config --global alias.co checkout
git config --global alias.cob "checkout -b"
git config --global alias.ci commit
git config --global alias.cp cherry-pick

#  去掉默认的前缀'a b'
git config --global alias.df "diff --no-prefix"

#  按单词diff，而不是行
git config --global alias.dw "diff --no-prefix --color-words"

#  与HEAD^进行diff
git config --global alias.dh "diff --no-prefix HEAD^"
git config --global alias.st status
git config --global alias.pl "pull --ff-only"
git config --global alias.ps push

git config --global alias.lg "log --graph --format='%C(auto)%h%C(reset) %C(dim white)%an%C(reset) %C(green)%ai%C(reset) %C(auto)%d%C(reset)%n   %s'"
git config --global alias.lg10 "log --graph --pretty=format:'%C(yellow)%h%C(auto)%d%Creset %s %C(white)- %an, %ar%Creset' -10"
git config --global alias.lg20 "log --graph --pretty=format:'%C(yellow)%h%C(auto)%d%Creset %s %C(white)- %an, %ar%Creset' -20"
git config --global alias.lg30 "log --graph --pretty=format:'%C(yellow)%h%C(auto)%d%Creset %s %C(white)- %an, %ar%Creset' -30"
git config --global alias.fp "format-patch --stdout --no-prefix"

{% endhighlight %}

<h4>下载单一文件</h4>

很多时候我们只需要下载代码仓库的单一文件，而不想将下载整个repo，进入文件页面，点击文件的`Raw`视图按钮，然后使用`wget`下载即可。Windows下在Raw视图界面右击另存为即可。另外，安装Chrome扩展插件Octotree，当鼠标停留到文件位置的时候会显示下载的提示，点击即可下载。

{% highlight shell %}
wget -O ~/.git-completion.bash https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash
wget -O ~/.git-prompt.sh https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh
{% endhighlight %}

如果想要下载某个文件夹，参照：

- [download-a-single-folder-or-directory-from-a-github-repo][download-a-single-folder-or-directory-from-a-github-repo]
- [how-do-i-clone-a-subdirectory-only-of-a-git-repository][how-do-i-clone-a-subdirectory-only-of-a-git-repository]

<h4>Prompt和Completion</h4>

Prompt可以修改命令提示符，而Completion用于自动补全。

修改~/.bashrc文件，添加如下内容

{% highlight shell %}
# git
# [[ -f ~/.git-completion.bash ]] && . ~/.git-completion.bash
[[ -f ~/.git-prompt.sh ]] && . ~/.git-prompt.sh
export GIT_PS1_SHOWDIRTYSTATE=1
export PS1='\[\e[1;36m\]→\[\e[m\] \[\e[0;32m\]\w\[\e[0;35m\]$(__git_ps1)\[\e[1;32m\] \$\[\e[m\] '
{% endhighlight %}

<h4><a href="https://git-scm.com/book/en/v2/Getting-Started-Git-Basics#The-Three-States">Git Three States</a></h4>

![ThreeStates](/assets/201610/ThreeStates.png){: width="500px"}

Git中的文件可以分为三种状态：Commited，Modified和Staging。Commited为已经安全保存在本地仓库的数据，Modified为有改动但是并没有提交到本地仓库的文件，而Staged是将改动过的文件标记为可以进入下次commit snapshot的状态。

基本的Git工作流为：

- 修改working directory中的文件
- 标记修改过的文件为Stage状态，将它们的Snapshot加入到staging区
- 提交，将staging区的文件存储到本地Git仓库

<h4>Git Branching</h4>

有人称Git的分支模型为它"Killer feature"，因为对于其它VCS，分支意味着一份新的源码拷贝，这是一个昂贵的操作。而Git的分支模型是非常轻量的。

具体参照 [https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell][gitbranch]

<h4>解决冲突</h4>

{% highlight shell %}
$ git fetch develop  # 远程仓库的分支
$ git checkout bug-feature # 出现冲突的分支
$ git rebase develop
# merge conflict ...
# 把有冲突的文件进行修改
$ git add . #解决完冲突需要把修改过的文件git add到staging区
$ git rebase --continue
# 可能还会出现冲突，继续解决
...
$ git push -f # 将解决完之后的代码强制推送到远程仓库
{% endhighlight %}

<h4>删除远程分支</h4>

{% highlight shell %}
$ git checkout develop
$ git pull  # 将merge的bug-feature更新到develop分支
$ git branch -d bug-feature
$ git push --delete origin bug-feature # 或者在网页上删除，由Reviewer或自己删除
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. Git Basics: [https://git-scm.com/book/en/v2/Getting-Started-Git-Basics][gitbasics]<br>
2. [http://www-cs-students.stanford.edu/~blynn/gitmagic/intl/zh_cn/][gitmagic]<br>
3. [https://www.atlassian.com/git/tutorials/][atlassiangit]<br>
4. Comparing Workflows: [https://www.atlassian.com/git/tutorials/comparing-workflows][workflow]
</span>

[installgit]: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git
[download-a-single-folder-or-directory-from-a-github-repo]: http://stackoverflow.com/questions/7106012/download-a-single-folder-or-directory-from-a-github-repo
[how-do-i-clone-a-subdirectory-only-of-a-git-repository]: http://stackoverflow.com/questions/600079/how-do-i-clone-a-subdirectory-only-of-a-git-repository
[gitbasics]: https://git-scm.com/book/en/v2/Getting-Started-Git-Basics
[gitbranch]: https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
[gitmagic]: http://www-cs-students.stanford.edu/~blynn/gitmagic/intl/zh_cn/
[atlassiangit]: https://www.atlassian.com/git/tutorials/
[workflow]: https://www.atlassian.com/git/tutorials/comparing-workflows
