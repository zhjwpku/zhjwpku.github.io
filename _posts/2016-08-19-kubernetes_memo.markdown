---
layout: post
title:  "Kubernetes学习备忘"
date:   2016-07-19 19:30:00 +0800
categories: docker
tags:
- k8s
- docker
---

由于项目需要，现开始研究Kubernetes。本文记录从2016.8.15日开始到2016.8.19日学习Kubernetes的要点和心得，
希望通过一周的学习，能对Kubernetes有一个大致的了解。

<h4>Kubernetes是什么</h4>

Kubernetes是Google开源的容器集群管理系统。它构建在Docker技术之上，为容器化的应用提供资源调度、部署运行、
服务发现、扩容缩容等一整套功能。Google从2004年开始使用容器技术，于2006年发布Cgroup，Google内部开发的集
群资源管理平台Borg和Omega广泛使用在Google的各个基础设施中，Kubernetes的灵感来源于Borg系统，更是吸收了
包括Omega在内的容器管理器的经验和教训。

Kubernetes，在古希腊语中是舵手的意思，它利用Google在容器技术上的实践经验和技术积累，同时吸收了Docker社
区的最佳实践，已经成为云计算服务的舵手。

Kubernetes有以下优秀特性：

* 强大的容器编排能力

  Kubernetes深度集成了Docker，天然适应容器的特点，设计出强大的容器编排能力，比如容器组合、标签选择和
  服务发现等，可以满足企业级需求。
* 轻量级

  Kubernetes遵循微服务架构，整个系统划分出各个功能独立的组件，组件之间边界清晰，部署简单，可以轻易地
  运行在各种系统和环境中。同时，Kubernetes中的许多功能都实现了插件化，可以非常方便地进行扩展和替换。
* 可移植

  适用于公有云、私有云、混合云以及多重云
* 可扩展

  Kubernetes支持模块化、插件化、钩子化及可编排
* 自愈

  自动定位、自动重启、自动创建副本

Kubernetes基本概念：

* [Cluster][cluster](集群): 一个集群由一组物理机、虚拟机及其他基础架构资源构成,这些资源被Kubernetes
用来跑你的应用程序。

* [Node][node](节点): 一个节点是运行Kubernetes的物理机或虚拟机，在节点之上可以调度Pods。Pod最终运行
在Node上，Node可以认为是Pod的宿主机。

* [Pod][pod]: Pod是若干共享卷的一组应用容器的组合。Pod是最小的部署单元（而非容器）,可以使用Kubernetes
创建、调度并管理Pod。Pods可以被单独地创建，但比较好的方式是用replication controller来创建Pod。Pod通
过提供更高层次的抽象，提供了更加灵活的部署和管理模式。

* [Replication controller][replication controller]: Replication controller管理Pods的生命周期，它通过
创建或杀死Pod保证在任何时间都有指定数目的pods在运行，Replication controller是弹性伸缩、滚动升级的实
现核心。

* [Service][service](服务): 服务为一组Pods提供单一、稳定的名字和地址。它是真实应用服务的抽象，定义了
Pod的逻辑集合和访问这个Pod集合的策略。Service将代理Pod对外表现为一个单一访问接口，外部不需要了解后端
Pod如何运行，这给扩展和维护带来很多好处，提供了一套简化的服务代理和发现机制。

* [Label][label](标签): 标签是用于区分Pod、Service、Replication controller的Key/Value对，实际上，
Kubernetes中的任意API对象都可以通过Label进行标识。Label是Service和Replication Controller运行的基础，
它们都通过Label来关联Pod，相比于强绑定模型，这是一种很好的松耦合关系。

<h4>Minikube安装</h4>

如果为了开发和测试，想在本地创建一个单点的Kubernetes集群，[Minikube][minikube]是一种推荐的方式，它打
包并配置了一个Linux VM，Docker以及所有的Kubernetes组件，并为本地部署进行了优化。Minikube支持k8s的如
下特性：

* DNS
* NodePorts
* ConfigMaps and Secrets
* Dashboards

Minikube现在还不能支持云提供商的部分特性：

* Loadbalancers
* PersistentVolumes
* Ingress

Minikube需要打开VT-x/AMD-v硬件虚拟化，查看是否打开运行如下命令：
{% highlight shell %}
$cat /proc/cpuinfo | grep 'vmx\|svm'
{% endhighlight %}

安装最新的Virtualbox：
{% highlight shell %}
$cd /etc/yum.repos.d
$wget http://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo
$sudo yum update -y
$sudo yum --enablerepo=epel install dkms -y
$sudo yum groupinstall "Development Tools" -y
$sudo yum install kernel-devel -y
$sudo yum install VirtualBox-5.0.x86_64 -y
{% endhighlight %}

下载minikube二进制文件并放置在系统可以调用的路径下：
{% highlight shell %}
$curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.7.1/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
{% endhighlight %}

下载kubectl二进制文件:
{% highlight shell %}
// linux/amd64
// generic download path is:
// https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/${GOOS}/${GOARCH}/${K8S_BINARY}
$curl -Lo kubectl http://storage.googleapis.com/kubernetes-release/release/v1.3.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
{% endhighlight %}

启动Kubernetes单点集群，以下命令会启动一个轻量级的本地集群，包含一个master, etcd, Docker以及一个Node。
{% highlight shell %}
$yum install "kernel-devel-uname-r == $(uname -r)"
$sudo /sbin/rcvboxdrv setup
$minikube start
Starting local Kubernetes cluster...
Kubernetes is available at https://192.168.99.100:8443.
Kubectl is now configured to use the cluster.
{% endhighlight %}

如下命令可以显示Kube-system Pods:
{% highlight shell %}
$kubectl get pods --all-namespaces
NAMESPACE     NAME                            READY     STATUS    RESTARTS   AGE
kube-system   kube-addon-manager-minikubevm   1/1       Running   0          1h
kube-system   kubernetes-dashboard-tava9      1/1       Running   0          1h
{% endhighlight %}

`minikube dashboard`用来打开Kubernetes的Dashboard:

![minikube dashboard](/assets/201608/minikube_dashboard.png)

一些其它有用的命令：
{% highlight shell %}
$kubectl get nodes
$eval $(minikube docker-env)
{% endhighlight %}


<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
[1] 吴龙辉. Kubernetes实战. 电子工业出版社. 2016.
</span>

[cluster]: http://kubernetes.io/docs/admin/
[node]: http://kubernetes.io/docs/admin/node/
[pod]: http://kubernetes.io/docs/user-guide/pods/
[replication controller]: http://kubernetes.io/docs/user-guide/replication-controller/
[service]: http://kubernetes.io/docs/user-guide/services/
[label]: http://kubernetes.io/docs/user-guide/labels/
[minikube]: http://kubernetes.io/docs/getting-started-guides/minikube/
