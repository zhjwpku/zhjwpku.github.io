---
layout: post
title: Ceph环境搭建【CentOS】
date: 2016-01-13 11:17:00 +0800
categories: ceph
tags:
- ceph
---

从某种程度上该文章是ceph.com[快速安装](http://docs.ceph.com/docs/master/start/)文档的翻译，不过在跑不通
的情况下我会以自己的方法进行处理。本文的所有内容均在CentOS环境下运行。

<h4>架构</h4>

建议配置一个`ceph-deploy` admin节点和三个Ceph Storage Cluster节点来探索ceph。如下图所示：

![ceph-deploy](http://img.blog.csdn.net/20160113090921137)

使用VMware WorkStation来创建4个虚拟机，我用的是CentOS 7作为安装镜像，安装过程我就不赘述了。分别将主机名命名为admin-node, node1, node2, node3, 如果安装时未设置hostname，可以参照[CentOS 7修改主机名【hostnamectl】](http://blog.csdn.net/zhjwpku/article/details/50506760)来配置。

为了方便主机间使用hostname进行通信，对每台机器的hosts进行配置，编辑/etc/hosts，在末尾添加：

{% highlight shell %}
192.168.126.141 admin-node
192.168.126.142 node1
192.168.126.143 node2
192.168.126.144 node3
{% endhighlight %}

上述操作后，各主机之间就可以通过hostname进行通信了。可以使用ping操作进行验证。如`ping node1`。

*注，上述IP应该根据自己虚拟机的IP地址进行更改。另外如果虚机使用的是无线，应该把网路链接设置为NAT模式，若使用的有线网卡则两种模式都可以。*

<h4>CEPH DEPLOY SETUP</h4>

在**admin-node**上添加Ceph安装库，然后安装`ceph-deploy`

{% highlight shell %}
$sudo vim /etc/yum.repos.d/ceph.repo
#添加如下内容：
[ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-hammer/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
#然后运行：
$sudo yum update && sudo yum install ceph-deploy
{% endhighlight %}

*注：在后面的操作中会出现[ceph_deploy][ERROR ] RuntimeError: NoSectionError: No section: 'Ceph'这样的错误提示，可以将ceph.repo改名为ceph-deploy.repo来解决这个问题，我是在http://www.virtualtothecore.com/en/adventures-with-ceph-storage-part-5-install-ceph-in-the-lab/ 中看到评论里边有人这样做（搜Ben Town），尝试了一下真的OK。*

<h4>CEPH NODE SETUP</h4>

admin-node必须对Ceph节点有免输密码（password-less）的SSH权限。当ceph-deploy使用一个用户登陆进入一个Ceph节点，该用户必须拥有sudo权限。

**安装NTP**

防止时间漂移，提供时钟同步，在所有node节点上输入：
`#sudo yum install ntp ntpdate ntpp-doc`

**安装SSH SERVER**

CentOS默认是安装过openssh-server的，如果没有安装，输入：
{% highlight shell %}
$sudo vim install openssh-server
$systemctl start sshd.service //启动sshd服务
$systemctl enble sshd.service //开机自启动sshd服务
{% endhighlight %}

**创建一个Ceph Deploy User**

`ceph-deploy`必须登录进入一个Ceph节点并拥有免输密码的sudo权限。虽然并不建议使用root，但由于只是做实验，并不是生产环境，我还是用了root，如果不想使用root，参照http://docs.ceph.com/docs/master/start/quick-start-preflight/#create-a-ceph-deploy-user 来设置用户并更改其权限。

**PASSWORD-LESS SSH**

由于`ceph-deploy`不会提示输入密码，所以必须在admin-node上生成SSH keys并分发到每个Ceph节点上。

{% highlight shell %}
$sudo ssh-keygen //生成SSH keys
$sudo ssh-copy-id root@node1
$sudo ssh-copy-id root@node2
$sudo ssh-copy-id root@node3
{% endhighlight %}

**防火墙设置**

Ceph Monitors通过6789端口通信，Ceph OSDs通过6800:7300端口范围进行通信。
在使用**firewall**的RHEL7中，使用如下命令打开Monitor(node1)的6789端口以及OSDs(node2, node3)的6800:7300端口:

{% highlight shell %}
$sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent
##或使用iptables命令：
$sudo iptables -A INPUT {iface} -p tcp -s {ip-address}/{netmask} --dport 6789 -j ACCEPT
##由于我对这些命令并不熟悉，所以干脆一点，把firewall禁掉：
$sudo systemctl stop firewalld.service
$sudo systemctl disable firewalld.service
{% endhighlight %}

**TTY**

在Ceph nodes上将`Defaults requiretty` 设置为`Defaults:ceph requiretty`，使用sudo visudo定位并更改。

**SELINUX**

`sudo setenfore 0`

**还需要安装一些包，`yum-plugin-priorites`, `epel-release`，在各节点使用yum安装就可以**

<h4>Create A Cluster</h4>

1 之后的命令只需要在admin-node上运行就可以了

{% highlight shell %}
$mkdir my-cluster
$cd my-cluster
{% endhighlight %}

2 如果在任何时刻出错了并且想重新来一遍，应该先运行如下命令：

{% highlight shell %}
$sudo ceph-deploy purgedata node1, node2, node3
$sudo ceph-deploy forgetkeys
$sudo ceph-deploy purge node1 node2 node3 //把ceph安装包也清理掉
{% endhighlight %}

3 创建cluster:

{% highlight shell %}
## 该命令将node1初始化为Monitor节点，同时生成Ceph configure file，也就是Ceph的配置文件。
$ceph-deploy new node1 //initial-monitor-node(s)
{% endhighlight %}

4 编辑ceph.conf，在[global] section下添加一行

osd pool default size 2
默认的replicas为3，改为2可以使ceph仅有两个OSD的情况下达到`active+clean`的状态。

5 安装Ceph:

{% highlight shell %}
## 这步会出现之前提到的NoSectionError: No section: 'Ceph'，按照上文说的，在admin-node中把/etc/yum.repos.d/下ceph.repo改为ceph-deploy.repo即可。
## ceph-deploy会在各节点上安装Ceph，在/etc/yum.repos.d/中会出现ceph.repo。
$ceph-deploy install admin-node node1 node2 node3
{% endhighlight %}

6 添加初始monitor并收集keys

{% highlight shell %}
$ceph-deploy mon create-initial
{% endhighlight %}

7 添加两个OSDs

{% highlight shell %}
$ssh node2
$sudo mkdir /var/local/osd0
$sudo chmod 777 /var/local/osd0  //防止后文osd activate操作被拒绝
$exit
 
$ssh node3
$sudo mkdir /var/local/osd1
$sudo chmod 777 /var/local/osd1
$exit
 
$ceph-deploy osd prepare node2:/var/local/osd0 node3:/var/local/osd0
$ceph-deploy osd activate node2:/var/local/osd0 node3:/var/local/osd1
{% endhighlight %}

8 使用ceph-deploy将configure file和admin key拷贝到admin节点和Ceph节点上

{% highlight shell %}
$ceph-deploy admin admin-node node1 node2 node3
{% endhighlight %}

9 确保对ceph.client.admin.keyring有正确的权限

{% highlight shell %}
$sudo chmod +r /etc/ceph/ceph.client.admin.keyring
{% endhighlight %}

<h6></h6>
至此，这个四个节点的Ceph环境就搭好了。
检查cluster的健康及cluster状态：
{% highlight shell %}
$sudo chmod +r /etc/ceph/ceph.client.admin.keyring
$ceph health
$ceph status
{% endhighlight %}

*补充：*
如果在实验过程中所有机器都关机了，那么重启之后Ceph存储默认不启动的，如果ceph status显示集群错误，需要做的工作：

{% highlight shell %}
1. 把各个node的IP设置为与之前一样，因为DHCP可能更改机器IP
2. $ceph-deploy mon create-initial
3. $ceph-deploy osd activate node2:/var/local/osd0  node3:/var/local/osd1
{% endhighlight %}
