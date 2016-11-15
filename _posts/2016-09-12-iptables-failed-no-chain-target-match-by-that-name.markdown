---
layout: post
title: iptables failed. No chain/target/match by that name
date: 2016-09-12 17:23:00 +0800
categories: docker
tags:
- iptables
---

之前在创建docker实例的时候，出现过标题显示的问题，改用Kubernetes的rc进行创建，当时没有出现问题，所以并没有在意，以为Kubernetes会对iptables进行接管。今天使用Kubernetes创建Pod，出现了同样的问题，意识到了问题的严重性。

Kubernetes创建Pod，使用`kubectl describe -f pod-name`，可以看到如下错误：

{% highlight shell %}
10m	10m	1	{kubelet anakin}		Warning	FailedSync	Error syncing pod,
skipping: failed to "StartContainer" for "POD" with RunContainerError: "runContainer: Error respon
se from daemon: {\"message\":\"driver failed programming external connectivity on endpoint k8s_POD
.38f00554_jenkins-server_default_8fdef1e2-78bd-11e6-8f65-14187735bad0_7b519dc8 (a15c19b10f8baa8287
e20aee35afe05a452e647a6949badeafb437a7fe96281f): iptables failed: iptables --wait -t filter -A DOC
KER ! -i docker0 -o docker0 -p tcp -d 10.0.7.11 --dport 8080 -j ACCEPT: iptables: No chain/target/
match by that name.\\n (exit status 1)\"}"
{% endhighlight %}

Google后找到解决的办法如下：

{% highlight shell %}
$ sudo iptables -t filter -N DOCKER
{% endhighlight %}

问题及解决方法：[docker issues 16816][issues16816]

现在对iptables的原理还没有研究，今后应该加强这方面的学习。

[issues16816]: https://github.com/docker/docker/issues/16816
