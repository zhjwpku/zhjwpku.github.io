---
layout: post
title: Vagrant 初级实践
date: 2017-01-15 22:00:00 +0800
tags:
- vagrant
---

[Vagrant][vagrant] 是一个用来构建完整开发环境的工具，凭借易用的工作流及对自动化的专注，Vagrant 降低了开发环境的设置时间，提高了生产效率。

Vagrant 宣称支持各种虚拟化平台，VirtualBox、VMware、AWS、OpenStack，甚至是 Docker 容器及原生的 LXC，但笔者认为其最常用的使用场景是结合 VirtualBox 来快速构建虚拟机，适用于开发环境而非部署大量的生产环境，本文仅介绍 Vagrant 最基本的使用方法。

**2017-03-22 更新**

如果使用的provider为VirtualBox并且有配置`config.vm.synced_folder "data", "/data"`，会出现*mount: unknown filesystem type 'vboxsf'* 错误，安装vgrant-vbguest插件可以解决这个问题。

{% highlight shell %}
[root@c3po ~]# vagrant plugin install vagrant-vbguest
[root@c3po ~]# vagrant destroy && vagrant up
{% endhighlight %}

**--end--**

Vagrant 和 Docker 的区别可阅读以下链接：

[What is the difference between Docker and Vagrant? When should you use each one?][diff]

<h4>安装 VirtualBox</h4>

{% highlight shell %}
[root@c3po ~]# cd /etc/yum.repos.d
[root@c3po yum.repos.d]# wget http://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo
[root@c3po yum.repos.d]# cd
[root@c3po ~]# yum --enablerepo=epel install dkms
[root@c3po ~]# yum install VirtualBox-5.1.x86_64 -y
{% endhighlight %}

<h4>安装 Vagrant</h4>

去 Vagrant 官网[下载界面][download]下载 rpm 包并安装

{% highlight shell %}
[root@c3po ~]# wget https://releases.hashicorp.com/vagrant/1.9.1/vagrant_1.9.1_x86_64.rpm
[root@c3po ~]# rpm -i vagrant_1.9.1_x86_64.rpm
{% endhighlight %}

<h4>测试</h4>

通过 `Vagrant init` 下载一个 box 的配置文件，像 dockerhub 一样，Vagrant 也有自己的[镜像仓库][boxes]。使用官方 Ubuntu 14.04 配置。

{% highlight shell %}
[root@c3po ~]# mkdir vagrant_test && cd vagrant_test/
[root@c3po vagrant_test]# vagrant init ubuntu/trusty64
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
[root@c3po vagrant_test]# ls
Vagrantfile
{% endhighlight %}

修改 Vagrantfile，使得配置文件如下：

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/trusty64"

  # 配置两台虚拟机 vbox1 vbox2
  config.vm.define "vbox1" do |vbox1|
    # vagrant 有三种网络模式：port forward/host-only/bridge，本文使用host-only模式
    # 该模式可指定静态 IP
    vbox1.vm.network "private_network", ip: "192.168.33.10"
    # 这里可以指定一个脚本文件，也可以使用Puppet/Chef等配置管理工具
    vbox1.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y git
    SHELL
  end

  config.vm.define "vbox2" do |vbox2|
    vbox2.vm.network "private_network", ip: "192.168.33.11"
  end

end
{% endhighlight %}

启动：

{% highlight shell %}
[root@c3po vagrant_test]# vagrant up
...output omitted...
[root@c3po vagrant_test]# vagrant status
Current machine states:

vbox1                     running (virtualbox)
vbox2                     running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
{% endhighlight %}

检验 vbox1 是否安装了 git：

{% highlight shell %}
[root@c3po vagrant_test]# vagrant ssh vbox1
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-107-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Sat Jan 21 15:04:33 UTC 2017

  System load:  0.94              Processes:           79
  Usage of /:   3.6% of 39.34GB   Users logged in:     0
  Memory usage: 25%               IP address for eth0: 10.0.2.15
  Swap usage:   0%

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

New release '16.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@vagrant-ubuntu-trusty-64:~$ dpkg -l git
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                    Version          Architecture     Description
+++-=======================-================-================-===================================================
ii  git                     1:1.9.1-1ubuntu0 amd64            fast, scalable, distributed revision control system
vagrant@vagrant-ubuntu-trusty-64:~$ 
{% endhighlight %}

git 已经成功安装进 vbox1 这台 Ubuntu 14.04 的虚机。

如果对 Vagrantfile 进行了编辑，可以使用 vagrant reload 进行重新加载，reload 相当于 halt + up。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Installing and using VirtualBox on CentOS][vb]<br>
2 Mitchell Hashimoto. [Vagrant: Up and Running][uprunning]. O'REILLY. 2013.
</span>

[vagrant]: https://github.com/mitchellh/vagrant
[vb]: https://wiki.centos.org/HowTos/Virtualization/VirtualBox
[uprunning]: https://www.amazon.com/gp/product/1449335837/
[download]: https://www.vagrantup.com/downloads.html
[boxes]: https://atlas.hashicorp.com/boxes/search
[diff]: https://www.quora.com/What-is-the-difference-between-Docker-and-Vagrant-When-should-you-use-each-one
