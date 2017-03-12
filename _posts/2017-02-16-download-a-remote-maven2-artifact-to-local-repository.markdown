---
layout: post
title: 下载Maven2构件到本地
date: 2017-02-16 17:10:00 +0800
tags:
- gradle
- nexus
---

写gradle脚本的时候，除了mavenCentral ( 位置 [https://repo1.maven.org/maven2/][ref1] )，还会添加一些其它仓库，如*mavenLocal()*, *maven { url 'http://repo.spring.io/plugins-release' }*，甚至自己搭建的nexus仓库。但有时构建的过程中某些构件的下载异常缓慢，本文介绍笔者加速该过程的一些技巧。

**2017-03-12 更新**

**要将私有仓库放置在 repositories 的前边，因为构建时是按顺序在各个仓库查找需要的构件，如果将私有仓库置于后边，即使含有相应的构件，也会优先从公有仓库中下载，这样自己的仓库就起不到代理的作用了。**

请注意区分：构建 > build; 构件 > artifact

假如构建卡在了下面而你又不想死等。

{% highlight shell %}
Download https://repo1.maven.org/maven2/org/aspectj/aspectjweaver/1.7.4/aspectjweaver-1.7.4.jar
> Building 88% > :test > 1.18 MB/1.76 MB downloaded
{% endhighlight %}

**手动下载并上传到Nexus仓库**

复制构件地址并使用wget、浏览器或迅雷进行下载。然后将下载的构建进行上传：

![nexus upload](/assets/201702/nexus_upload.png)

在gradle脚本中添加nexus仓库地址，即可从Nexux仓库下载需要的构件了。需要注意的是添加的顺序，个人认为添加到mavenCentral()前较好。

至此已经解决了加速的问题，但跟本文的题目没有半毛钱关系啊！我TM要把构件下载到 mavenLocal()，好吧，刚才那个过程已经把构件下载到本地了。

**其实我想介绍下面这个命令：**

{% highlight shell %}
# 从命令行下载构件到本地
→ ~ $ mvn org.apache.maven.plugins:maven-dependency-plugin:2.4:get \
-DartifactId=aspectjweaver \
-DgroupId=org.aspectj \
-Dversion=1.7.4
{% endhighlight %}

顺便贴一个Maven的配置文件，详细的配置解释见 [Apache Maven Settings Reference][ref3]。加入下面的配置文件配置了Nexus仓库的地址，使用上述的命令会先把构件下载到Nexus仓库，再下载到mavenLocal()，相当于把Nexus作为代理。

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<settings
  xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <profiles>

    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>nexus</id>
          <name>local private nexus</name>
          <url>http://repo.example.me/nexus/content/groups/public</url>
        </repository>
      </repositories>
    </profile>

    <profile>
      <id>nexus-snapshots</id>
      <repositories>
        <repository>
          <id>snapshots</id>
          <name>Snapshots</name>
          <url>http://repo.example.me/nexus/content/repositories/snapshots</url>
        </repository>
      </repositories>
    </profile>

    <profile>
      <id>nexus-releases</id>
      <repositories>
        <repository>
          <id>releases</id>
          <name>Releases</name>
          <url>http://repo.example.me/nexus/content/repositories/releases</url>
        </repository>
      </repositories>
    </profile>

  </profiles>

  <activeProfiles>
    <activeProfile>nexus</activeProfile>
    <activeProfile>nexus-snapshots</activeProfile>
    <activeProfile>nexus-releases</activeProfile>
  </activeProfiles>

  <servers>
    <server>
    <id>releases</id>
    <username>example</username>
    <password>PassW0rd</password>
    </server>

    <server>
    <id>snapshots</id>
    <username>example</username>
    <password>PassW0rd</password>
    </server>
  </servers>

</settings>
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [A simple command line to download a remote maven2 artifact to the local repository?][ref2]
</span>

[ref1]: https://repo1.maven.org/maven2/
[ref2]: http://stackoverflow.com/questions/1776496/a-simple-command-line-to-download-a-remote-maven2-artifact-to-the-local-reposito
[ref3]: https://maven.apache.org/settings.html
