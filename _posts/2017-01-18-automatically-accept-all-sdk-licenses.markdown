---
layout: post
title: Automatically accept all SDK licences
date: 2017-01-18 12:00:00 +0800
tags:
- android-sdk
---

Android Gradle Plugin 更新到 2.2.1 之后，在执行 gradle 任务时会尝试下载缺少的项目依赖的 SDK 模块，但下载 SDK 模块需要接受证书，造成 gradle 报错。

{% highlight shell %}
→ ~/android (master) $ ./gradlew tasks
File /home/zhjwpku/.android/repositories.cfg could not be loaded.

FAILURE: Build failed with an exception.

* What went wrong:
A problem occurred configuring project ':com.star.mobile.video'.
> A problem occurred configuring project ':com.star.share'.
   > A problem occurred configuring project ':facebook_sdk'.
      > A problem occurred configuring project ':com.star.util'.
         > You have not accepted the license agreements of the following SDK components:
           [SDK Patch Applier v4, Google Repository, Android Support Repository].
           Before building your project, you need to accept the license agreements and complete the installation of the missing components using the Android Studio SDK Manager.
           Alternatively, to learn how to transfer the license agreements from one workstation to another, go to http://d.android.com/r/studio-ui/export-licenses.html

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED
{% endhighlight %}

可以看出，平台缺少 `SDK Patch Applier v4`、`Google Repository`、`Android Support Repository`三个 SDK 模块。可以手动安装：

{% highlight shell %}
→ ~ $ android list sdk --all
...
166- Android Support Repository, revision 41
...
173- Google Repository, revision 41
...

## 安装
→ ~ $ android update sdk -u -a -t 166,173

...
November 20, 2015
Do you accept the license 'android-sdk-license-c81a61d9' [y/n]: y
...

## 可能需要
→ ~ $ android update sdk --no-ui --filter extra-android-m2repository
{% endhighlight %}

在其中的交互中输入 y 接受证书，至于为什么没有安装 `SDK Patch Applier v4` 见 [Ref3][ref3]。

另一种方式是提前将证书文本的的 sha1 存储在 $ANDROID_HOME/licenses文件夹下：

{% highlight shell %}
mkdir -p "$ANDROID_SDK/licenses"
echo -e "\n8933bad161af4178b1185d1a37fbf41ea5269c55" > "$ANDROID_SDK/licenses/android-sdk-license"
echo -e "\n84831b9409646a918e30573bab4c9c91346d8abd" > "$ANDROID_SDK/licenses/android-sdk-preview-license"
{% endhighlight %}

但这个哈希会随时间变更，不推荐这种方式。手动装也不是很费事 :)

See also: [Ref4][ref4]，`extra-google-m2repository` 可能会解决自动接受证书的问题，我没试 :(

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [命令行安装 Android SDK Build Tools][ref1]<br>
2. [Automatically accept all SDK licences][ref2]<br>
3. [what is SDK Patch Applier in android SDK Tool?][ref3]<br>
4. [Travis Ci build error caused by Android SDK license agreements][ref4]
</span>

[ref1]: http://zhjwpku.com/2016/12/11/install-android-sdk-build-tools.html
[ref2]: http://stackoverflow.com/questions/38096225/automatically-accept-all-sdk-licences
[ref3]: http://stackoverflow.com/questions/38527793/what-is-sdk-patch-applier-in-android-sdk-tool
[ref4]: http://stackoverflow.com/questions/40057865/travis-ci-build-error-caused-by-android-sdk-license-agreements
