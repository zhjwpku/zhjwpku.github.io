---
layout: post
title: Nexus Latest Snapshot and BadRequest
date: 2016-11-11 19:30:00 +0800
tags:
- nexus
- gradle
---

ijkplayer安卓库使用gradle构建，今天给它编写Jenkinsfile的时候遇到了两个问题。

<h4>问题1</h4>

gradle的maven插件上传完artifact之后并不会打印包上传的位置，这对Jenkins Pipeline是不太友好，因为我希望在Jenkinsfile中将上传的位置发送给需要这些artifact的开发者。

Nexus提供了"redirect"的API，如，我们可以使用如下url来定位最新的构建结果：

{% highlight shell %}
https://repo.example.com/nexus/service/local/artifact/maven/redirect?r=snapshots&g=tv.danmaku.ijk.media&a=ijkplayer-armv7a&v=LATEST&p=aar
{% endhighlight %}

其中r为repository的名字，g为groupId，a为artifactId，v代表version。

在任务uploadArchives中加入如下操作，就可以在上传之后打印出artifact的信息。

{% highlight groovy %}
doLast {
    println "Upload Successful!\n"
        println "  groupID: $GROUP"
        println "  artifactId: $POM_ARTIFACT_ID"
        println "  version: $VERSION_NAME"
        println "  type: aar"
        println "  url: http://repo.startimes.me/nexus/service/local/artifact/maven/redirect?r=${isReleaseBuild()? 'releases' : 'snapshots'}&g=$GROUP&a=$POM_ARTIFACT_ID&v=$VERSION_NAME&p=aar"   
}
{% endhighlight %}

<h4 style="margin-top: 20px">问题2</h4>

由于同一个release版本三元组完全一样，上传会出现如下错误：

![Picture](/assets/201611/nexus_err400.png)

这是由于Nexus仓库禁止Redeploy，只需要更改仓库的Deployment Policy为Allow Redeploy即可。

![Picture](/assets/201611/nexus_repo.png)

*注：不建议将Deployment Policy设置为Allow，这样不利于回滚*

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
[How do I provide URL access to the latest snapshot of an artifact in Nexus?][url]
</span>

[url]: http://stackoverflow.com/questions/9280447/how-do-i-provide-url-access-to-the-latest-snapshot-of-an-artifact-in-nexus
