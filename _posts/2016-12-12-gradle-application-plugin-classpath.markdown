---
layout: post
title: Gradle Application Plugin classpath problem
date: 2016-12-12 08:00:00 +0800
tags:
- gradle
---

使用 Gradle 的 [Application Plugin][application_plugin] 打包生成可以用来执行 Java 程序的脚本时遇到一个问题，当使用`startScripts.classpath.add(files('$APP_HOME/conf'))`将配置文件加入到 classpath 的时候，生成的启动脚本中的入口却是`$APP_HOME/lib/conf`。

解决这个问题的一个办法是将 conf 文件夹放置于发布包的 lib 目录下：

{% highlight groovy %}
// 向发布的包增加额外文件 
distributions {
    main {
        contents {
            from(copyConfFiles) {
                into "lib/conf"
            }   
        }   
    }   
}
{% endhighlight %}

这种办法由于目录结构不合理被领导否认了。最终使用的办法是将启动脚本中的入口进行文本替换：

{% highlight groovy %}
// 向发布的包增加额外文件 
distributions {
    main {
        contents {
            from(copyConfFiles) {
                into "conf"
            }   
        }   
    }   
}

// 添加额外的配置到classpath
startScripts {
    classpath += files('$APP_HOME/conf')
        doLast {
            def windowsScriptFile = file getWindowsScript()
            def unixScriptFile = file getUnixScript()
            windowsScriptFile.text = windowsScriptFile.text.replace('%APP_HOME%\\lib\\conf', '%APP_HOME%\\conf')
            unixScriptFile.text = unixScriptFile.text.replace('$APP_HOME/lib/conf', '$APP_HOME/conf')
        }   
}
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [Adding classpath entries using Gradle's Application plugin][ref1]<br>
2. [Classpath in Application plugin is building always relative to %APP_HOME%/lib directory][ref2]
</span>

[application_plugin]: https://docs.gradle.org/current/userguide/application_plugin.html
[ref1]: http://stackoverflow.com/questions/10518603/adding-classpath-entries-using-gradles-application-plugin
[ref2]: https://discuss.gradle.org/t/classpath-in-application-plugin-is-building-always-relative-to-app-home-lib-directory/2012
