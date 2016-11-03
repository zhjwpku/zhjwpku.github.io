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

    // 将构建好的包上传到Nexus仓库，其中用户名密码在Jenkins `Configure System`中添加
    stage('Publish') {
        timeout(time: 10, unit: 'MINUTES') {
            input "Publish ${env.ENV} archive package to Nexus?"

            env.REPO_URL = sh(returnStdout: true, script: "./gradlew -Penv=${env.ENV} publish | grep -oP -m 1 'http.*?zip'").trim()
            echo "Publish to url: ${env.REPO_URL}"
        }   
    }   

    // 将上传到Nexus上的包拉下来部署到运行环境，需要在Jenkins上用ssh-copy-id来设置免密码登陆
    stage('Deploy') {
        timeout(time: 10, unit: 'MINUTES') {
            input "Deploy to ${env.ENV} environment?"
                def host = map["${env.ENV}"]
                if ("${env.ENV}" == "dev") {
                    // -x让shell打印每一行执行的命令，-s表示从标准输入读取要运行的脚本，这里重定向了deploy.sh
                    sh "ssh root@${host} 'bash -x -s' < ./deploy.sh " + "${env.REPO_URL}"
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

def archiveUnitTestResults() {
    step([$class: "JunitResultArchiver", testResults: "build/**/TEST-*.xml"])
}

def archiveCheckStyleResults() {
    step([$class: "CheckStylePublisher",
        canComputeNew: false,
        defaultEncoding: "",
        health: "",
        pattern: "build/reports/checkstyle/main.xml",
        unHealthy: ""
    ])
}

{% endhighlight %}

deploy.sh脚本中为远程机器上执行的命令。
{% highlight shell %}
#!/bin/bash

WORK_DIR="/opt/startimestv"
SERVER_DIR="$WORK_DIR/upms"
RUN_PID="$SERVER_DIR/bin/run.pid"

if [ ! -d $WORK_DIR ]; then
    mkdir -p $WORK_DIR
fi

# 删除之前运行的进程
if [ -f $RUN_PID ]; then
    PID=`cat $RUN_PID`
    echo "Killing former running progress $PID"
    cat $RUN_PID | xargs kill -9
    echo "Killing former running progree $PID done."
fi

yes | rm -r $WORK_DIR/*

cd $WORK_DIR

wget $1

unzip *.zip -d $SERVER_DIR

mv $SERVER_DIR/*/* $SERVER_DIR

cd $SERVER_DIR/bin/

nohup $SERVER_DIR/bin/xxxx-start > /dev/null 2>&1 &

echo $! > $RUN_PID
echo "SERVER started with pid `cat $RUN_PID`"

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
