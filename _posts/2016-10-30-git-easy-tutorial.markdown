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
git config --global alias.ci "commit -s"
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

git本地由三个部分组成：工作区（working directory），暂存区（snapshot/stage/index）和本地库（local repo）。.git目录是仓库的灵魂，.git目录里边包括暂存区和本地库，.git目录外的地方是工作区。git中包含四类对象：commit提交对象，tree目录，blob文件，tag标记。

基本的Git工作流为：

- 修改工作区中的文件
- 标记修改过的文件为Stage状态，将它们的Snapshot加入到暂存区
- 提交，将暂存区的文件存储到本地Git仓库


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

<h4>命令解释</h4>

{% highlight shell %}
$ git push origin master # 上传master分支
$ git fetch # 从远程仓库下载更新代码到本地仓库，工作区和暂存区不变
$ git pull  # = git fetch + git checkout 下载到本地仓库，并更新到暂存区和工作区
$ git add -i # 进入交互式add
$ git format-patch -n # 格式化生成patch
$ git blame <file> # 显示文件的每行的版本以及最后更改的人员
$ git am [<mbox>|<Maildir>] # Apply a series of patches from a mailbox
$ git apply <patch> # Apply a patch to files and/or to the index

# clone时提供用户名、密码
$ git clone https://username:password@github.com/username/repository.git

# git diff
$ git diff # 比较工作目录和暂存区的差异
$ git diff --cached # 比较暂存区和HEAD的差异
$ git diff --name-only # 只列出不同文件的名字，而不显示具体的差异之处
$ git diff --stat # 显示diff变更的数据，--numstat会显示增加的行数和删除的行数

# git checkout 恢复工作区的内容
$ git checkout test.txt # 从暂存区恢复到工作区，相当于撤销test.txt文件未add的修改
$ git checkout -f # 从本地库恢复到暂存区和工作区，强制修改工作区的内容，一般对于当前修改不满意，放弃修改后重新开始
$ git checkout HEAD test.txt # 从本地库恢复到暂存区和工作区，取HEAD中的test.txt文件
$ git checkout $(git rev-list master -n -1 --first-parent --before=2016-10-30) # 恢复到2016-10-30号前的最后一个版本

# git reset
$ git reset --soft <commit> # 暂存区和工作区中的内容不作任何改变，仅把HEAD指向<commit>。这个模式下可以在不改动当前工作环境的情况下，改变head指针，即切换到任意commit
$ git reset --mixed <commit> # reset的默认模式。暂存区中的内容会发生变化（恢复到工作区），但工作区中的内容不变
$ git reset --hard <commit> # 暂存区和工作区都会发生变化，还原到<commit>

# git revert
$ git revert <commit> # 撤销到<commit>，且撤销会作为一次提交进行保存
$ git revert -n master~5..master~2 # 丢弃从最近第五个commit（含）到第二个（不含），但是不自动生成commit，仅修改工作区和暂存区

# git tag
$ git tag # list all tag
$ git tag -a 3.10.1 -m "Release 3.10.1" # 打标签，最好在Pull Request合并之后
$ git show <tag> # 显示tag详情
$ git tag -d 3.10.1 # 删除标签
$ git push origin --tags # 上传本地标签
$ git checkout <tag> # 切换到tag，同切换分支一样

# git stash
# case.1 工作空间与upstream冲突，导致pull失败
$ git stash
$ git pull
$ git stash pop

# case.2 当你正处于某个任务之中的时候，老板突然让你去修复一个不相干的bug
# 一般你会这样操作
# ... hack hack hack ...
$ git checkout -b my_wip
$ git commit -a -m "WIP"
$ git checkout master
$ edit emergency fix # 紧急修复
$ git commit -a -m "Fix in a hurry"
$ git checkout my_wip
$ git reset --soft HEAD^
# ... continue hacking ...

# 可以使用stash简化操作
# ... hack hack hack ...
$ git stash
$ edit emergency fix
$ git commit -a -m "Fix in a hurry"
$ git stash pop
# ... continue hacking ...

# 可能存在多个Stash的内容，使用栈来管理，pop会从最近一个stash中读取内容并恢复
$ git stash list # 显示Git栈内的所有备份
$ git stash clear # 清空Git栈
$ git stash apply <版本号> # 取出特定的版本号对应的工作空间
{% endhighlight %}

**2016-12-07 更新**

`git reset`往往是个危险的命令，假如使用了`git reset --hard`，又没有记下`reset`前的`commit id`，那么该如何挽救呢？这时`reflog`应该能够帮你一把。

![gitreflog](/assets/201612/gitreflog.png)

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. Git Basics: [https://git-scm.com/book/en/v2/Getting-Started-Git-Basics][gitbasics]<br>
2. [http://www-cs-students.stanford.edu/~blynn/gitmagic/intl/zh_cn/][gitmagic]<br>
3. [https://www.atlassian.com/git/tutorials/][atlassiangit]<br>
4. Comparing Workflows: [https://www.atlassian.com/git/tutorials/comparing-workflows][workflow]<br>
5. 蒋鑫. [Git 权威指南](/assets/pdf/GotGit.pdf). 机械工业出版社. 2011.<br>
</span>

[installgit]: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git
[download-a-single-folder-or-directory-from-a-github-repo]: http://stackoverflow.com/questions/7106012/download-a-single-folder-or-directory-from-a-github-repo
[how-do-i-clone-a-subdirectory-only-of-a-git-repository]: http://stackoverflow.com/questions/600079/how-do-i-clone-a-subdirectory-only-of-a-git-repository
[gitbasics]: https://git-scm.com/book/en/v2/Getting-Started-Git-Basics
[gitbranch]: https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
[gitmagic]: http://www-cs-students.stanford.edu/~blynn/gitmagic/intl/zh_cn/
[atlassiangit]: https://www.atlassian.com/git/tutorials/
[workflow]: https://www.atlassian.com/git/tutorials/comparing-workflows
