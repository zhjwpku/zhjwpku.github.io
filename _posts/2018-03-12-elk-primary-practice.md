---
layout: post
title: ELK 初级实践
date: 2018-03-12 20:00:00 +0800
tags:
- elk
---

ELK 集群的安装算是比较简单了，下面列举几个重要的参考点，方便今后查看。

<h4>1. 安装 elasticsearch</h4>

```
## Import the Elasticsearch PGP Key
[root@alice ~]# rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

## Installing from the RPM repository
[root@alice ~]# cat << EOF > /etc/yum.repos.d/elasticsearch.repo
> [elasticsearch-6.x]
> name=Elasticsearch repository for 6.x packages
> baseurl=https://artifacts.elastic.co/packages/6.x/yum
> gpgcheck=1
> gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
> enabled=1
> autorefresh=1
> type=rpm-md
> EOF
[root@alice ~]# yum install elasticsearch
```

这种方式安装的文件分散在多个地方，如下：

```
/usr/share/elasticsearch        Elasticsearch home directory or $ES_HOME
/etc/elasticsearch              Configuration files including elasticsearch.yml
/etc/sysconfig/elasticsearch    Environment variables including heap size, file descriptors
/var/log/elasticsearch          Log files location
```

启动前需要对配置文件进行相应的修改(只列举改动的配置)：

```
[root@alice ~]# cat /etc/elasticsearch/elasticsearch.yml
cluster.name: es-test
node.name: alice
path.data: /home/db1/es-data1,/home/db2/es-data2
path.logs: /var/log/elasticsearch
network.host: 10.0.63.202
discovery.zen.ping.unicast.hosts: ["10.0.63.202", "10.0.63.203", "10.0.63.204"]
...
[root@alice ~]# cat /etc/elasticsearch/jvm.options
-Xms32g
-Xmx32g
...
```

由于使用包管理器安装，因此启动很简单，*systemctl enable/start elasticsearch* 即可。

在多台机器上分别进行相应的配置然后启动，集群之间是自治的，不需进行任何操作。

X-Pack 提供了一些扩展功能的插件，虽然是可选项，但是一般都会进行安装：

```
[root@alice ~]# cd /usr/share/elasticsearch/
[root@alice elasticsearch]# bin/elasticsearch-plugin install x-pack
## 设置三个内置用户的密码，整个集群共享此处设置的密码，不需要在其他 Elasticsearch 节点进行该操作
[root@alice elasticsearch]# bin/x-pack/setup-passwords interative
```

<h4>2. 安装 kibana</h4>

Kibana 是用于数据展示的，而且提供了方便的查询工具。

```
## Installing from the RPM repository
[root@alice ~]# cat << EOF > /etc/yum.repos.d/kibana.repo
> [kibana-6.x]
> name=Kibana repository for 6.x packages
> baseurl=https://artifacts.elastic.co/packages/6.x/yum
> gpgcheck=1
> gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
> enabled=1
> autorefresh=1
> type=rpm-md
> EOF
## 其实直接装就可以可以了，上面的源和 Elasticsearch 的源其实是同一个
[root@alice ~]# yum install kibana
```

目录结构：

```
/usr/share/kibana       Kibana home directory or $KIBANA_HOME
/etc/kibana             Configuration files including kibana.yml
```

安装 X-Pack:

```
[root@alice kibana]# bin/kibana-plugin install x-pack
```

修改 kibana 配置文件：

```
[root@alice kibana]# cat /etc/kibana/kibana.yml
server.port: 5601
# The URL of the Elasticsearch instance to use for all your queries.
## Kibana 目前没有访问多个后台节点的功能，如果此处配置的 elasticsearch 节点失效，Kibana 也会变为不可用
## 一种可行的方式配置多个 Kibana，并对这些 Kibana 做负载均衡方案。
elasticsearch.url: "http://10.0.63.202:9200"

## 该密码是 bin/x-pack/setup-passwords 设置的密码
elasticsearch.username: "kibana"
elasticsearch.password: "elastic"
```

... 未完 ... 

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Install Elasticsearch with RPM](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html)<br>
2 [Install Kibana with RPM](https://www.elastic.co/guide/en/kibana/current/rpm.html)<br>
</span>

