---
layout: post
title: 第二次安装 Ceph
date: 2017-05-07 22:00:00 +0800
categories: ceph
tags:
- ceph
---

手头有三台 R930 服务器，配置还可以，40核CPU，256G内存，三块300G磁盘，两块做RAID1，其中一块替补。两块1.2T硬盘做OSD（本来三块的，后来被人借走一块），总而言之，做实验够了。

R930

![R930](/assets/201705/R930.jpeg){: width="400px"}

**机器 hostname & IP**

| Name | ADDRESS | HOSTNAME
|:-:|:-:|:-:
| ceph-1 | 10.0.63.202/172.16.111.202 | alice
| ceph-2 | 10.0.63.203/172.16.111.203 | bob
| ceph-3 | 10.0.63.204/172.16.111.204 | carol

<h6></h6>

<h3>准备阶段</h3>

<h4>更新源</h4>

将三台机器的源更改为阿里云的源以加快下载速度。

```shell
## backup
[root@alice ~]# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.ori
## Use alibaba centos repo
[root@alice ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@alice ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
## 阿里云实例使用 aliyuncs 源可以免下载流量，这里不需要，将其删除
[root@alice ~]# sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
[root@alice ~]# sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo
## 下载 repo 的 metadata
[root@alice ~]# yum makecache
```

**Ceph 源（Aliyun）**

`vim /etc/yum.repos.d/ceph.repo`，添加：

```shell
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
```

运行 `yum makecache`。

<h4>安装ntp</h4>

```
[root@alice ~]# yum install ntp -y
[root@alice ~]# systemctl start ntpd
[root@alice ~]# systemctl enable ntpd.service
```

<h4>安装ceph-deploy</h4>

在 ceph-1 节点安装 ceph-deploy 并设置对其他机器的无密码登录。

```shell
## 安装 ceph-deploy
[root@alice ~]# yum install ceph-deploy
[root@alice ~]# ceph-deploy --version
1.5.36
```

配置 /etc/hosts, 添加：

```
10.0.63.202 alice
10.0.63.203 bob
10.0.63.204 carol
```

配置免密码登录：

```shell
[root@alice ~]# ssh-keygen
[root@alice ~]# ssh-copy-id root@bob
[root@alice ~]# ssh-copy-id root@carol
```

<h4>关闭selinux和firewalld</h4>

*这是一种极其偷懒的做法，笔者非常不建议*😀

```
[root@alice ~]# sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
[root@alice ~]# setenforce 0
[root@alice ~]# systemctl stop firewalld
[root@alice ~]# systemctl disable firewalld
```

<h3>安装部署</h3>

以下操作全部在 ceph-1 节点进行。所谓的 ceph-deploy 节点用于存储ceph的配置文件和各种秘钥，因此可以选择任一机器来操作。

```shell
## 创建 ceph cluster 目录
[root@alice ~]# mkdir ceph-cluster && cd ceph-cluster
## 创建集群
[root@alice ceph-cluster]# ceph-deploy new node1 node2 node3
[root@alice ceph-cluster]# ls
ceph-deploy-ceph.log  ceph.conf  ceph.mon.keyring
## 编辑 ceph-cluster 目录下的 ceph.conf
[root@alice ceph-cluster]# vim ceph.conf
[global]
fsid = c9b8862d-6943-4bb5-8785-20b0a2c694b8
mon_initial_members = alice, bob, carol
mon_host = 10.0.63.202,10.0.63.203,10.0.63.204
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
# change default replica 3 to 2
osd pool default size = 2

public network = 10.0.63.0/24
cluster network = 172.16.111.0/24
## 安装ceph
[root@alice ceph-cluster]# ceph-deploy install bob carol
## Deploy monitors and then gatherkeys
[root@alice ceph-cluster]# ceph-deploy mon create-initial
[root@alice ceph-cluster]# ll
total 136
-rw-r--r--. 1 root root 110672 May 12 08:47 ceph-deploy-ceph.log
-rw-------. 1 root root    113 May 12 08:47 ceph.bootstrap-mds.keyring
-rw-------. 1 root root    113 May 12 08:47 ceph.bootstrap-osd.keyring
-rw-------. 1 root root    113 May 12 08:47 ceph.bootstrap-rgw.keyring
-rw-------. 1 root root    129 May 12 08:47 ceph.client.admin.keyring
-rw-r--r--. 1 root root    353 May 12 08:37 ceph.conf
-rw-------. 1 root root     73 May 12 08:36 ceph.mon.keyring
```

查看集群状态：

```shell
[root@alice ceph-cluster]# ceph -s
    cluster c9b8862d-6943-4bb5-8785-20b0a2c694b8
     health HEALTH_ERR
            64 pgs are stuck inactive for more than 300 seconds
            64 pgs stuck inactive
            no osds
     monmap e1: 3 mons at {alice=10.0.63.202:6789/0,bob=10.0.63.203:6789/0,carol=10.0.63.204:6789/0}
            election epoch 14, quorum 0,1,2 alice,bob,carol
     osdmap e1: 0 osds: 0 up, 0 in
            flags sortbitwise
      pgmap v2: 64 pgs, 1 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
                  64 creating
```

部署 OSD

```shell
[root@alice ceph-cluster]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   1.1T  0 disk
sdb           8:16   0   1.1T  0 disk
sdc           8:32   0 278.9G  0 disk
├─sdc1        8:33   0     1G  0 part /boot
└─sdc2        8:34   0 277.9G  0 part
  ├─cl-root 253:0    0    50G  0 lvm  /
  ├─cl-swap 253:1    0     4G  0 lvm  [SWAP]
  └─cl-home 253:2    0 223.9G  0 lvm  /home
sr0          11:0    1  1024M  0 rom

## Zap disks
[root@alice ceph-cluster]# ceph-deploy disk zap alice:sda alice:sdb bob:sda bob:sdb carol:sda carol:sdb
## Prepare OSDs
[root@alice ceph-cluster]# ceph-deploy osd prepare alice:sda alice:sdb bob:sda bob:sdb carol:sda carol:sdb
## 以上命令会将硬盘分为两个分区
[root@alice ceph-cluster]# fdisk -l
...
Disk /dev/sda: 1200.2 GB, 1200243695616 bytes, 2344225968 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: gpt


#         Start          End    Size  Type            Name
 1     10487808   2344225934    1.1T  unknown         ceph data
 2         2048     10487807      5G  unknown         ceph journal
...
## 通过 mount 命令查看分区挂载的位置
[root@alice ceph-cluster]# mount
...
/dev/sda1 on /var/lib/ceph/osd/ceph-0 type xfs (rw,noatime,attr2,inode64,noquota)
/dev/sdb1 on /var/lib/ceph/osd/ceph-1 type xfs (rw,noatime,attr2,inode64,noquota)
## 启动 OSD
[root@alice ceph-cluster]# ceph-deploy osd activate \
alice:/var/lib/ceph/osd/ceph-0 alice:/var/lib/ceph/osd/ceph-1 \
bob:/var/lib/ceph/osd/ceph-2 bob:/var/lib/ceph/osd/ceph-3 \
carol:/var/lib/ceph/osd/ceph-4 carol:/var/lib/ceph/osd/ceph-5
[root@alice ceph-cluster]# ceph -s
    cluster c9b8862d-6943-4bb5-8785-20b0a2c694b8
     health HEALTH_WARN
            too few PGs per OSD (21 < min 30)
     monmap e1: 3 mons at {alice=10.0.63.202:6789/0,bob=10.0.63.203:6789/0,carol=10.0.63.204:6789/0}
            election epoch 18, quorum 0,1,2 alice,bob,carol
     osdmap e31: 6 osds: 6 up, 6 in
            flags sortbitwise
      pgmap v93: 64 pgs, 1 pools, 0 bytes data, 0 objects
            201 MB used, 6673 GB / 6673 GB avail
                  64 active+clean

[root@alice ceph-cluster]# ceph osd tree
ID WEIGHT  TYPE NAME      UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 6.51718 root default
-2 2.17239     host alice
 0 1.08620         osd.0       up  1.00000          1.00000
 1 1.08620         osd.1       up  1.00000          1.00000
-3 2.17239     host bob
 2 1.08620         osd.2       up  1.00000          1.00000
 3 1.08620         osd.3       up  1.00000          1.00000
-4 2.17239     host carol
 4 1.08620         osd.4       up  1.00000          1.00000
 5 1.08620         osd.5       up  1.00000          1.00000
```

从 `ceph -s` 可以看出，集群的状态有 WARNING，原因在于一个OSD对应的 placement group 太少，这个没关系，当创建一个新的 pool 的时候回创建更多的 PG.

```shell
[root@alice ceph-cluster]# ceph osd pool create s3 64 64 replicated
pool 's3' created
[root@alice ceph-cluster]# ceph osd lspools
0 rbd,1 s3,
[root@alice ceph-cluster]# ceph -s
    cluster c9b8862d-6943-4bb5-8785-20b0a2c694b8
     health HEALTH_WARN
            8 pgs peering
     monmap e1: 3 mons at {alice=10.0.63.202:6789/0,bob=10.0.63.203:6789/0,carol=10.0.63.204:6789/0}
            election epoch 18, quorum 0,1,2 alice,bob,carol
     osdmap e33: 6 osds: 6 up, 6 in
            flags sortbitwise
      pgmap v97: 128 pgs, 2 pools, 0 bytes data, 0 objects
            201 MB used, 6673 GB / 6673 GB avail
                 104 active+clean
                  16 creating
                   8 creating+peering
[root@alice ceph-cluster]# ceph -s
    cluster c9b8862d-6943-4bb5-8785-20b0a2c694b8
     health HEALTH_OK
     monmap e1: 3 mons at {alice=10.0.63.202:6789/0,bob=10.0.63.203:6789/0,carol=10.0.63.204:6789/0}
            election epoch 18, quorum 0,1,2 alice,bob,carol
     osdmap e33: 6 osds: 6 up, 6 in
            flags sortbitwise
      pgmap v103: 128 pgs, 2 pools, 0 bytes data, 0 objects
            202 MB used, 6673 GB / 6673 GB avail
                 128 active+clean
```

Ceph 集群搭建完成。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [ADD/REMOVE OSDS][addosds]<br>
2 [POOL, PG AND CRUSH CONFIG REFERENCE][ref2]<br>
3 [CEPH部署完整版(el7+jewel)][ref3]
</span>

[addosds]: http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-osd/
[ref2]: http://docs.ceph.com/docs/master/rados/configuration/pool-pg-config-ref/#pool-pg-and-crush-config-reference
[ref3]: http://xuxiaopang.com/2016/10/10/ceph-full-install-el7-jewel/
