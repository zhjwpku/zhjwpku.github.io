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

// 或使用如下方式将 git describe 信息作为 version
version = gitVersion()
{% endhighlight %}

版本管理参照：[语义化版本][SemanticVersioning]

**The Maven way**

{% highlight shell %}
1.0-SNAPSHOT   =>   1.0
      |              |
 during dev      when released
{% endhighlight %}

插件: Maven Release plugin

**The Continuous Delivery way**

{% highlight shell %}
   1.0.134   =>   1.0.134
      |              |
 during dev      when released
{% endhighlight %}

Every commit can be a release

**implemented with Gradle**

*gradle/versioning.gradle*

{% highlight java %}
ext.buildTimestamp = new Date().format('yyyy-MM-dd HH:mm:ss')

def ciBuildNumber = System.env.SOURCE_BUILD_NUMBER      // Jenkins Build Number
version = (ciBuildNumber) ? new ProjectVersion(1, 0, Integer.parseInt(ciBuildNumber)) :
        new ProjectVersion(1, 0, 0)

class ProjectVersion {

    Integer major
    Integer minor
    Integer build

    ProjectVersion(Integer major, Integer minor, Integer build) {
        this.major = major
        this.minor = minor
        this.build = build
    }

    @Override
    String toString() {     // Builds version String representation
        "$major.$minor.$build"
    }
}
{% endhighlight %}

在根项目的build.gradle中包含以下内容：

{% highlight java %}
allprojects {
    apply from: "$rootDir/gradle/versioning.gradle"
}
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Building a Continuous Delivery Pipeline with Gradle and Jenkins][ref1]
</span>

[gradle-git-version]: https://github.com/palantir/gradle-git-version
[SemanticVersioning]: http://semver.org/lang/zh-CN/
[ref1]: https://www.youtube.com/watch?v=V0FpbDkKYtA
