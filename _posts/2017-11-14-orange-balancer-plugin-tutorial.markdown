---
layout: post
title: Orange Balancer 安装及使用
date: 2017-11-14 15:00:00 +0800
tags:
- orange
---

部门的架构师通过对比[Zuul][zuul]、[Kong][kong]、[Traefik][traefik]和[Orange][orange]四个网关，最终选定了Orange。但Orange不支持动态配置upstream，需要自己实现（[PR#138][pr138]）。实现的逻辑之后我会在在另一篇博客里记录，本文只介绍该插件的安装及使用方法。

**环境准备**

使用 `Vagrant` 创建一台具有桥接网络地址的 CentOS/7 虚拟机:

![Orange vagrant Environment](/assets/201711/orange_vagrant_env.png)

安装[OpenResty][openresty]:

```
→ ~/work/vagrant/orange $ vagrant ssh       # 之后为虚拟机环境提示符
Last login: Tue Nov 14 08:38:08 2017 from 10.0.2.2
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[vagrant@localhost ~]$ sudo yum install -y yum-utils epel epel-release
[vagrant@localhost ~]$ sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
[vagrant@localhost ~]$ sudo yum install -y openresty
[vagrant@localhost sbin]$ sudo yum install -y openresty-resty
## 创建软链接使得 nginx -v 和 openresty -v 的输出一样
[vagrant@localhost sbin]$ sudo ln -s /usr/local/openresty/nginx/sbin/nginx /usr/bin/nginx
[vagrant@localhost sbin]$ nginx -v
## 通过以下命令可以看出nginx和openresty链接到同一个二进制包
[vagrant@localhost sbin]$ which openresty
/usr/bin/openresty
[vagrant@localhost sbin]$ ls -al /usr/bin/openresty
lrwxrwxrwx. 1 root root 37 Nov 14 14:14 /usr/bin/openresty -> /usr/local/openresty/nginx/sbin/nginx
nginx version: openresty/1.13.6.1
```

安装[lor][lor]:

```
[vagrant@localhost ~]$ git clone https://github.com/sumory/lor
[vagrant@localhost ~]$ cd lor/
[vagrant@localhost lor]$ sudo make install
```

安装MySQL并导入Schema

```
[vagrant@localhost ~]$ yum install -y wget
[vagrant@localhost ~]$ wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
[vagrant@localhost ~]$ sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
[vagrant@localhost ~]$ sudo yum install -y mysql-server

## 启动 MySQL
[vagrant@localhost ~]$ sudo systemctl enable mysqld
[vagrant@localhost ~]$ sudo systemctl start mysqld

## 重置root密码
[vagrant@localhost ~]$ mysql_secure_installation

## 创建数据库orange
[vagrant@localhost ~]$ mysql -uroot -p123456
mysql> CREATE DATABASE orange CHARACTER SET utf8 COLLATE utf8_general_ci;
```

**安装Orange**

因为该该PR还在Review阶段，所以得从我的仓库拉代码。

```
[vagrant@localhost ~]$ sudo yum install -y git luarocks
[vagrant@localhost ~]$ git clone -b add_balancer_by_lua https://github.com/infracc/orange.git

## 导入表结构
[vagrant@localhost orange]$ mysql -uroot -p orange < install/orange-v0.6.4.sql

## 安装依赖包
[vagrant@localhost orange]$ sudo yum install -y lua-devel.x86_64
[vagrant@localhost orange]$ sudo luarocks install Penlight
[vagrant@localhost orange]$ sudo luarocks install lua-resty-dns-client

#下面的命令切换到root用户执行
[vagrant@localhost orange]$ sudo su
[root@localhost orange]# echo "export PATH=$PATH:/usr/local/bin:/usr/local/sbin" >> ~/.bashrc
## 设置LUA_PATH和LUA_CPATH
[root@localhost orange]# echo "export LUA_PATH=/usr/share/lua/5.1/?.lua" >> ~/.bashrc
[root@localhost orange]# echo "export LUA_CPATH=/usr/lib64/lua/5.1/?.so" >> ~/.bashrc
[root@localhost orange]# . ~/.bashrc

## 更改conf/orange.conf.example中Mysql数据库用户名密码

## 初始化&启动
[root@localhost orange]# make install
[root@localhost orange]# orange start
```

注: *在root用户下的命令不要加sudo，因为centos的安全策略可能会导致路径被重写，见 Ref 2*

**使用方法**

在nginx.conf下添加三台测试虚拟机:

```
    # test server 1
    server {
        listen 8881;

        location / {
            content_by_lua_block {
                ngx.say("8881")
            }
        }
    }

    # test server 2
    server {
        listen 8882;

        location / {
            content_by_lua_block {
                ngx.say("8882")
            }
        }
    }

    # test server 3
    server {
        listen 8883;

        location / {
            content_by_lua_block {
                ngx.say("8883")
            }
        }
    }
```

dashboard 添加 upstream 及相应 host:

![Balancer Upstream](/assets/201711/balancer_upstream.gif)

设置divide规则:

![divide rule](/assets/201711/test_balancer.gif)

在浏览器测试:

![Test result](/assets/201711/test_result.gif)


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Orange 安装][orange_install] <br>
2 [command-not-found-when-using-sudo][command-not-found-when-using-sudo]
</span>

[kong]: https://github.com/Kong/kong
[traefik]: https://github.com/containous/traefik
[orange]: https://github.com/sumory/orange
[zuul]: https://github.com/Netflix/zuul
[pr138]: https://github.com/sumory/orange/pull/138
[openresty]: http://openresty.org/en/
[lor]: https://github.com/sumory/lor
[orange_install]: http://orange.sumory.com/install/
[command-not-found-when-using-sudo]: https://superuser.com/questions/709515/command-not-found-when-using-sudo
