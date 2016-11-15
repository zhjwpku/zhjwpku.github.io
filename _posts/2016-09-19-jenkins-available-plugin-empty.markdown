---
layout: post
title: Jenkins可选插件列表为空
date: 2016-09-19 12:30:00 +0800
tags:
- jenkins
- CI
---

安装完Jenkins之后，在浏览器进行初始化配置的时候，提示`This Jenkins instance appears to be offline`。

在Jenkins的`系统管理`界面，点击`管理插件`，发现`可更新`和`可选插件列表`全部为空，从stackoverflow上找[解决办法][solution]，提示让点击Advanced标签，点击右下角的`立即获取`。

![advanced](/assets/201609/advanced.png)

但事情并没有按照应该的情节发展：

![update-center-oops](/assets/201609/update-center-oops.png)

根据提示的错误，将链接`http://updates.jenkins-ci.org/update-center.json?id=default&version=2.7.4`贴入浏览器进行查看，发现它重定位到了`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/stable-2.7/update-center.json`，于是用这个链接替换原来的`http://updates.jenkins-ci.org/update-center.json`。

提交之后点击立即获取，WoW，`可选插件`标签下列出了一片可选插件。

但是在安装插件的时候，又出现了消息摘要不一致的问题，[JENKINS-38195][38195]对这个问题进行了记录，目前还没有找到解决的办法。

**2016-09-21更新**

2.7.2版本可用：
{% highlight shell %}
$ wget http://mirrors.jenkins-ci.org/war-stable/2.7.2/jenkins.war
$ nohup java -jar jenkins.war &
{% endhighlight %}

如果想要使用不同的端口号，需要添加`--httpPort`参数：
{% highlight shell %}
$ nohup java -jar jenkins.war --httpPort=9090 &
{% endhighlight %}


[solution]: http://stackoverflow.com/questions/16213982/unable-to-find-plugins-in-list-of-available-plugins-in-jenkins
[38195]: https://issues.jenkins-ci.org/browse/JENKINS-38195
