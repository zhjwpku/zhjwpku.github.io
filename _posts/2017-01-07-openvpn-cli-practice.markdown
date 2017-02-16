---
layout: post
title: OpenVPN CLI Practice
date: 2017-01-07 09:00:00 +0800
tags:
- openvpn
---

本文记录使用 OpenVPN 建立虚拟通道的命令行实践，不涉及 OpenVPN 原理及图形界面相关的配置。文中的所有操作均在 CentOS 7 上实验通过。

**---Update 2017-01-16---**

推荐一个 OpenVPN 管理工具 [pritunl][pritunl]，对 OpenVPN 进行了封装，提供 Web 访问接口，方便使用。搭建配置都也比较容易，只不过免费版的只支持单台Server，Site-to-Site 不免费。

<h4>安装</h4>

{% highlight shell %}
$ sudo yum install epel-release
# easy-rsa is a CLI utility to build and manage a PKI CA
$ sudo yum install openvpn easy-rsa -y
{% endhighlight %}

<h4>点到点网络 - Shortest Setup</h4>

**使用TUN设备**

服务端启动 OpenVPN 进程，等待连接，注意时间戳。
{% highlight shell %}
[root@anakin ~]# openvpn --ifconfig 10.200.0.1 10.200.0.2 --dev tun
Thu Jan 12 03:50:45 2017 OpenVPN 2.3.14 x86_64-redhat-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Dec  7 2016
Thu Jan 12 03:50:45 2017 library versions: OpenSSL 1.0.1e-fips 11 Feb 2013, LZO 2.06
Thu Jan 12 03:50:45 2017 ******* WARNING *******: all encryption and authentication features disabled -- all data will be tunnelled as cleartext
Thu Jan 12 03:50:45 2017 TUN/TAP device tun0 opened
Thu Jan 12 03:50:45 2017 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan 12 03:50:45 2017 /usr/sbin/ip link set dev tun0 up mtu 1500
Thu Jan 12 03:50:45 2017 /usr/sbin/ip addr add dev tun0 local 10.200.0.1 peer 10.200.0.2
Thu Jan 12 03:50:45 2017 UDPv4 link local (bound): [undef]
Thu Jan 12 03:50:45 2017 UDPv4 link remote: [undef]
## Wait for connection
Thu Jan 12 03:51:58 2017 Peer Connection Initiated with [AF_INET]10.0.63.203:1194
Thu Jan 12 03:51:59 2017 Initialization Sequence Completed
{% endhighlight %}

客户端启动 OpenVPN 进程，连接服务端 10.0.63.202。
{% highlight shell %}
[root@boba ~]# openvpn --ifconfig 10.200.0.2 10.200.0.1 --dev tun --remote 10.0.63.202
Thu Jan 12 03:51:47 2017 OpenVPN 2.3.14 x86_64-redhat-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Dec  7 2016
Thu Jan 12 03:51:47 2017 library versions: OpenSSL 1.0.1e-fips 11 Feb 2013, LZO 2.06
Thu Jan 12 03:51:47 2017 ******* WARNING *******: all encryption and authentication features disabled -- all data will be tunnelled as cleartext
Thu Jan 12 03:51:47 2017 TUN/TAP device tun0 opened
Thu Jan 12 03:51:47 2017 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan 12 03:51:47 2017 /usr/sbin/ip link set dev tun0 up mtu 1500
Thu Jan 12 03:51:47 2017 /usr/sbin/ip addr add dev tun0 local 10.200.0.2 peer 10.200.0.1
Thu Jan 12 03:51:48 2017 UDPv4 link local (bound): [undef]
Thu Jan 12 03:51:48 2017 UDPv4 link remote: [AF_INET]10.0.63.202:1194
Thu Jan 12 03:51:58 2017 Peer Connection Initiated with [AF_INET]10.0.63.202:1194
Thu Jan 12 03:51:59 2017 Initialization Sequence Completed
{% endhighlight %}

连接建立，使用 `ip -a` 可以看到如下网络接口：

{% highlight shell %}
## 服务端
53: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 100
    link/none 
    inet 10.200.0.1 peer 10.200.0.2/32 scope global tun0
        valid_lft forever preferred_lft forever

## 客户端
30: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 100
    link/none 
    inet 10.200.0.2 peer 10.200.0.1/32 scope global tun0
        valid_lft forever preferred_lft forever
{% endhighlight %}

可以使用 `ping` 来测试虚拟 IP 的连通性。

**使用TAP设备**

服务端启动 OpenVPN 进程，等待连接，注意时间戳。
{% highlight shell %}
[root@anakin ~]# openvpn --ifconfig 10.200.0.1 255.255.255.0 --dev tap
Thu Jan 12 04:03:28 2017 OpenVPN 2.3.14 x86_64-redhat-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Dec  7 2016
Thu Jan 12 04:03:28 2017 library versions: OpenSSL 1.0.1e-fips 11 Feb 2013, LZO 2.06
Thu Jan 12 04:03:28 2017 ******* WARNING *******: all encryption and authentication features disabled -- all data will be tunnelled as cleartext
Thu Jan 12 04:03:28 2017 TUN/TAP device tap0 opened
Thu Jan 12 04:03:28 2017 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan 12 04:03:28 2017 /usr/sbin/ip link set dev tap0 up mtu 1500
Thu Jan 12 04:03:28 2017 /usr/sbin/ip addr add dev tap0 10.200.0.1/24 broadcast 10.200.0.255
Thu Jan 12 04:03:28 2017 UDPv4 link local (bound): [undef]
Thu Jan 12 04:03:28 2017 UDPv4 link remote: [undef]
## Wait for connection
Thu Jan 12 04:06:02 2017 Peer Connection Initiated with [AF_INET]10.0.63.203:1194
Thu Jan 12 04:06:03 2017 Initialization Sequence Completed
{% endhighlight %}

客户端启动 OpenVPN 进程，连接服务端 10.0.63.202。
{% highlight shell %}
[root@boba ~]# openvpn --ifconfig 10.200.0.2 255.255.255.0 --dev tap --remote 10.0.63.202
Thu Jan 12 04:06:02 2017 OpenVPN 2.3.14 x86_64-redhat-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Dec  7 2016
Thu Jan 12 04:06:02 2017 library versions: OpenSSL 1.0.1e-fips 11 Feb 2013, LZO 2.06
Thu Jan 12 04:06:02 2017 ******* WARNING *******: all encryption and authentication features disabled -- all data will be tunnelled as cleartext
Thu Jan 12 04:06:02 2017 TUN/TAP device tap0 opened
Thu Jan 12 04:06:02 2017 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan 12 04:06:02 2017 /usr/sbin/ip link set dev tap0 up mtu 1500
Thu Jan 12 04:06:02 2017 /usr/sbin/ip addr add dev tap0 10.200.0.2/24 broadcast 10.200.0.255
Thu Jan 12 04:06:02 2017 UDPv4 link local (bound): [undef]
Thu Jan 12 04:06:02 2017 UDPv4 link remote: [AF_INET]10.0.63.202:1194
Thu Jan 12 04:06:12 2017 Peer Connection Initiated with [AF_INET]10.0.63.202:1194
Thu Jan 12 04:06:13 2017 Initialization Sequence Completed
{% endhighlight %}

连接建立，使用 `ip -a` 可以看到如下网络接口：

{% highlight shell %}
## 服务端
54: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 100
    link/ether 06:d7:04:6d:b4:2c brd ff:ff:ff:ff:ff:ff
    inet 10.200.0.1/24 brd 10.200.0.255 scope global tap0
        valid_lft forever preferred_lft forever
    inet6 fe80::4d7:4ff:fe6d:b42c/64 scope link 
        valid_lft forever preferred_lft forever

## 客户端
31: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 100
    link/ether b6:f7:82:d1:3f:3a brd ff:ff:ff:ff:ff:ff
    inet 10.200.0.2/24 brd 10.200.0.255 scope global tap0
        valid_lft forever preferred_lft forever
    inet6 fe80::b4f7:82ff:fed1:3f3a/64 scope link 
        valid_lft forever preferred_lft forever
{% endhighlight %}

*注：使用如上方式没有任何加密和认证，数据以明文的方式传输。默认使用 UPD 协议进行传输，使用TCP需指明 --proto tcp-server/tcp-client*

<h4>点到点网络 - OpenVPN secret keys</h4>

首先在服务端生成共享密钥，并将该秘钥传输到客户端：

{% highlight shell %}
[root@anakin ~]# openvpn --genkey --secret secret.key
[root@anakin ~]# scp secret.key root@boba:~
{% endhighlight %}

然后启动 OpenVPN:

{% highlight shell %}
## 服务端
[root@anakin ~]# openvpn --ifconfig 10.200.0.1 10.200.0.2 --dev tun --secret secret.key 

## 客户端
[root@boba ~]# openvpn --ifconfig 10.200.0.2 10.200.0.1 --dev tun --secret secret.key --remote 10.0.63.202
{% endhighlight %}

<h4>点到点网络 - Multiple secret keys</h4>

依然使用上面生成的共享密钥，然而使用多种密钥，包括：

- 客户端 Cipher key
- 客户端 HMAC key
- 服务端 Cipher key
- 服务端 HMAC key

{% highlight shell %}
## 服务端
[root@anakin ~]# openvpn --ifconfig 10.200.0.1 10.200.0.2 --dev tun --secret secret.key 0 --verb 7

## 客户端
[root@boba ~]# openvpn --ifconfig 10.200.0.2 10.200.0.1 --dev tun --secret secret.key 1 --remote 10.0.63.202 --verb 7
{% endhighlight %}

<h4>点到点网络 - 路由</h4>

考虑如下网络拓扑图：

![network-toplogy](/assets/201701/network-toplogy.png){: style="padding-left: 87.5px"}

建立 VPN 通道：

{% highlight shell %}
## 服务端
[root@anakin ~]# openvpn --ifconfig 10.200.0.1 10.200.0.2 --dev tun --secret secret.key --daemon --log /var/log/openvpnserver.log

## 客户端
→ ~ # openvpn --ifconfig 10.200.0.2 10.200.0.1 --dev tun --secret secret.key --remote 10.0.63.202 --daemon --log /var/log/openvpnclient.log
{% endhighlight %}

VPN 服务端配置路由：

{% highlight shell %}
## 更改路由规则表，到达子网192.168.126.0/24 的包经过网关 10.200.0.2 
[root@anakin ~]# route add -net 192.168.126.0/24 gw 10.200.0.2
{% endhighlight %}

VPN 客户端配置：

{% highlight shell %}
## 打开ip流量转发功能
→ ~ # sysctl -a | grep ip_forward
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_use_pmtu = 0

## 如果为0则进行如下操作
→ ~ # sysctl -w net.ipv4.ip_forword=1
{% endhighlight %}

Linux Client 上增加路由规则：

{% highlight shell %}
## 更改路由规则表，到达子网10.200.0.0/24 的包经过 192.168.126.151
hunter@debian:~$ sudo route add -net 10.200.0.0/24 gw 192.168.126.151
{% endhighlight %}

这样就能在 10.0.63.202 ping 通 192.168.126.153。

<h4>点到点网络 - 使用配置文件</h4>

**服务端 .ovpn**

{% highlight shell %}
dev tun
proto udp
local openvpnserver.example.com
lport 1194
remote openvpnclient.example.com
rport 1194

secret secret.key 0
ifconfig 10.200.0.1 10.200.0.2
route 192.168.126.0 255.255.255.0

user nobody
group nobody
persist-tun
persist-key
keepalive 10 60
ping-timer-rem

verb 3
daemon
log-append /var/log/openvpnserver.log
{% endhighlight %}

**客户端 .ovpn**

{% highlight shell %}
dev tun
proto udp
local openvpnclient.example.com
lport 1194
remote openvpnserver.example.com
rport 1194

secret secret.key 0
ifconfig 10.200.0.2 10.200.0.1
route 10.0.63.0 255.255.255.0

user nobody
group nobody
persist-tun
persist-key
keepalive 10 60
ping-timer-rem

verb 3
daemon
log-append /var/log/openvpnclient.log
{% endhighlight %}

<h4>配置公钥/私钥</h4>

使用公钥/私钥来进行访问应该是一种更安全的方式，这种方式需要事先配置公钥基础设置（Public Key Infrastructure）。

使用 `easy-rsa` (a handy set of wrapper scripts around some of the openssl ca commands) 提供的脚本来配置 PKI 并生成公钥私钥及 Diffie-Hellman 密钥交换算法配置文件。

{% highlight shell %}
[root@anakin ~]# yum install easy-rsa
[root@anakin ~]# rpm -ql easy-rsa
/usr/share/doc/easy-rsa-2.2.2
/usr/share/doc/easy-rsa-2.2.2/COPYING
/usr/share/doc/easy-rsa-2.2.2/COPYRIGHT.GPL
/usr/share/doc/easy-rsa-2.2.2/doc
/usr/share/doc/easy-rsa-2.2.2/doc/Makefile.am
/usr/share/doc/easy-rsa-2.2.2/doc/README-2.0
/usr/share/easy-rsa
/usr/share/easy-rsa/2.0
/usr/share/easy-rsa/2.0/build-ca
/usr/share/easy-rsa/2.0/build-dh
/usr/share/easy-rsa/2.0/build-inter
/usr/share/easy-rsa/2.0/build-key
/usr/share/easy-rsa/2.0/build-key-pass
/usr/share/easy-rsa/2.0/build-key-pkcs12
/usr/share/easy-rsa/2.0/build-key-server
/usr/share/easy-rsa/2.0/build-req
/usr/share/easy-rsa/2.0/build-req-pass
/usr/share/easy-rsa/2.0/clean-all
/usr/share/easy-rsa/2.0/inherit-inter
/usr/share/easy-rsa/2.0/list-crl
/usr/share/easy-rsa/2.0/openssl-0.9.6.cnf
/usr/share/easy-rsa/2.0/openssl-0.9.8.cnf
/usr/share/easy-rsa/2.0/openssl-1.0.0.cnf
/usr/share/easy-rsa/2.0/pkitool
/usr/share/easy-rsa/2.0/revoke-full
/usr/share/easy-rsa/2.0/sign-req
/usr/share/easy-rsa/2.0/vars
/usr/share/easy-rsa/2.0/whichopensslcnf
[root@anakin ~]# mkdir -m 700 -p /etc/openvpn/openvpn-ca/keys
[root@anakin ~]# cd /etc/openvpn/openvpn-ca/
[root@anakin openvpn-ca]# cp -drp /usr/share/easy-rsa/2.0/* .
[root@anakin openvpn-ca]# ls
build-ca     build-key         build-key-server  clean-all      openssl-0.9.6.cnf  pkitool      vars
build-dh     build-key-pass    build-req         inherit-inter  openssl-0.9.8.cnf  revoke-full  whichopensslcnf
build-inter  build-key-pkcs12  build-req-pass    list-crl       openssl-1.0.0.cnf  sign-req
## 编辑 vars 文件中与使用者相关及密钥长度相关的信息
[root@anakin openvpn-ca]# vim vars
[root@anakin openvpn-ca]# . ./vars
NOTE: If you run ./clean-all, I will be doing a rm -rf on /etc/openvpn/openvpn-ca/keys
[root@anakin openvpn-ca]# ./clean-all
## 注意，这里生成了 keys 文件夹，用来存放PKI相关的密钥
[root@anakin openvpn-ca]# ls
build-ca     build-key         build-key-server  clean-all      list-crl           openssl-1.0.0.cnf  sign-req
build-dh     build-key-pass    build-req         inherit-inter  openssl-0.9.6.cnf  pkitool            vars
build-inter  build-key-pkcs12  build-req-pass    keys           openssl-0.9.8.cnf  revoke-full        whichopensslcnf

ot@anakin openvpn-ca]# KEY_SIZE=4096 ./build-ca --pass
Generating a 4096 bit RSA private key
.....++
..........................................++
writing new private key to 'ca.key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name (full name) [CA]:
Locality Name (eg, city) [SanFrancisco]:
Organization Name (eg, company) [Fort-Funston]:
Organizational Unit Name (eg, section) [MyOrganizationalUnit]:
Common Name (eg, your name or your server's hostname) [Fort-Funston CA]:
Name [EasyRSA]:
Email Address [me@myhost.mydomain]:
## 增加了 ca.crt 和 ca.key
[root@anakin openvpn-ca]# ls keys/
ca.crt  ca.key  index.txt  serial

{% endhighlight %}

**生成服务端证书**

{% highlight shell %}
[root@anakin openvpn-ca]# ./build-key-server openvpnserver
Generating a 2048 bit RSA private key
...........................................................................................................................................................................+++
..................+++
writing new private key to 'openvpnserver.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name (full name) [CA]:
Locality Name (eg, city) [SanFrancisco]:
Organization Name (eg, company) [Fort-Funston]:
Organizational Unit Name (eg, section) [MyOrganizationalUnit]:
Common Name (eg, your name or your server's hostname) [openvpnserver]:
Name [EasyRSA]:
Email Address [me@myhost.mydomain]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /etc/openvpn/openvpn-ca/openssl-1.0.0.cnf
Enter pass phrase for /etc/openvpn/openvpn-ca/keys/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'US'
stateOrProvinceName   :PRINTABLE:'CA'
localityName          :PRINTABLE:'SanFrancisco'
organizationName      :PRINTABLE:'Fort-Funston'
organizationalUnitName:PRINTABLE:'MyOrganizationalUnit'
commonName            :PRINTABLE:'openvpnserver'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'me@myhost.mydomain'
Certificate is to be certified until Jan 14 02:22:12 2027 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
## Notice openvpnserver.crt openvpnserver.key
[root@anakin openvpn-ca]# ls keys/
01.pem  ca.key     index.txt.attr  openvpnserver.crt  openvpnserver.key  serial.old
ca.crt  index.txt  index.txt.old   openvpnserver.csr  serial
{% endhighlight %}

**生成客户端使用的不带密码的证书**

{% highlight shell %}
[root@anakin openvpn-ca]# ./build-key --batch openvpnclient1
Generating a 2048 bit RSA private key
............................................................................+++
..........................................................................................................................................................+++
writing new private key to 'openvpnclient1.key'
-----
Using configuration from /etc/openvpn/openvpn-ca/openssl-1.0.0.cnf
Enter pass phrase for /etc/openvpn/openvpn-ca/keys/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'US'
stateOrProvinceName   :PRINTABLE:'CA'
localityName          :PRINTABLE:'SanFrancisco'
organizationName      :PRINTABLE:'Fort-Funston'
organizationalUnitName:PRINTABLE:'MyOrganizationalUnit'
commonName            :PRINTABLE:'openvpnclient1'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'me@myhost.mydomain'
Certificate is to be certified until Jan 14 02:33:41 2027 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
## Notice openvpnclient1.crt openvpnclient1.key
[root@anakin openvpn-ca]# ls keys/
01.pem  ca.crt  index.txt       index.txt.attr.old  openvpnclient1.crt  openvpnclient1.key  openvpnserver.csr  serial
02.pem  ca.key  index.txt.attr  index.txt.old       openvpnclient1.csr  openvpnserver.crt   openvpnserver.key  serial.old
{% endhighlight %}

**生成客户端使用的带密码 Pass Phrase 的证书**
{% highlight shell %}
[root@anakin openvpn-ca]# ./build-key-pass openvpnclient2
Generating a 2048 bit RSA private key
...........................+++
......................................+++
writing new private key to 'openvpnclient2.key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name (full name) [CA]:
Locality Name (eg, city) [SanFrancisco]:
Organization Name (eg, company) [Fort-Funston]:
Organizational Unit Name (eg, section) [MyOrganizationalUnit]:
Common Name (eg, your name or your server's hostname) [openvpnclient2]:
Name [EasyRSA]:
Email Address [me@myhost.mydomain]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /etc/openvpn/openvpn-ca/openssl-1.0.0.cnf
Enter pass phrase for /etc/openvpn/openvpn-ca/keys/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'US'
stateOrProvinceName   :PRINTABLE:'CA'
localityName          :PRINTABLE:'SanFrancisco'
organizationName      :PRINTABLE:'Fort-Funston'
organizationalUnitName:PRINTABLE:'MyOrganizationalUnit'
commonName            :PRINTABLE:'openvpnclient2'
name                  :PRINTABLE:'EasyRSA'
emailAddress          :IA5STRING:'me@myhost.mydomain'
Certificate is to be certified until Jan 14 02:38:36 2027 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
## Notice openvpnclient2.crt openvpnclient2.key
[root@anakin openvpn-ca]# ls keys
01.pem  ca.crt     index.txt.attr      openvpnclient1.crt  openvpnclient2.crt  openvpnserver.crt  serial
02.pem  ca.key     index.txt.attr.old  openvpnclient1.csr  openvpnclient2.csr  openvpnserver.csr  serial.old
03.pem  index.txt  index.txt.old       openvpnclient1.key  openvpnclient2.key  openvpnserver.key
{% endhighlight %}

**Diffie-Hellman**

{% highlight shell %}
[root@anakin openvpn-ca]# ./build-dh 
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
..................................................................................................................................................+.................................................................+............................................................++*++*
## Notice dh2048.pem
[root@anakin openvpn-ca]# ls keys/
01.pem  ca.crt      index.txt           index.txt.old       openvpnclient1.key  openvpnclient2.key  openvpnserver.key
02.pem  ca.key      index.txt.attr      openvpnclient1.crt  openvpnclient2.crt  openvpnserver.crt   serial
03.pem  dh2048.pem  index.txt.attr.old  openvpnclient1.csr  openvpnclient2.csr  openvpnserver.csr   serial.old
{% endhighlight %}

**tls-auth key**

{% highlight shell %}
[root@anakin openvpn-ca]# openvpn --genkey --secret ta.key
{% endhighlight %}

<h4>Server-side routing</h4>

OpenVPN 客户端能访问到所有 OpenVPN 服务器后端的机器。

**服务端配置**

{% highlight shell %}
dev tun
proto udp
port 1194
server 192.168.200.0 255.255.255.0

ca ./ca.crt
cert ./openvpnserver.crt
key ./openvpnserver.key
dh ./dh2048.pem
tls-auth ./ta.key 0

persist-key
persist-tun
keepalive 10 60

push "route 10.0.0.0 255.255.0.0"
toplogy subnet

user nobody
user nobody
daemon
log-append /var/log/openvpn.log
{% endhighlight %}

**客户端配置**

{% highlight shell %}
client
proto udp
remote 10.0.63.202
port 1194
dev tun
nobind

verb 12
ca ./ca.crt
cert ./openvpnclient1.crt
key ./openvpnclient1.key
tls-auth ./ta.key 1

ns-cert-type server
{% endhighlight %}

另外，还需要在 Server 端的网关添加路由规则，使得所有 VPN 的流量能返回到 VPN Server。

{% highlight shell %}
[gateway]> ip route add 192.168.200.0/24 via 10.0.63.202
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 Markus Feilner, Norbert Graf. [Beginning OpenVPN 2.0.9](/assets/pdf/BeginningOpenVPN2.0.9.pdf). PACKT PUBLISHING. 2009.<br>
2 Jan Just Keijser. [OpenVPN 2 Cookbook](/assets/pdf/openvpn-2-cookbook.pdf). PACKT PUBLISHING. 2011.
</span>

[pritunl]: https://github.com/pritunl/pritunl
