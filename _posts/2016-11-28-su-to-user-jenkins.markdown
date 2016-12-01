---
layout: post
title: nologin用户登录
date: 2016-11-28 16:50:00 +0800
tags:
- su
- jenkins
- id
---

安装完Jenkins后，Jenkins服务是用Jenkins用户启动的。由于想在Jenkinsfile中配置CD，ssh-agent插件用的又不好，所以想了一个比较trick的方法——在命令行配置无密码登陆，也就是使用`ssh-copy-id`命令。可是用jenkins用户进行登陆的时候出现了问题。

![su-jenkins](/assets/201611/su-jenkins.png)

查看系统所有用户：

{% highlight shell %}
-bash-4.2# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
avahi-autoipd:x:170:170:Avahi IPv4LL Stack:/var/lib/avahi-autoipd:/sbin/nologin
systemd-bus-proxy:x:999:997:systemd Bus Proxy:/:/sbin/nologin
systemd-network:x:998:996:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:997:995:User for polkitd:/:/sbin/nologin
abrt:x:173:173::/etc/abrt:/sbin/nologin
libstoragemgmt:x:996:994:daemon account for libstoragemgmt:/var/run/lsm:/sbin/nologin
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
mysql:x:27:27:MariaDB Server:/var/lib/mysql:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:995:993::/var/lib/chrony:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
ntp:x:38:38::/etc/ntp:/sbin/nologin
tcpdump:x:72:72::/:/sbin/nologin
jenkins:x:994:992:Jenkins Continuous Integration Server:/var/lib/jenkins:/bin/false
zabbix:x:1000:1000::/home/zabbix:/sbin/nologin
{% endhighlight %}

可以看到不同于`root`用户，好多用户条目的最后都有个`/sbin/nologin`命令，`jenkins`用户则为`/bin/false`，这意味着这些用户在设计上没有shell交互，可以如下命令进行login：

![su-s-bin-bash](/assets/201611/su-s-bin-bash.png)

`/etc/passwd`中每个用户的各个域（以`:`隔开）的含义如下：
{% highlight shell %}
operator:x:11:0:operator:/root:/sbin/nologin
   |     |  | |    |       |        |
   1     2  3 4    5       6        7
{% endhighlight %}

1. Username: 用来登陆的ID
2. Password: x说明加密过的密码存储在`/etc/shadow`中
3. User ID (UID): 每个账户必须指定一个UID，UID `0` 为root用户，UID `1-99`用于其它预定账户，UID `100-999`则由系统为管理和系统账户/组保留
4. Group ID (GID): 组ID（存储在/etc/group文件中）
5. Full name of the user
6. User home directory
7. Login shell

可以使用`id`命令来查看账户的uid和gid
{% highlight shell %}
# 不加账户名给出当前登录账户的uid
$ id -u
0
$ id -u operator
11

$ id -G
0
$ id -G operator
0
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [Can't su to user jenkins after installing Jenkins][ref1]
</span>

[ref1]: http://stackoverflow.com/questions/18068358/cant-su-to-user-jenkins-after-installing-jenkins
