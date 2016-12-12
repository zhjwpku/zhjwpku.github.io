---
layout: post
title: Controll version with `git describe`
date: 2016-12-12 18:45:00 +0800
tags:
- gradle
- git-describe
---

在 Gradle 构建中需要定义构件的三元组（groupId, artifactID, version），其中第三项 version 随着程序的演进需要不停地增加，如果你习惯于使用 git tag 定义程序的版本，那么你不再需要通过修改代码来管理版本了。[gradle-git-version][gradle-git-version] 就是这剂灵丹妙药。

{% highlight groovy %}
plugins {
    id 'com.palantir.git-version' version '0.5.2'
}

def details = versionDetails()
version = details.commitDistance ? details.lastTag + '-SNAPSHOT' : details.lastTag
{% endhighlight %}

[gradle-git-version]: https://github.com/palantir/gradle-git-version
