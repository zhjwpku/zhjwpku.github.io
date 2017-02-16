---
layout: post
title: Jenksin Pipeline Input with timeout
date: 2016-12-15 22:00:00 +0800
tags:
- Jenkinsfile
---

在 Jenkins Pipeline 中如何在定时输入任务超时后设定一个默认的变量？Answer is: 捕获异常，并根据异常的制造者来进行后续处理。

**代码：**

{% highlight groovy %}
def didInput = true
if (env.BRANCH_NAME == 'develop') {
    timeout(time: 1, unit: 'MINUTES') {
        try {
            def userInput = input(
              id: 'userInput', message: 'Choose paramters', ok: 'Submit', parameters: [
                [$class: 'ChoiceParameterDefinition', name: 'target', description:'', choices: 'dev\ntesting\nstaging\nproduction'],
              ]   
            )   
            env.ENV = userInput
        } catch(err) {
            def user = err.getCauses()[0].getUser()
            if ('SYSTEM' == user.toString()) { // SYSTEM means timeout
                env.ENV = 'dev'     // Set default Environment to 'dev'
            } else {
                didInput = false
                echo "Aborted by: [${user}]"
            }   
        }   
    }   

    if (didInput) {
        try {
            sh "./gradlew clean build -Penv=${env.ENV}"
        } catch(err) {
            stage('Failure Notification') {
                def to = emailextrecipients([
                  [$class: 'DevelopersRecipientProvider']
                ])  

                mail to: to, subject: "${env.JOB_NAME} Failed!",
                body: "${env.JOB_NAME} failed the last time it was run. See ${env.BUILD_URL} for more information."
                currentBuild.result = 'FAILURE'
            }       
        }       
    }       
} else {
    try {
        sh "./gradlew clean build"
    } catch(err) {
        stage('Failure Notification') {
            def to = emailextrecipients([
              [$class: 'DevelopersRecipientProvider']
            ])

            mail to: to, subject: "${env.JOB_NAME} Failed!",
            body: "${env.JOB_NAME} failed the last time it was run. See ${env.BUILD_URL} for more information."
            currentBuild.result = 'FAILURE'
        }
    }
}       
{% endhighlight %}

在构建时如果遇到如下问题：

{% highlight groovy %}
org.jenkinsci.plugins.scriptsecurity.sandbox.RejectedAccessException: Scripts not permitted to use method org.jenkinsci.plugins.workflow.steps.FlowInterruptedException getCauses
    at org.jenkinsci.plugins.scriptsecurity.sandbox.whitelists.StaticWhitelist.rejectMethod(StaticWhitelist.java:176)
    at org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SandboxInterceptor.onMethodCall(SandboxInterceptor.java:119)
    at org.kohsuke.groovy.sandbox.impl.Checker$1.call(Checker.java:149)
    at org.kohsuke.groovy.sandbox.impl.Checker.checkedCall(Checker.java:146)
    at com.cloudbees.groovy.cps.sandbox.SandboxInvoker.methodCall(SandboxInvoker.java:16)
    at WorkflowScript.run(WorkflowScript:33)
at ___cps.transform___(Native Method)

... Code Snipped ...
{% endhighlight %}

进入 Manage Jenkins => In-process Script Approval，会看到如下提示：
![in-process-script-approval](/assets/201612/in-process-script-approval.png)

点击 Approve 按钮便可解决问题前面的异常。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Pipeline: How to add an input step, with timeout, that continues if timeout is reached, using a default value][ref1]<br>
2 [Jenkins CI Pipeline Scripts not permitted to use method groovy.lang.GroovyObject][ref2]
</span>

[ref1]: https://support.cloudbees.com/hc/en-us/articles/226554067-Pipeline-How-to-add-an-input-step-with-timeout-that-continues-if-timeout-is-reached-using-a-default-value
[ref2]: http://stackoverflow.com/questions/38276341/jenkins-ci-pipeline-scripts-not-permitted-to-use-method-groovy-lang-groovyobject
