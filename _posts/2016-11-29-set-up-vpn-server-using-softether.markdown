---
layout: post
title: 使用SoftEther设置VPN Server
date: 2016-11-29 22:00:00 +0800
tags:
- softether
- vpn
---

[SoftEther][softether](means Software Ethernet) 是日本[筑波大学][tsukuba]的一个[开源跨平台多协议的 VPN][softethergit]，是世界上最强大、易用的多协议VPN软件之一。可以运行在Windows、Linux、Mac、FreeBSD和Solaris上。支持SSL-VPN(HTTPS)及6种主流VPN协议（OpenVPN、IPsec、L2TP、MS-SSTP、L2TPv3和EtherIP）。

![softether_arch](/assets/201611/softether_arch.jpg){: width="770px" style="padding-left: 10px"}

下面介绍如何使用SoftEther搭建VPN Server。

<h4>Step 1: 获取机器</h4>

Amazon AWS提供12个月每月750小时（31.25天）t2.micro实例随意搭配使用，你可以同时开10个Linux连续运行75小时，超过这个期限就要收费了。但搭建VPN只需要一个实例，所以时间足够了。建议选择比较近的DC，如Tokyo、Seoul。

在亚马逊上启动一台实例需要一个keypair，创建keypair时要把key（如tokyo.pem）保存下来，登陆或与虚机传数据都会用到这个key。

{% highlight shell %}
# 使用key来登陆时可能会遇到权限的问题
$ sudo chmod 500 tokyo.pem

# 登陆，instance-ip为实例占有的公网IP
$ sudo ssh -i tokyo.pem <instance-ip>

# 将本地file.txt拷贝到EC2实例
$ sudo scp -i tokyo.pem file.txt ec2-user@<instance-ip>:~/

# 将EC2实例中的数据拷贝到本地
$ sudo scp -i tokyo.pem ec2-user@<instance-ip>:~/remote-file.txt .
{% endhighlight %}

**更新服务器上的软件**

Debian/Ubuntu:
{% highlight shell %}
$ sudo apt-get update && sudo apt-get upgrade

# 编译SoftEther依赖的工具
$ sudo apt-get install build-essential -y
{% endhighlight %}

CentOS/RHEL:
{% highlight shell %}
$ sudo yum upgrade

# 编译SoftEther依赖的工具
$ sudo yum groupinstall "Development Tools"
{% endhighlight %}

<h4>Step 2: 获取SoftEther</h4>

[SoftEther][softether]网站在国内访问不到，又不能在EC2实例上使用浏览器，这时需要用到文本网络浏览器[lynx][lynx]这个利器。

**下载lynx**

Debian/Ubuntu:
{% highlight shell %}
$ sudo apt-get install lynx -y
{% endhighlight %}

CentOS/RHEL:
{% highlight shell %}
# 在AWS RHEL EC2实例上安装lynx需要开启下面的通道
$ sudo yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
$ sudo yum install lynx -y
{% endhighlight %}

**下载softether**

{% highlight shell %}
$ lynx http://www.softether-download.com/files/softether/
{% endhighlight %}

![lynx](/assets/201611/lynx.png)

选择平台
![platform](/assets/201611/platform.png)

选择服务端软件
![vpnserver](/assets/201611/softether_vpn_server.png)

选择机器位数
![64bit](/assets/201611/64bit.png)
![package](/assets/201611/softether-package.png)

下载
![download](/assets/201611/download.png)

SoftEther客户端也可以使用这种方式下载，之后通过前边的scp将其拷贝到本地。

<h4>Step 3: 安装并配置SoftEther</h4>

{% highlight shell %}
# 解压
$ tar xzvf softether-vpn-v4.22-9634-beta-2016.11.24-linux-x64-64bit.tar.gz

# 编译
$ cd vpnserver
$ make
{% endhighlight %}

编译的过程中会多次提示License Agreement的选择，想用就选Yes（type '1'）。
![lisence](/assets/201611/lisence_agreement.png)

**配置**
{% highlight shell %}
# 更改目录位置
$ cd ..
$ mv vpnserver /usr/local
$ cd /usr/local/vpnserver/

# 更改权限
$ sudo chmod 600 *
$ sudo chmod 700 vpnserver
$ sudo chmod 700 vpncmd
{% endhighlight %}

**开机启动**

将SoftEther创建问一个service，并配置开机自动启动。

首先创建一个文件`vim /etc/init.d/vpnserver`，将以下内容粘贴进去：
{% highlight shell %}
#!/bin/sh
# chkconfig: 2345 99 01
# description: SoftEther VPN Server
DAEMON=/usr/local/vpnserver/vpnserver
LOCK=/var/lock/subsys/vpnserver
test -x $DAEMON || exit 0
case "$1" in
start)
    $DAEMON start
    touch $LOCK
    ;;
stop)
    $DAEMON stop
    rm $LOCK
    ;;
restart)
    $DAEMON stop
    sleep 3
    $DAEMON start
    ;;
*)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
esac
exit 0
{% endhighlight %}

如果不存在/var/lock/subsys文件夹，则需要创建一个`mkdir /var/lock/subsys`。

**启动服务**
{% highlight shell %}
# 启动
$ sudo chmod 755 /etc/init.d/vpnserver && /etc/init.d/vpnserver start

# 开机自启动
# Debian/Ubuntu
$ sudo update-rc.d vpnserver defaults

# CentOS/RHEL
$ sudo chkconfig --add vpnserver
{% endhighlight %}

<h4>Step 4: 更改Admin密码</h4>
{% highlight shell %}
$ cd /var/local/vpnserver/
$ ./vpncmd    # 选择1 "Management of VPN Server or VPN Bridge"

VPN Server> ServerPasswordSet
{% endhighlight %}

<h4>Step 5: 创建一个虚拟Hub</h4>

{% highlight shell %}
$ cd /var/local/vpnserver/
$ ./vpncmd    # 选择1 "Management of VPN Server or VPN Bridge"
{% endhighlight %}

![vpncmd](/assets/201611/vpncmd.png)

注意看上图的提示，如果选择连接到某一个Virtual Hub则写入Hub的名称，如果要使用Admin模式连接Server，则直接按Enter键。这一点很重要。创建Hub需要Admin权限。你可以根据提示符是`VPN Server>`还是`VPN Server/VPN>`来区分。
{% highlight shell %}
# 创建一个名为VPN的Hub
VPN Server> HubCreate VPN

# 进入名为VPN的Hub的上下文
VPN Server> Hub VPN
{% endhighlight %}

<h4>Step 6: Enable SecureNAT</h4>

SecureNAT需要绑定到特定的Virtual Hub上，所以这里需要进入Hub的上下文。
{% highlight shell %}
VPN Server/VPN> SecureNatEnable
{% endhighlight %}

<h4>Step 7: 创建用户并设置用户密码</h4>

{% highlight shell %}
VPN Server/VPN> UserCreate zhjwpku
VPN Server/VPN> UserPasswordSet zhjwpku
{% endhighlight %}

![usercreate](/assets/201611/usercreate.png)

<h4>Step 8: Setup L2TP/IPsec</h4>

运行该命令需要Admin Mode。
![ipsec](/assets/201611/IPsecEnable.png)

<h4>Step 9: Setup SSTP/OpenVPN</h4>
SoftEther支持Microsoft SSTP VPN和OpenVPN。
{% highlight shell %}
# 生成自签名证书
VPN Server> ServerCertRegenerate <instance-ip>

# 生成证书，该证书可被导入操作系统或浏览器
VPN Server> ServerCertGet ~/cert.cer

# enable OpenVPN
VPN Server> OpenVpnEnable yes /PORTS:1194

# 获取OpenVPN配置
VPN Server> OpenVpnMakeConfig ~/my_openvpn_config.zip
{% endhighlight %}

<h4>客户端设置</h4>

这里只讲在Windows上使用SoftEther客户端进行配置。安装SoftEther的过程很简单，略。

**创建虚拟适配器**

![adapter](/assets/201611/adapter.gif)

**创建VPN连接**

![connetion](/assets/201611/connection.gif)

另外，可以使用docker的方式来部署SoftEther VPN Server，这个我还没有试过，想玩的请参考[2]。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [How to Setup a Multi-Protocol VPN Server Using SoftEther][howto]<br>
2. [https://github.com/cnf/docker-softether][docker-softether]
</span>

[softether]: http://www.softether.org/
[howto]: https://www.digitalocean.com/community/tutorials/how-to-setup-a-multi-protocol-vpn-server-using-softether
[tsukuba]: http://www.tsukuba.ac.jp/en/
[softethergit]: https://github.com/SoftEtherVPN/SoftEtherVPN
[docker-softether]: https://github.com/cnf/docker-softether
[lynx]: http://lynx.browser.org/
