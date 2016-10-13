---
layout: post
title: Jenkins Pipeline
date: 2016-10-12 22:00:00 +0800
tags:
- Jenkins
- CI
---

Jenkins2.0的[Pipeline][overview]允许用户定义软件项目的整个生命周期来支持持续集成。在项目根目录下创建一个[Jenkinsfile][jenkinsfile]文件，编写从构建到部署的整个生命周期，减少了复杂的Jenkins界面操作。

Pipeline特性如下：

- 耐用
- 暂停：支持任务暂停并等待手动触发
- 灵活：Pipeline支持复杂的持续集成需求，包括fork, join, loop, work in parallel
- 扩展：支持DSL(Domain Scripting Language)并可与其他插件集成

![real world pipeline flow](/assets/201610/realworld-pipeline-flow.png)


**扩展阅读**

Jenkins1.0版本的Workflow插件介绍: [Continuous Delivery With Jenkins Workflow](/assets/pdf/jenkins-workflow.pdf)

Pipeline Overview: [https://jenkins.io/doc/book/pipeline/overview/][overview]

Jenkinsfile: [https://jenkins.io/doc/book/pipeline/jenkinsfile/][jenkinsfile]

[overview]: https://jenkins.io/doc/book/pipeline/overview/
[jenkinsfile]: https://jenkins.io/doc/book/pipeline/jenkinsfile/
