---
layout: post
title: Gitlab备份恢复
date: 2016-11-18 12:30:00 +0800
tags:
- gitlab
---

使用Gitlab社区版搭建代码管理平台的用户应该都会遇到备份恢复的问题。甚至有些用户想把代码仓库迁移到另外一台Gitlab Server上。本文介绍了了Gitlab的备份恢复及代码的迁移。

<h4>备份Gitlab</h4>

{% highlight shell %}
sudo gitlab-rake gitlab:backup:create
{% endhighlight %}

![backup](/assets/201611/backup.png)

生成的tar包默认存放在`/var/opt/gitlab/backups`目录下。如果是将Gitlab迁移到另外一台Gitlab服务器，需要将tar包拷贝到远程服务器上。

<h4>恢复Gitlab</h4>

{% highlight shell %}
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
# verify
sudo gitlab-ctl status

# This command will overwrite the contents of your GitLab database!
sudo gitlab-rake gitlab:backup:restore BACKUP=1479433301

# restart and check Gitlab
sudo gitlab-ctl start
sudo gitlab-rake gitlab:check SANITIZE=true
{% endhighlight %}

如果是迁移Gitlab，则需要目标Gitlab服务器的版本和进行备份的Gitlab服务器版本相同。

![version](/assets/201611/version_switch.png)

**更新目标Gitlab服务器版本**

上图提示目标服务器的版本需要升级到8.13.5。如果直接在命令行输入`yum update gitlab-ce`会将gitlab升级到最新的版本，而我们需要升级到特定的版本。Gitlab所有发行版都可以在[gitlab packagecloud][packagecloud]中找到，点击某个发行版，会提示：

{% highlight shell %}
> curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
> sudo yum install gitlab-ce-8.13.5-ce.0.el7.x86_64
{% endhighlight %}

由于我们要进行更新操作，用`update`替换`install`即可将Gitlab升级到特定的版本。

升级完成后进行仓库的恢复，在提示`yes/no`的地方都输入`yes`，否则会失败。

![restore](/assets/201611/restore.png)

*注：restore操作会删除原有的仓库及数据库表，如果想合并两个Gitlab Server请勿用*

<h4>Gitlab Repo迁移</h4>

两个团队A/B分别拥有各自的Gitlab Server，为了节省资源并加强交流学习，现将Gitlab Server进行合并——将A Server中的所有代码都迁移到B Server。这里不能使用backup/restore的方式，Gitlab在新建项目的时候有`Import project from 'Any repo by URL'`选项，可以使用这种方式来导入仓库。

这里涉及到两个团队账号管理的问题，由于在B Server上没有A Server的账户信息，因此需要重新创建用户或开放注册选项，如果A团队使用LDAP而B团队使用Native的账号管理方式，则可以将A团队的LDAP配置到B Server上。如果两个团队都使用LDAP，则考虑将两个团队的LDAP合并。

还有一个问题就是Group的问题，推荐用户登陆之后`Explore Groups`，在想要加入的组中`Request Access`，而组的管理员可以同意或拒绝。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [https://docs.gitlab.com/omnibus/update/README.html][updatereadme]<br>
2. [https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md][backup_restore]
</span>

[packagecloud]: https://packages.gitlab.com/gitlab/gitlab-ce?filter=rpms
[backup_restore]: https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md
[updatereadme]: https://docs.gitlab.com/omnibus/update/README.html
