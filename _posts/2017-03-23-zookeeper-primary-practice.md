---
layout: post
title: ZooKeeper 初级实践
date: 2017-03-23 10:00:00 +0800
tags:
- zookeeper
- zkui
---

[ZooKeeper][zookeeper] 是一种分布式协调服务。在分布式环境中协调和管理服务是一个复杂的过程。 ZooKeeper 使用简单的架构和API解决了这个问题。 ZooKeeper 允许开发人员专注于核心应用程序逻辑，而不必担心应用程序的分布式性质。[zkui][zkui] 是一个提供了 ZooKeeper CRUD 操作的 UI。 本文介绍 ZooKeeper 的简单搭建过程及zkui的配置。

使用 Vagrant 创建三台 Zookeeper 虚拟机，三台虚拟机分布在三台物理机，基本信息如下：

|---
| Name | ADDRESS | HOSTNAME
|:-|:-|:-
| zookeeper1 | 172.16.63.101 | zookeeper1
|---
| zookeeper2 | 172.16.63.102 | zookeeper2
|---
| zookeeper3 | 172.16.63.103 | zookeeper3

<h6></h6>
ZooKeeper 的搭建在任意一台机器上的过程都是一样的，这里以 zookeeper1 为例进行说明：

{% highlight shell %}
# 设置 MaxHeapSize 以 Avoid swapping
[vagrant@zookeeper1 ~]$ export _JAVA_OPTIONS="-Xmx3072m"
# 下载安装包
[vagrant@zookeeper1 ~]$ wget http://apache.mirrors.ionfish.org/zookeeper/stable/zookeeper-3.4.9.tar.gz
[vagrant@zookeeper1 ~]$ tar xvf zookeeper-3.4.9.tar.gz
[vagrant@zookeeper1 ~]$ cd zookeeper-3.4.9/
# 配置文件重命名为zoo.cfg
[vagrant@zookeeper1 zookeeper-3.4.9]$ mv conf/zoo_sample.cfg conf/zoo.cfg
# 编辑 conf/zoo.cfg
[vagrant@zookeeper1 zookeeper-3.4.9]$ cat conf/zoo.cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/data/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60

# ZooKeeper集群含三台机器，本机的Server应该配置为IP: 0.0.0.0，这点很重要
server.1=0.0.0.0:2888:3888
server.2=zookeeper2:2888:3888
server.3=zookeeper3:2888:3888
[vagrant@zookeeper1 zookeeper-3.4.9]$ 
{% endhighlight %}

在`/data/zookeeper`下创建一个名为myid的文件，并填写自己的Server ID，zookeeper1 对应 1，zookeeper2 对应 2， zookeeper3 对应 3。

**启动ZooKeeper**

{% highlight shell %}
[vagrant@zookeeper1 zookeeper-3.4.9]$ bin/zkServer.sh start
# 查看日志
[vagrant@zookeeper1 zookeeper-3.4.9]$ tail -f zookeeper.out
{% endhighlight %}

**测试**

{% highlight shell %}
[vagrant@zookeeper1 zookeeper-3.4.9]$ bin/zkCli.sh
Picked up _JAVA_OPTIONS: -Xmx3072m
Connecting to localhost:2181
2017-03-22 07:45:28,716 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2017-03-22 07:45:28,720 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=zookeeper1
2017-03-22 07:45:28,720 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_121
2017-03-22 07:45:28,723 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2017-03-22 07:45:28,723 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.121-0.b13.el7_3.x86_64/jre
2017-03-22 07:45:28,723 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/home/vagrant/zookeeper-3.4.9/bin/../build/classes:/home/vagrant/zookeeper-3.4.9/bin/../build/lib/*.jar:/home/vagrant/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/vagrant/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/home/vagrant/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/home/vagrant/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/home/vagrant/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/home/vagrant/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/home/vagrant/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/home/vagrant/zookeeper-3.4.9/bin/../conf:
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-514.6.2.el7.x86_64
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=vagrant
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/vagrant
2017-03-22 07:45:28,724 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/home/vagrant/zookeeper-3.4.9
2017-03-22 07:45:28,726 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@69d0a921
2017-03-22 07:45:28,769 [myid:] - INFO  [main-SendThread(zookeeper1:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server zookeeper1/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
Welcome to ZooKeeper!
JLine support is enabled
2017-03-22 07:45:28,944 [myid:] - INFO  [main-SendThread(zookeeper1:2181):ClientCnxn$SendThread@876] - Socket connection established to zookeeper1/127.0.0.1:2181, initiating session
2017-03-22 07:45:28,966 [myid:] - INFO  [main-SendThread(zookeeper1:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server zookeeper1/127.0.0.1:2181, sessionid = 0x15af4d71e230001, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
[zk: localhost:2181(CONNECTED) 1] create /hello my_data
Created /hello
[zk: localhost:2181(CONNECTED) 2] ls /
[hello, zookeeper]
[zk: localhost:2181(CONNECTED) 3] get /hello
my_data
cZxid = 0x100000006
ctime = Wed Mar 22 07:47:01 UTC 2017
mZxid = 0x100000006
mtime = Wed Mar 22 07:47:01 UTC 2017
pZxid = 0x100000006
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 7
numChildren = 0
[zk: localhost:2181(CONNECTED) 4] set /hello another_data
cZxid = 0x100000006
ctime = Wed Mar 22 07:47:01 UTC 2017
mZxid = 0x100000007
mtime = Wed Mar 22 07:47:25 UTC 2017
pZxid = 0x100000006
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0
[zk: localhost:2181(CONNECTED) 5] get /hello
another_data
cZxid = 0x100000006
ctime = Wed Mar 22 07:47:01 UTC 2017
mZxid = 0x100000007
mtime = Wed Mar 22 07:47:25 UTC 2017
pZxid = 0x100000006
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0
[zk: localhost:2181(CONNECTED) 6]
{% endhighlight %}

**ZKUI**

ZKUI 到现在我都没有构建成功，只好使用docker镜像来启动应用，比较简单，不多说了，大家知道有这么个工具就行了。


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [ZooKeeper Administrator's Guide][ref1]<br>
2 [Zookeeper error: Cannot open channel to X at election address][ref2]
</span>

[zookeeper]: http://zookeeper.apache.org/
[zkui]: https://github.com/DeemOpen/zkui
[ref1]: http://zookeeper.apache.org/doc/trunk/zookeeperAdmin.html
[ref2]: http://stackoverflow.com/questions/30940981/zookeeper-error-cannot-open-channel-to-x-at-election-address
