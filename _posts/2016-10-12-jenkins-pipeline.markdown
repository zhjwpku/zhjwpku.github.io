---
layout: post
title: Jenkins Pipeline
date: 2016-10-12 22:00:00 +0800
tags:
- Jenkins
- CI
---

Jenkins2.0的[Pipeline][overview](formerly called 'workflow')允许用户定义软件项目的整个生命周期来支持持续集成。在项目根目录下创建一个[Jenkinsfile][jenkinsfile]文件，编写从构建到部署的整个生命周期，减少了复杂的Jenkins界面操作。

Pipeline特性如下：

- 持久性(durable)：在Jenkins master按计划和非计划的重启后，pipeline仍然能够工作，类似nohup
- 暂停：支持任务暂停并等待手动触发
- 灵活：Pipeline支持复杂的持续集成需求，包括fork, join, loop, work in parallel
- 扩展：支持DSL(Domain Scripting Language)并可与其他插件集成

一些跟Pipeline相关的插件：

- Pipeline: 包含了核心pipeline引擎以及其从属的插件，`Pipeline: API, Pipeline: Basic Steps, Pipeline: Durable Task Step, Pipeline: Execution Support, Pipeline: Global Shared Library for CPS pipeline, Pipeline: Groovy CPS Execution, Pipeline: Job, Pipeline: SCM Step, Pipeline: Step API`
- Pipeline Stage View: 图形化的泳道阶段图
- Multibranch Pipeline: 自动构建包含`Jenkinsfile的分支`
- Docker Pipeline: 在Jenkinsfile脚本中使用Docker containers进行构建

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
            // 目前这种写法不好，因为在node里使用input将使得node本身和workspace被lock，不能够被别的job使用
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

WORK_DIR="/opt/workdir"
PRJ_DIR="$WORK_DIR/prj1"
PRJ_TMP="$WORK_DIR/tmp"
RUN_PID="$PRJ_DIR/bin/run.pid"

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

yes | rm -r $PRJ_DIR

cd $WORK_DIR
mkdir -p $UPMS_DIR

wget $1

url=$1

unzip ${url##*/} -d $PRJ_TMP
yes | rm ${url##*/}

mv $PRO_TMP/*/* $PRO_DIR

cd $PRO_DIR/bin/

nohup $PRO_DIR/bin/xxxx-start > /dev/null 2>&1 &

echo $! > $RUN_PID
echo "SERVER started with pid `cat $RUN_PID`"

{% endhighlight %}

Stage View

![pipeline stage view](/assets/201610/pipeline_stage_view.png)

<h4>Another Jenkinsfile</h4>

[https://github.com/zhjwpku/ijkplayer/blob/master/Jenkinsfile][jenkinsfile2]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. Jenkins1.0版本的Workflow插件介绍: [Continuous Delivery With Jenkins Workflow](/assets/pdf/jenkins-workflow.pdf)<br>
2. Pipeline Overview: [https://jenkins.io/doc/book/pipeline/overview/][overview]<br>
3. Jenkinsfile: [https://jenkins.io/doc/book/pipeline/jenkinsfile/][jenkinsfile]<br>
4. Pipeline Steps Reference: [https://jenkins.io/doc/pipeline/steps/][stepRef]<br>
5. [Top 10 Best Practices for Jenkins Pipeline Plugin][top10]<br>
6. [Using the Pipeline Plugin to Accelerate Continuous Delivery — Part 1][part1]<br>
7. [Using the Pipeline Plugin to Accelerate Continuous Delivery — Part 2][part2]<br>
8. [Using the Pipeline Plugin to Accelerate Continuous Delivery — Part 3][part3]
</span>

[overview]: https://jenkins.io/doc/book/pipeline/overview/
[jenkinsfile]: https://jenkins.io/doc/book/pipeline/jenkinsfile/
[stepRef]: https://jenkins.io/doc/pipeline/steps/
[top10]: https://www.cloudbees.com/blog/top-10-best-practices-jenkins-pipeline-plugin
[part1]: https://www.cloudbees.com/blog/using-pipeline-plugin-accelerate-continuous-delivery-part-1
[part2]: https://www.cloudbees.com/blog/using-pipeline-plugin-accelerate-continuous-delivery-part-2
[part3]: https://www.cloudbees.com/blog/using-pipeline-plugin-accelerate-continuous-delivery-part-3
[jenkinsfile2]: https://github.com/zhjwpku/ijkplayer/blob/master/Jenkinsfile
