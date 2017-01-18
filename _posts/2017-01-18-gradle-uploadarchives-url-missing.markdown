---
layout: post
title: Gradle 上传链接不显示的问题
date: 2017-01-18 17:00:00 +0800
tags:
- gradle
- android
---

将 Gradle Android Plugin 版本从 1.5.0 升级到 2.2.1 之后，其中有些 Task 也有相应变化，如 uploadArchives 默认不输出上传的 url 。

升级之前运行 `./gradlew clean upload` 输出的日志包含：

{% highlight shell %}
:com.example.android.video:uploadArchives
Uploading: com/example/android/ARTIFACT-testing/3.14_17447/ARTIFACT-testing-3.14_17447.apk to repository remote at http://10.0.251.224:8081/nexus/content/repositories/releases
Transferring 8698K from remote
Uploaded 8698K
Uploading: com/example/android/ARTIFACT-testing/3.14_17447/ARTIFACT-testing-3.14_17447.map to repository remote at http://10.0.251.224:8081/nexus/content/repositories/releases
Transferring 4540K from remote
Uploaded 4540K
{% endhighlight %}

在 Jenkins Job 中通过添加 Post-build Actions 将上传的 url 进行拼接给 QA 提供相应的下载地址。

{% highlight shell %}
Set build description
Regular expression: Uploading: ([^\s]*)/(ARTIFACT.*)apk to repository remote at ([^\s]*)
Description:        Download APK:<br/> <a href="\3/\1/\2apk">\2apk</a><br> Download map:<br/><a href="\3/\1/\2map">\2map</a>
{% endhighlight %}

但在升级后，默认 uploadArchives 不将上传的地址打印，解决办法：

{% highlight shell %}
→ ~/android (develop) $./gradlew clean upload --info
{% endhighlight %}

Kind of trick, but it works :)

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [uploadArchives does nothing. What am I missing?][ref]
</span>

[ref]: https://github.com/Codearte/gradle-nexus-staging-plugin/issues/20
