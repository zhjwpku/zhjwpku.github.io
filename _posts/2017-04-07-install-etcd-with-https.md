---
layout: post
title: Install etcd with https
date: 2017-04-07 16:00:00 +0800
categories: docker
tags:
- etcd
---

安装 [etcd][etcd] 是一件很简单的事情了，因为有太多教程了，而且 [etcd playground][playground] 提供了很好的安装向导。本文仅作为笔者安装 etcd 的记录，方便查看。

|---
| Name | ADDRESS | HOSTNAME
|:-|:-|:-
| etcd1 | 172.16.63.11 | etcd1
|---
| etcd2 | 172.16.63.12 | etcd2
|---
| etcd3 | 172.16.63.13 | etcd3

<h6></h6>

**Vagrantfile**

{% highlight shell %}
## ectd1 Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.hostname = "etcd1"

  config.vm.network "public_network", ip: "172.16.63.11"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum update
    sudo yum install -y vim git bind-utils
  SHELL
end

## ectd2 Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.hostname = "etcd2"

  config.vm.network "public_network", ip: "172.16.63.12"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum update
    sudo yum install -y vim git bind-utils
  SHELL
end

## ectd3 Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.hostname = "etcd3"

  config.vm.network "public_network", ip: "172.16.63.13"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum update
    sudo yum install -y vim git bind-utils
  SHELL
end
{% endhighlight %}

**在 etcd playgound 进行相应的设置**

![etcd-install](/assets/201704/etcd-install.png)

Playgound 会给出所有安装及启动的命令。

**安装 cfssl**

{% highlight shell %}
→ ~ # curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /tmp/cfssl
→ ~ # chmod +x /tmp/cfssl
→ ~ # mv /tmp/cfssl /usr/local/bin/cfssl
→ ~ # curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /tmp/cfssljson
→ ~ # chmod +x /tmp/cfssljson
→ ~ # mv /tmp/cfssljson /usr/local/bin/cfssljson
# 验证
→ ~ # cfssl version
→ ~ # cfssljson -h
# 创建证书目录
→ ~ # mkdir -p $HOME/certs
{% endhighlight %}

**生成自签名根CA证书**

{% highlight shell %}
→ ~ # cat > $HOME/certs/etcd-root-ca-csr.json <<EOF
{
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd security",
      "L": "Beijing",
      "ST": "Beijing",
      "C": "CHN"
    }
  ],
  "CN": "etcd-root-ca"
}
EOF
→ ~ # cfssl gencert --initca=true $HOME/certs/etcd-root-ca-csr.json | cfssljson --bare $HOME/certs/etcd-root-ca
→ ~ # ll $HOME/certs
total 24
drwxr-xr-x.  2 root root 4096 Apr  7 15:31 .
dr-xr-x---. 16 root root 4096 Apr  7 11:52 ..
-rw-r--r--.  1 root root 1708 Apr  7 15:31 etcd-root-ca.csr
-rw-r--r--.  1 root root  219 Apr  7 15:30 etcd-root-ca-csr.json
-rw-------.  1 root root 3243 Apr  7 15:31 etcd-root-ca-key.pem
-rw-r--r--.  1 root root 2082 Apr  7 15:31 etcd-root-ca.pem
# 验证证书
→ ~ # openssl x509 -in $HOME/certs/etcd-root-ca.pem -text -noout

# 配置证书参数
→ ~ # cat > $HOME/certs/etcd-gencert.json <<EOF
{
  "signing": {
    "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
    }
  }
}
EOF
→ ~ # 
{% endhighlight %}

**生成本地使用的证书**

{% highlight shell %}
# etcd1
→ ~ # cat > $HOME/certs/etcd1-ca-csr.json <<EOF
{
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd security",
      "L": "Beijing",
      "ST": "Beijing",
      "C": "CHN"
    }
  ],
  "CN": "etcd1",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "172.16.63.11"
  ]
}
EOF
→ ~ # cfssl gencert \
    --ca $HOME/certs/etcd-root-ca.pem \
    --ca-key $HOME/certs/etcd-root-ca-key.pem \
    --config $HOME/certs/etcd-gencert.json \
    $HOME/certs/etcd1-ca-csr.json | cfssljson --bare $HOME/certs/etcd1
2017/04/07 15:38:50 [INFO] generate received request
2017/04/07 15:38:50 [INFO] received CSR
2017/04/07 15:38:50 [INFO] generating key: rsa-4096
2017/04/07 15:38:58 [INFO] encoded CSR
2017/04/07 15:38:59 [INFO] signed certificate with serial number 698252832011377188153463075715675969984492806137
2017/04/07 15:38:59 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

# etcd2/etcd3与上述过程一样，更改相应的IP和名字即可
# 最终certs目录下的文件
→ ~ # ll certs/
total 76
drwxr-xr-x.  2 root root 4096 Apr  7 15:44 .
dr-xr-x---. 16 root root 4096 Apr  7 11:52 ..
-rw-r--r--.  1 root root  283 Apr  7 15:37 etcd1-ca-csr.json
-rw-r--r--.  1 root root 1769 Apr  7 15:38 etcd1.csr
-rw-------.  1 root root 3247 Apr  7 15:38 etcd1-key.pem
-rw-r--r--.  1 root root 2155 Apr  7 15:38 etcd1.pem
-rw-r--r--.  1 root root  283 Apr  7 15:43 etcd2-ca-csr.json
-rw-r--r--.  1 root root 1769 Apr  7 15:43 etcd2.csr
-rw-------.  1 root root 3247 Apr  7 15:43 etcd2-key.pem
-rw-r--r--.  1 root root 2155 Apr  7 15:43 etcd2.pem
-rw-r--r--.  1 root root  283 Apr  7 15:43 etcd3-ca-csr.json
-rw-r--r--.  1 root root 1769 Apr  7 15:44 etcd3.csr
-rw-------.  1 root root 3243 Apr  7 15:44 etcd3-key.pem
-rw-r--r--.  1 root root 2155 Apr  7 15:44 etcd3.pem
-rw-r--r--.  1 root root  204 Apr  7 15:34 etcd-gencert.json
-rw-r--r--.  1 root root 1708 Apr  7 15:31 etcd-root-ca.csr
-rw-r--r--.  1 root root  219 Apr  7 15:30 etcd-root-ca-csr.json
-rw-------.  1 root root 3243 Apr  7 15:31 etcd-root-ca-key.pem
-rw-r--r--.  1 root root 2082 Apr  7 15:31 etcd-root-ca.pem
{% endhighlight %}

更详细生成自签证书的文档见 [Generate self-signed certificates][ref2]。

**安装etcd**

{% highlight shell %}
# 以etcd1为例
[vagrant@etcd1 ~]$ ETCD_VER=v3.1.5
# choose either URL
[vagrant@etcd1 ~]$ GOOGLE_URL=https://storage.googleapis.com/etcd
[vagrant@etcd1 ~]$ GITHUB_URL=https://github.com/coreos/etcd/releases/download
[vagrant@etcd1 ~]$ DOWNLOAD_URL=${GOOGLE_URL}
[vagrant@etcd1 ~]$ 
[vagrant@etcd1 ~]$ rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
[vagrant@etcd1 ~]$ rm -rf /tmp/test-etcd && mkdir -p /tmp/test-etcd
[vagrant@etcd1 ~]$ 
[vagrant@etcd1 ~]$ curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
[vagrant@etcd1 ~]$ tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/test-etcd --strip-components=1
[vagrant@etcd1 ~]$ 
[vagrant@etcd1 ~]$ sudo cp /tmp/test-etcd/etcd* /usr/local/bin
[vagrant@etcd1 ~]$ 
[vagrant@etcd1 ~]$ /usr/local/bin/etcd --version
[vagrant@etcd1 ~]$ ETCDCTL_API=3 /usr/local/bin/etcdctl version
{% endhighlight %}

**启动etcd集群**

把相应的证书拷贝分别拷贝到三台机器的*/etc/ssl/certs* 文件夹下。然后编写 systemd 启动文件，如下：

{% highlight shell %}
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd
Conflicts=etcd.service
Conflicts=etcd2.service

[Service]
Type=notify
Restart=always
RestartSec=5s
LimitNOFILE=40000
TimeoutStartSec=0

ExecStart=/usr/local/bin/etcd --name etcd1 \
    --data-dir /var/lib/etcd \
    --listen-client-urls https://172.16.63.11:2379 \
    --advertise-client-urls https://172.16.63.11:2379 \
    --listen-peer-urls https://172.16.63.11:2380 \
    --initial-advertise-peer-urls https://172.16.63.11:2380 \
    --initial-cluster etcd1=https://172.16.63.11:2380,etcd2=https://172.16.63.12:2380,etcd3=https://172.16.63.13:2380 \
    --initial-cluster-token my-etcd-token \
    --initial-cluster-state new \
    --auto-compaction-retention 1 \
    --client-cert-auth \
    --trusted-ca-file /etc/ssl/certs/etcd-root-ca.pem \
    --cert-file /etc/ssl/certs/etcd1.pem \
    --key-file /etc/ssl/certs/etcd1-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file /etc/ssl/certs/etcd-root-ca.pem \
    --peer-cert-file /etc/ssl/certs/etcd1.pem \
    --peer-key-file /etc/ssl/certs/etcd1-key.pem

[Install]
WantedBy=multi-user.target
{% endhighlight %}

使用如下命令启动：

{% highlight shell %}
# to start service
sudo systemctl daemon-reload
sudo systemctl cat etcd1.service
sudo systemctl enable etcd1.service
sudo systemctl start etcd1.service

# to get logs from service
sudo systemctl status etcd1.service -l --no-pager
sudo journalctl -u etcd1.service -l --no-pager|less
sudo journalctl -f -u etcd1.service

# to stop service
sudo systemctl stop etcd1.service
sudo systemctl disable etcd1.service
{% endhighlight %}

etcd2/etcd3 同 etcd1，不再罗列。

**验证集群状态**

使用上述的CA证书生成一个客户端证书，这里假设名字为 k8s。在集群之外查看集群的健康状态。

{% highlight shell %}
→ ~/certs # ETCDCTL_API=3 etcdctl --endpoints 172.16.63.11:2379,172.16.63.12:2379,172.16.63.13:2379 \
    --cacert /root/certs/etcd-root-ca.pem \
    --cert /root/certs/k8s.pem \
    --key /root/certs/k8s-key.pem \
    endpoint health
2017-04-07 16:17:27.065814 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-04-07 16:17:27.068223 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-04-07 16:17:27.070589 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
172.16.63.12:2379 is healthy: successfully committed proposal: took = 4.685102ms
172.16.63.11:2379 is healthy: successfully committed proposal: took = 6.539865ms
172.16.63.13:2379 is healthy: successfully committed proposal: took = 6.253371ms
{% endhighlight %}

上面出现的 warn 最近在 github 上正在讨论，本文会持续跟进。详见 [issue #7647][ref1]。

[etcd]: https://github.com/coreos/etcd
[playground]: http://play.etcd.io/install
[ref1]: https://github.com/coreos/etcd/issues/7647
[ref2]: https://coreos.com/os/docs/latest/generate-self-signed-certificates.html
