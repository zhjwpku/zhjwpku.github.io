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

<h4>Jenkinsfile Template</h4>

{% highlight groovy %}
#!groovy

// 设置Pipe只保存最近5次的build
def projectProperties = [
    [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numberToKeepStr: '5']],
]

if(!env.CHANGE_ID) {
    if (env.BRANCH_NAME == null) {
        projectProperties.add(pipelineTriggers([cron('H/30 * * * *')]))
    }
}

properties(projectProperties)

node {
    def workspace = pwd()

    // 将源码checkout到pipeline的workspace，使用Pipeline任务配置的Repo
    // 等同于：git branch: 'jenkinsfile', credentialsId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', url: 'repo url'
    // 但不需要credentialsId
    stage('Checkout Source Code') {
        checkout scm
    }

    stage('build') {
        try {
            def userInput = input(
                id: 'userInput', message: 'Choose the build Environment', ok: 'Submit', parameters[
                    //[$class: 'TextParameterDefinition', defaultValue: '', description: 'Data Sufix', name: 'snapshot'],
                    [$class: 'ChioceParameterDefinition', name: 'target', description: 'Build Env', choices: 'dev\ntesting\nstaging\nproduction'],
                ]
            )

            //sh "./gradlew clean build -Penv=" + userInput['target']
            // 如果只有一个Input值，则直接使用userInput获取输入值
            sh "./gradlew clean build -Penv=" + userInput
        }
        catch(err) {
            stage('Failure Notification') {
                def to emailextrecipients([
                    [$class: 'DevelopersRecipientProvider']
                ])
                mail to: to, subject: "${env.JOB_NAME} Failed!",
                body: "${env.JOB_NAME} failed the last time it was run. See ${env.BUILD_URL} for more information."
                currentBuild.result = 'FAILURE'
            }
        }
    }

    //if (env.BRANCE_NAME == 'release') {
    //    stage('Tag release') {
    //        String version = readFile('Version')
    //        Object git = load 'groovy/git.groovy'
    //        git.gitTag "v${version}", "Release ${version}"
    //    }
    //}
}

{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. Jenkins1.0版本的Workflow插件介绍: [Continuous Delivery With Jenkins Workflow](/assets/pdf/jenkins-workflow.pdf)<br>
2. Pipeline Overview: [https://jenkins.io/doc/book/pipeline/overview/][overview]<br>
3. Jenkinsfile: [https://jenkins.io/doc/book/pipeline/jenkinsfile/][jenkinsfile]
</span>

[overview]: https://jenkins.io/doc/book/pipeline/overview/
[jenkinsfile]: https://jenkins.io/doc/book/pipeline/jenkinsfile/
