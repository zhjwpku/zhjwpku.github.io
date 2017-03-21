---
layout: post
title: Bind 初级实践
date: 2017-03-21 18:00:00 +0800
tags:
- bind
---

在局域网中部署一个域名服务器能够让开发者免于脑记各种IP地址，本文介绍使用 [BIND (Berkeley Internet Name Domain)][bind] 部署一个简单的DNS Server。

<h4>Vagrant 运行BIND运行的虚拟机</h4>

**Vagrantfile:**

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "centos/7"
  config.vm.hostname = "bind"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # 暴露局域网可访问的IP
  config.vm.network "public_network", ip: "172.16.63.251"

  config.vm.synced_folder "data", "/data"
end
{% endhighlight %}

<h4>Install BIND & Configure</h4>

{% highlight shell %}
$ sudo yum install bind bind-utils
{% endhighlight %}

**/etc/named.conf**
{% highlight shell %}
// See the BIND Administrators Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

acl "trusted" {
    10.0.63.0/24;
	172.16.63.0/24;
};

options {
	listen-on port 53 { 127.0.0.1; 172.16.63.251; };
	# listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-transfer { trusted; };
	allow-query     { trusted; };

	/*
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable
	   recursion.
	 - If your recursive DNS server has a public IP address, you MUST enable access
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface
	*/
	recursion yes;

	dnssec-enable yes;
	dnssec-validation yes;
	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
include "/etc/named/named.conf.local";
{% endhighlight %}

**/etc/named/named.conf.local**

{% highlight shell %}
zone "bind.com" {
	type master;
	file "/etc/named/zones/db.bind.com";
};

zone "63.16.172.in-addr.arpa" {
	type master;
	file "/etc/named/zones/db.172.16.63";
};

zone "63.0.10.in-addr.arpa" {
	type master;
	file "/etc/named/zones/db.10.0.63";
};
{% endhighlight %}

**/etc/named/zones/db.bind.com** 用于正向解析 - 域名 => IP

{% highlight shell %}
$TTL 604800
@       IN      SOA     ns1.bind.com. admin.bind.com. (
              3         ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
;
; name servers - NS records
    IN      NS      ns1.bind.com.

; name servers - A records
ns1.bind.com.		IN	A	172.16.63.251

; 10.0.63.0/24 - A records
harbor.bind.com.	IN	A	10.0.63.205
alice.bind.com.		IN	A	10.0.63.202
{% endhighlight %}

**/etc/named/zones/db.172.16.63** 用于反向解析 - IP => 域名

{% highlight shell %}
$TTL 604800
@       IN      SOA     ns1.bind.com. admin.bind.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
      IN      NS      ns1.bind.com.

; PTR Records
251   IN      PTR     ns1.bind.com.    ; 172.16.63.251
{% endhighlight %}

**/etc/named/zones/db.10.0.63** 用于反向解析

{% highlight shell %}
$TTL 604800
@       IN      SOA     ns1.bind.com. admin.bind.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
      IN      NS      ns1.bind.com.

; PTR Records
205   IN      PTR     harbor.bind.com.  ; 10.0.63.205
202   IN      PTR     alice.bind.com. 	; 10.0.63.202
{% endhighlight %}

{% highlight shell %}
# 启动bind
$ sudo systemctl enable named.service
$ sudo systemctl start named.service
# 如果对配置文件进行了更改使用如下命令让其生效
$ sudo systemctl reload named.service
{% endhighlight %}

<h4>客户端配置及测试</h4>

**/etc/resolv.conf**

{% highlight shell %}
search bind.com
nameserver 172.16.63.251
{% endhighlight %}

**安装 nslookup/dig:**

{% highlight shell %}
$ sudo yum install bind-utils
{% endhighlight %}

**正向解析**

{% highlight shell %}
# 根据 /etc/resolv.conf 中的 search 域将 alice 扩展为 alice.bind.com
[vagrant@swagger ~]$ nslookup alice
Server:		172.16.63.251
Address:	172.16.63.251#53

Name:	alice.bind.com
Address: 10.0.63.202
{% endhighlight %}

**反向解析**

{% highlight shell %}
[vagrant@swagger ~]$ nslookup 10.0.63.205
Server:		172.16.63.251
Address:	172.16.63.251#53

205.63.0.10.in-addr.arpa	name = harbor.bind.com.
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [How To Configure BIND as a Private Network DNS Server on CentOS 7][ref1]<br>
2 [http://ftp.isc.org/isc/bind9/cur/9.9/doc/arm/Bv9ARM.html][ref2]
</span>

[bind]: https://www.isc.org/downloads/bind/
[ref1]: https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-centos-7
[ref2]: http://ftp.isc.org/isc/bind9/cur/9.9/doc/arm/Bv9ARM.html
