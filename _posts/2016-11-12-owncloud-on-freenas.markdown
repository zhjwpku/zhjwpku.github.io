---
layout: post
title: ownCloud on FreeNAS
date: 2016-11-12 20:00:00 +0800
tags:
- freenas
- owncloud
---

360云盘的个人云盘服务即将停止，让我之前DIY家用NAS的想法更加坚定了。本文介绍了使用FreeNAS和ownCloud打造家庭云盘服务的过程。

<h4>Freenas安装</h4>

[FreeNAS][freenas]是一款广受赞誉的开源免费NAS操作系统。它能把普通台式机瞬间变成一台多功能NAS服务器。不但适用于企业文件共享，同样适用于打造家庭媒体中心。

趁ComputerUniverse搞[活动][cu]买了一台惠普ProLiant MicroServer Gen8 G1610T服务器，内置MicroSD或USB可以作为系统盘接口，这样就留有四个盘位给硬盘。

**制作安装盘**

从FreeNAS官网下载稳定版本的ISO镜像，随便找一个U盘，使用如下命令制作安装盘，/dev/sdb是U盘的设备文件

{% highlight shell %}
> dd if=FreeNAS-9.10-U3.iso of=/dev/sdb bs=64k
{% endhighlight %}

**MicroServer iLO安装**

MicroServer Gen8有一个iLO专用网口，用来管理计算机的启动、关机、启动顺序设置等操作，并提供一个远程的控制台，不需要连接显示器、鼠标键盘就能够进行操作。

![ilo0](/assets/201611/ilo0.png)

使用`.NET IRC`来对计算机进行操作，这里注意要使用IE浏览器或Firefox，使用Chrome会启动不了控制台。点击`Launch`会出现如下界面：

![ilo01](/assets/201611/ilo01.png)

点击模拟的开机按钮，将计算机启动：

![ilo1](/assets/201611/ilo1.png)

![ilo2](/assets/201611/ilo2.png)

由于设置了从U盘启动：

![ilo_usb](/assets/201611/ilo_usb.png)

会直接进入FreeNAS安装界面：

![freenas0](/assets/201611/freenas0.png)

![freenas1](/assets/201611/freenas1.png)

选择将系统安装到内置的MicroSD上：

![freenas2](/assets/201611/freenas2.png)

![freenas3](/assets/201611/freenas3.png)

输入root密码，用于登陆FreeNAS管理界面：

![freenas4](/assets/201611/freenas4.png)

安装成功：

![freenas5](/assets/201611/freenas5.png)

通过浏览器管理FreeNAS：

![freenas6](/assets/201611/freenas6.png)

创建Volume及DataSet的部分比较容易上手，略。

**Windows CIFS/SMB**

CIFS/SMB是由微软公司开发的协议，用于连接Windows客户机和服务器，执行文件共享和打印等任务，微软操作系统家族和几乎所有Unix服务器都支持SMB协议。CIFS/SMB共享方式是用的最多，也是兼容性最好的共享方式之一。

在FreeNAS中开启SMB服务：

![smbservice](/assets/201611/smb_service.png)

创建一个Window SMB共享目录：

![smb](/assets/201611/SMB.png)

创建好之后，在Window系统中添加一个网络位置：

![add_smb](/assets/201611/add_smb.png)

Android本身对SMB有很好的支持，iOS推荐一个APP -> `FileExplorer`，SMB的权限设置参考[FreeNAS CIFS共享权限管理][auth]。

<h4>ownCloud Plugin</h4>

其实ownCloud插件的安装也是比较简单的操作，安装后配置一下挂载的Dataset，然后启动服务即可。但是在一次升级的过程中遇到了问题，在这做个记录。

第一次用上传PBI的方式安装ownCloud，版本为9.1.1，用了一段时间后提示有新版本可更新。更新之后遇到问题了，登陆ownCloud界面出现提示，大致意思是`你的ownCloud实例太大了，请在控制台进行命令行更新`，于是我按照提示，进行更新，提示找不到PDO类：

![php_pdoerr](/assets/201611/php_pdoerr.png)

感觉是路径不对，查看php-config:

![php-config](/assets/201611/php-config.png)

都是绝对路径，于是尝试做了个软连接：

![oc-softlink](/assets/201611/oc-softlink.png)

No lucky：

![upgrade_err](/assets/201611/upgrade_err.png)

因为不确定该向FreeNAS还是ownCloud求教，于是先在github上给ownCloud提[issue26640][issue26640]，被告知应该问FreeNAS，于是又在FreeNAS社区提[issue18949][issue18949]，知道了问题的原因——是因为我使用方式不对，应该使用FreeNAS的Jail系列命令：

{% highlight shell %}
> jls
> jexec 1 /bin/csh
{% endhighlight %}

这将会进入jail自己的根目录，感觉跟docker类似（原理不知），再次运行命令，出现不同的错误：

![jexec](/assets/201611/jexec.png)

看着像权限的问题，于是将jail的Dataset权限改为root:wheel，运行之后竟然更新成功了。然后打开浏览器输入ownCloud的地址，出现如下提示：

![maintainance](/assets/201611/maintainance_mode.png)

Google之后，进行如下操作：

![maintainance off](/assets/201611/maintainance_mode_off.png)

之后又对jail的权限鼓捣了半天，最后得以成功登陆。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [https://bugs.freenas.org/issues/18949][issue18949]<br>
2. [Blocked in maintenance mode after upgrade 8.0.4 to 8.1][issue17440]
</span>

[freenas]: http://www.freenas.org/
[cu]: http://www.smzdm.com/p/6517684/
[auth]: http://www.getnas.com/2015/02/729.html
[issue26640]: https://github.com/owncloud/core/issues/26640
[issue18949]: https://bugs.freenas.org/issues/18949
[issue17440]: https://github.com/owncloud/core/issues/17440
