---
layout: post
title: Add property files to Classpath
date: 2017-03-14 12:00:00 +0800
tags:
- IntelliJ
---

Eclipse 在运行程序的时候会自动寻找资源文件，如果 main/resources 没有响应的资源文件，会自动从 test/resources 中查找，以保证在程序能够正常运行。IntelliJ 没有这样的机制，需要开发者手动添加。

首先说明一下为什么有这样的需求。有人会觉得，直接把资源放置在 main/resources 文件夹下程序不就能正常运行了吗？

不这样做的原因是我们需要对不同的运行环境（dev、testing、staging、production）配置不同的资源文件，举个简单的例子，线上环境跟测试环境使用的数据库肯定不能是一个吧。如果将这些资源文件置于 main/resources 下，构建生成的 Jar 包中会包含这些文件，Gradle 打包生成的执行脚本中 Classpath 会包含 Jar 包和我们想要区分的配置目录（该目录由 Gradle 生成）。这样极有可能造成在不同环境运行的程序都使用的是 main/resources 下的配置文件。

明白了其中的缘由，下面来看在 IntelliJ IDEA 中如何配置运行时需要的资源文件。

1. 点击程序运行的子项目
2. 点击 F4 或 Cmd+Shift+A Open Module Settings
3. 点击 Dependencies 标签页
4. 点击 '+' 按钮病选择 'JARs or directories...'
5. 选择资源文件所在的目录（注意，必须是目录）然后点 OK
6. 在弹出的"Choose Categories of Selected File"窗口中选择 'classes' 并点击 OK
7. 在新添加的依赖选项的右侧选择Scope => Runtime

上述操作后资源文件会添加到 Classpath 中，从而让程序正常运行 :)

上一张配置图：

![intellij_classpath.png](/assets/201703/intellij_classpath.png)

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [how to add directory to classpath in an application run profile in intellij idea?][ref1]<br>
2 [Problem Setting Classpath For System Tests in IntelliJ][ref2]
</span>

[ref1]: http://stackoverflow.com/questions/854264/how-to-add-directory-to-classpath-in-an-application-run-profile-in-intellij-idea
[ref2]: https://intellij-support.jetbrains.com/hc/en-us/community/posts/206172509-Problem-Setting-Classpath-For-System-Tests-in-IntelliJ
