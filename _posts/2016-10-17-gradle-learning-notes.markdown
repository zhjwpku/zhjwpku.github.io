---
layout: post
title: Gradle学习记录
date: 2016-10-17 12:00:00 +0800
tags:
- gradle
- CI
---

[Gradle][gradle]堪称Java（JVM）世界构建技术的一个量子跃迁。本文介绍了Gradle的常用概念，并通过对Gradle项目（使用Gradle）自身构建过程的分析进一步了解Gradle。

Gradle提供：

- 类似Ant、灵活、通用的构建工具
- 可切换，build-by-convention（个人理解为可通过不同的命令进行不同的构建）
- 多项目构建（multi-project）的支持
- 强大的依赖管理（基于Apache Ivy）
- 支持已存在的Maven或Ivy仓库
- 支持传递依赖关系管理，而不需要远程存储库或pom.xml和ivy.xml文件
- Ant任务和构建是一等公民
- Groovy构建脚本
- 丰富的用于描述构建的域模型（Domain Specific Language）
- Gradle wrapper允许用户在没有安装Gradle的机器上执行Gradle builds, 这对于CI非常有用

<h4>安装Gradle</h4>

Gradle依赖于Java JDK或JRE，版本>=7。Gradle本身包含了Groovy库，已安装的Groovy会被Gradle忽略。下面使用[SDKMAN][sdkman]进行安装：

{% highlight shell %}
# 安装sdkman
$ curl -s "https://get.sdkman.io" | bash
$ source "$HOME/.sdkman/bin/sdkman-init.sh"

# 安装groovy，这一步不需要
$ sdk install groovy

# 安装gradle，版本3.1
$ sdk install gradle 3.1
{% endhighlight %}

<h4>Gradle Command-Line</h4>

任务之间有依赖关系，如下图所示：

![commandLineTutorialTasks](/assets/201610/commandLineTutorialTasks.png)

build.gradle文件内容如下：
{% highlight groovy %}
task comile << {
    println 'compiling source'
}

task compileTest(dependsOn: compile) << {
    println 'compiling unit tests'
}

task test(dependsOn: [compile, compileTest]) << {
    println 'running unit tests'
}

task dist(dependsOn: [compile, test]) << {
    println 'building the distribution'
}
{% endhighlight %}

当执行`gradle dist test`的时候，`compile`只执行一次，执行`gradle test test`，`test`也只执行一次。

{% highlight shell %}
# Excluding tasks, 任务test不会执行
$ gradle dist -x test

# Continue the build when a failure occurs
# 如果一个任务的依赖都没有出错，那么--continue会保证该任务一定会被执行，如果它的依赖失败了，则该任务不会执行
$ gradle dist --continue

# 任务名简写，可以使用驼峰
# same as `gradle dist`
$ gradle di

# same as `gradle comTest`
$ gradle cT

# gradle默认会查找在当前文件夹下的构建文件，可以使用-b选项来选择另一个build file，这样setting.gradle不会被使用
# -q, --quite    Log errors only
$ gradle -q -b subdir/myproject.gradle hello

# -p 选择项目目录
$ gradle -q -p subdir hello

# Forcing tasks to execute, 会让所有dist依赖的task强制执行
$ gradle --rerun-tasks dist

# Listing projects
$ gradle -q projects

# Listing tasks
$ gradle -q tasks
$ gradle -q tasks --all

# Show task usage detail
$ gradle -q help --task libs

# Listing project dependencies
$ gradle -q dependencies api:dependencies webapp:dependencies

# Listing project properties
$ gradle -q api:properties
{% endhighlight %}

<h4><a href="https://docs.gradle.org/current/userguide/gradle_wrapper.html">Gradle Wrapper</a></h4>

Gradle建议所有的Gradle项目设置`Wrapper`，它可以省去Gradle的安装并且保证使用正确的工具版本。每个`Wrapper`都与一个特定的Gradle版本绑定，第一次执行`./gradlew <task>`时，Wrapper会下载对应版本的Gradle（保存在~/.gradle/wrapper/dists/目录下）并使用它来进行构建。所以，使用`./gradlew`替换上述命令的`gradle`，效果是一样的。

**Adding the `Wrapper` to a Project**

Wrapper应该使用Version Control来管理，这就意味着任何人都可以clone你的repo，进行构建却不用安装Gradle。

{% highlight shell %}
# 在项目根目录运行
$ gradle wrapper --gradle-version 3.1
:wrapper

BUILD SUCCESSFUL

Total time: 4.507 secs
{% endhighlight %}

该命令会生成以下文件：

- .gradle
- gradlew (Unix Shell script)
- gradlew.bat (Windows batch file)
- gradle/wrapper/gradle-wrapper.jar (Wrapper JAR)
- gradle/wrapper/gradle-wrapper.properties (Wrapper properties)

`.gradle`是一个临时文件夹，不同的构建环境会导致其下的内容会频繁变化，因此需要将它排除在代码管理工具中，即在.gitignore中添加.gradle。构建过程中生成的.class文件及Jar包等都会存放到各个项目的`build`文件夹下，所以也需要将它写入.gitignore。

<h4><a href="https://docs.gradle.org/current/userguide/artifact_dependencies_tutorial.html">依赖管理</a></h4>

在Ant中，只能通过说明依赖jar包的绝对路径或相对路径，而Gradle只需要定义依赖包的名称，而由其它层来确定依赖包的具体位置。相同的特性Ant可以通过Apache Ivy来实现，但是Gradle做的更好。依赖本身也会有依赖的包，这叫做传递依赖`transitive dependencies`，这对Gradle来说不成问题。Gradle还可以将构建的包上传到远程的Maven或Ivy仓库，这叫做`Publishing`。

**`build.gradle文件`**

{% highlight groovy %}
apply plugin: 'java'
apply plugin: 'maven'

// maven仓库
repositories {
    // Use a Maven central repoitory
    mavenCentral()

    // Use a remote maven repoitory
    maven {
        url 'https://repo.gradle.org/gradle/libs-releases'
    }
}

// External dependencies
dependencies {
    compile group: 'org.hibernate', name: 'hibernate-core', version: '3.6.7.Final'
    testCompile group: 'junit', name: 'junit', version: '4.+'
    deployerJars "org.apache.maven.wagon:wagon-ssh:2.2"
}

// [Interacting with Maven repositories][maven-repo]
UploadArchives {
    repositories {
        mavenDeployer {
            configuration = configurations.deployerJars
            repository(url: "scp://repos.mycompany.com/releases") {
                authentications(userName: "me", password: "myPassword")
            }
        }
    }
}
{% endhighlight %}

Gradle的依赖被分组到不同的`configurations`。每个`configuration`都是简单的依赖名字集合，用于定义项目的外部依赖。

Java Plugin定义了一组标准的configurations，这些configuration代表了Java plugin使用的classpaths。除了上述的`compile`，`testCompile`，常见的还有`runtime`，`testRuntime`。

<h4><a href="https://docs.gradle.org/current/userguide/intro_multi_project_builds.html">multi-project build</a></h4>

一个multi-project build的结构通常都包含如下部分：

- 根文件夹中的`setting.gradle`
- 根文件夹中的`build.gradle`
- 各子项目文件夹包含自己的`*.gradle`（有些multi-project builds会忽略子项目的构建脚本）

可以使用子项目的文件夹名命名gradle构建脚本，或者更简单的方式是将所有的子项目的构建文件统一命名为`build.gradle`。

<h4><a href="https://docs.gradle.org/current/userguide/pt03.html">Writing Gradle build scripts</a></h4>

*从这一节开始，是Gradle的重点*

Gradle中的所有东西都基于两个基本的概念：`projects`和`tasks`，每一个Gradle项目由一个或多个`project`构成。

定义`task`

{% highlight shell %}
task hello {
    doLast {
        println 'Hello world!'
    }
}

# 使用shortcut定义
task hello << {
    println 'Hello world!'
}

# enhanced tasks
tasks.create(name: 'hello')

# print task name
println hello.name
println project.hello.name
println tasks.hello.name
println tasks['hello'].name
{% endhighlight %}

**Some useful definition**

{% highlight groovy %}
// 定义默认任务
defaultTasks 'clean', 'run'

task clean << {
    println 'Default Cleaning!'
}

// gradle -q distribution
task distribution << {
    println "We build the zip with version=$version"
}

// gradle -q release
task release(dependsOn: 'distribution') << {
    println 'We release now'
}

gradle.taskGraph.whenReady { taskGraph ->
    if (taskGraph.hasTask(release)) {
        version = '1.0'
    } else {
        version = '1.0-SNAPSHOT'
    }
}

task taskX << {
    println 'taskX'
}

task taskY << {
    println 'taskY'
}

taskX.dependsOn taskY

taskX.dependsOn {
    tasks.findAll {task -> task.name.startsWith('libs')}
}

task lib1 << {
    println 'lib1'
}

task lib2 << {
    println 'lib2'
}

// Ordering tasks
taskY.mustRunAfter taskX
taskY.shouldRunAfter taskX

// Multi-project
allprojects {
    task hello << { task -> println "I'm $task.project.name"}
}

subprojects {
    hello << {print "- I denpend on water"}
}

project(':bluewhale').hello << { task ->
    println "I'am subproject $task.project.name"
}
{% endhighlight %}

**Up-to-data checks (增量构建)**

增量构建避免了对同一个task重复执行，一旦你的源码被编译过了，那么在没有编写改变影响输出的代码前，不应该重新编译。

定义incremental task action

build.gradle
{% highlight ruby %}
class IncrementalReverseTask extends DefaultTask {
    @InputDirectory
    def File inputDir

    @OutputDirectory
    def File outputDir

    @Input
    def inputProperty

    @TaskAction
    void execute(IncrementalTaskInputs inputs) {
        println inputs.incremental ? "CHANGE inputs considered out of data"
                                   : "All inputs considered out of data"
        if (!inputs.incremental)
            project.delete(outputDir.listFiles())

        inputs.outOfDate { change ->
            println "out of data: ${change.file.name}"
            def targetFile = new File(outputDir, change.file.name)
            targetFile.text = change.file.test.reverse()
        }

        inputs.removed { change ->
            println "removed: ${change.file.name}"
            def targetFile = new File(outputDir, change.file.name)
            targetFile.delete()
        }
    }
}

// An incremental task in action
task incrementalReverse(type: IncrementalReverseTask) {
    inputDir = file('inputs')
    outputDir = file("$buildDir/outputs")
    inputProperty = project.properties['taskInputProperty'] ?: "original"
}
{% endhighlight %}

**<a href="https://docs.gradle.org/current/userguide/tutorial_java_projects.html">Java Plugin</a>**

这个插件自动向项目中添加一些任务来对Java源码编译和单元测试，并将它打包成JAR文件。常用的命令有：
{% highlight shell %}
# build and the tasks
$ gradle build
:compileJava
:processResources
:classes
:jar
:assemble
:compileTestJava
:processTestResources
:testClasses
:test
:check
:build

$ gradle clean

$ gradle assemble

$ gradle check
{% endhighlight %}

<h4><a href="https://gradle.org/migrating-a-maven-build-to-gradle/">将一个Maven项目迁移到Gradle</a></h4>

在项目根目录下执行如下命令，亲测很强大，稍改一下就可以build了。
{% highlight shell %}
$gradle init
{% endhighlight %}
该命令会逐一解析存在的POM文件并生成Gradle构建文件以及包含muilt-project各项目信息的settings.gradle文件。

需要注意的是assemblies并不会自动转换，需要手动编写。这需要借助到**[Application Plugin][applicationplugin]**。

如下是我写的gradle构建文件，仅供参考。提醒一点，拷贝文件时的filter对@someParam@有效，而不是${someParam}

[build.gradle][buildgradle]

<h4><a href="https://docs.gradle.org/current/userguide/publishing_maven.html">Maven Publish</a></h4>
{% highlight groovy %}
apply plugin: 'maven-publish'
apply plugin: 'application'

def deployUsername = hasProperty("MAVEN_USERNAME") ? MAVEN_USERNAME : "your_repo_username"
def deployPassword = hasProperty("MAVEN_PASSWORD") ? MAVEN_PASSWORD : "your_repo_password"

publishing {
    // 上传位置

    repositories {
        maven {
            url "$artifactRepoBase/${project.version.endsWith('-SNAPSHOT')? 'snapshots':'releases'}"
            credentials {
                username deployUsername
                password deployPassword
            }
        }
    }

    // 发布
    publications {
        // 上传Jar包
        //mavenJava(MavenPublication) {
        //    from components.java
        //}

        pubZip(MavenPublication) {
            groupId 'com.startimestv.server.upms'
            artifactId "upms-${env}"
            version '3.11-SNAPSHOT'

            artifact distZip
        }
    }

}
{% endhighlight %}

上传命令：`./gradlew publish`

<h4>Another Gradle project</h4>

[https://github.com/zhjwpku/ijkplayer/tree/master/android/ijkplayer][gradle_project]

gradle的构建我不想写了，自己看吧...

-- Updated 2016-11-21 --<br>
**Build C/C++**<br>
> [Gradle and C/C++ Workshop Part 1][part1]<br>
> [Gradle and C/C++ Workshop Part 2][part2]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Gradle User Guide][userguide]<br>
2 [Building and testing with Gradle](/assets/pdf/Building_and_testing_with_Gradle.pdf)<br>
3 [Gradle Beyond the Basics](/assets/pdf/Gradle_Beyond_the_Basics.pdf)
</span>

[userguide]: https://docs.gradle.org/current/userguide/userguide.html
[gradle]: https://github.com/gradle/gradle
[sdkman]: http://sdkman.io/
[maven-repo]: https://docs.gradle.org/current/userguide/maven_plugin.html#uploading_to_maven_repositories
[applicationplugin]: https://docs.gradle.org/current/userguide/application_plugin.html
[buildgradle]: https://gist.github.com/zhjwpku/a4ab0e9674432ae09206426bf2dfcf59
[gradle_project]: https://github.com/zhjwpku/ijkplayer/tree/master/android/ijkplayer
[part1]: https://www.youtube.com/watch?v=m3t4JKHrGAk
[part2]: https://www.youtube.com/watch?v=Z4QuY-PFC3Y
