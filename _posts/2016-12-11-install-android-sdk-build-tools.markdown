---
layout: post
title: 命令行安装 Android SDK Build Tools
date: 2016-12-11 20:00:00 +0800
tags:
- android-sdk
- build-tools
---

在 Jenkins 环境对 Android 项目进行构建有时会遇到缺少相应 SDK 或 buildTools 版本的问题。如 [countly-sdk-android][countly-sdk-android] sdk 子项目的 build.gradle 文件包含如下内容：

{% highlight groovy %}
android {
    compileSdkVersion 24
    buildToolsVersion "24.0.2"

    defaultConfig {
        minSdkVersion 9
        targetSdkVersion 24
        versionCode 1
        versionName "16.06.04"

        testInstrumentationRunner 'ly.count.android.sdk.test.InstrumentationTestRunner'
        testHandleProfiling true
        testFunctionalTest true
    }   
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }   
    }   

    lintOptions {
        abortOnError false
    }   
}
{% endhighlight %}

在 Jenkins 中我们不希望安装 Android Studio 来解决这样的版本问题。可以使用 Android SDK 提供的`android`命令来下载缺少的版本。

首先，在 /etc/bashrc 文件下添加如下内容，使得在命令行可以执行`android`命令：

{% highlight shell %}
export ANDROID_HOME=/usr/lib/android-sdk
export PATH=$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/tools
{% endhighlight %}

然后使用如下命令进行安装：

{% highlight shell %}
$ android list sdk --all
Refresh Sources:
  Fetching https://dl.google.com/android/repository/addons_list-2.xml
  Validate XML
  Parse XML
  Fetched Add-ons List successfully
  Refresh Sources
  Fetching URL: https://dl.google.com/android/repository/repository-11.xml
  Validate XML: https://dl.google.com/android/repository/repository-11.xml
  Parse XML:    https://dl.google.com/android/repository/repository-11.xml
  Fetching URL: https://dl.google.com/android/repository/addon.xml
  Validate XML: https://dl.google.com/android/repository/addon.xml
  Parse XML:    https://dl.google.com/android/repository/addon.xml
  Fetching URL: https://dl.google.com/android/repository/glass/addon.xml
  Validate XML: https://dl.google.com/android/repository/glass/addon.xml
  Parse XML:    https://dl.google.com/android/repository/glass/addon.xml
  Fetching URL: https://dl.google.com/android/repository/extras/intel/addon.xml
  Validate XML: https://dl.google.com/android/repository/extras/intel/addon.xml
  Parse XML:    https://dl.google.com/android/repository/extras/intel/addon.xml
  Fetching URL: https://dl.google.com/android/repository/sys-img/android/sys-img.xml
  Validate XML: https://dl.google.com/android/repository/sys-img/android/sys-img.xml
  Parse XML:    https://dl.google.com/android/repository/sys-img/android/sys-img.xml
  Fetching URL: https://dl.google.com/android/repository/sys-img/android-wear/sys-img.xml
  Validate XML: https://dl.google.com/android/repository/sys-img/android-wear/sys-img.xml
  Parse XML:    https://dl.google.com/android/repository/sys-img/android-wear/sys-img.xml
  Fetching URL: https://dl.google.com/android/repository/sys-img/android-tv/sys-img.xml
  Validate XML: https://dl.google.com/android/repository/sys-img/android-tv/sys-img.xml
  Parse XML:    https://dl.google.com/android/repository/sys-img/android-tv/sys-img.xml
  Fetching URL: https://dl.google.com/android/repository/sys-img/google_apis/sys-img.xml
  Validate XML: https://dl.google.com/android/repository/sys-img/google_apis/sys-img.xml
  Parse XML:    https://dl.google.com/android/repository/sys-img/google_apis/sys-img.xml
Packages available for installation or update: 177
   1- Android SDK Tools, revision 25.2.3
   2- Android SDK Platform-tools, revision 25.0.1
   3- Android SDK Build-tools, revision 25.0.1
   4- Android SDK Build-tools, revision 25
   5- Android SDK Build-tools, revision 24.0.3
   6- Android SDK Build-tools, revision 24.0.2
   7- Android SDK Build-tools, revision 24.0.1
   8- Android SDK Build-tools, revision 24
   9- Android SDK Build-tools, revision 23.0.3
  10- Android SDK Build-tools, revision 23.0.2
  11- Android SDK Build-tools, revision 23.0.1
  12- Android SDK Build-tools, revision 23 (Obsolete)
  13- Android SDK Build-tools, revision 22.0.1
  14- Android SDK Build-tools, revision 22 (Obsolete)
  15- Android SDK Build-tools, revision 21.1.2
  16- Android SDK Build-tools, revision 21.1.1 (Obsolete)
  17- Android SDK Build-tools, revision 21.1 (Obsolete)
  18- Android SDK Build-tools, revision 21.0.2 (Obsolete)
  19- Android SDK Build-tools, revision 21.0.1 (Obsolete)
  20- Android SDK Build-tools, revision 21 (Obsolete)
  21- Android SDK Build-tools, revision 20
  22- Android SDK Build-tools, revision 19.1
  23- Android SDK Build-tools, revision 19.0.3 (Obsolete)
  24- Android SDK Build-tools, revision 19.0.2 (Obsolete)
  25- Android SDK Build-tools, revision 19.0.1 (Obsolete)
  26- Android SDK Build-tools, revision 19 (Obsolete)
  27- Android SDK Build-tools, revision 18.1.1 (Obsolete)
  28- Android SDK Build-tools, revision 18.1 (Obsolete)
  29- Android SDK Build-tools, revision 18.0.1 (Obsolete)
  30- Android SDK Build-tools, revision 17 (Obsolete)
  31- Documentation for Android SDK, API 24, revision 1
  32- SDK Platform Android 7.1.1, API 25, revision 2
  33- SDK Platform Android 7.0, API 24, revision 2
  34- SDK Platform Android 6.0, API 23, revision 3
  35- SDK Platform Android 5.1.1, API 22, revision 2
  36- SDK Platform Android 5.0.1, API 21, revision 2
  ... Code Snipped ...

# 安装相应的包选项
# -u stands for --no-ui, -a stands for --all, -t stands for --filter
# 6 代表 Android SDK Build-tools, revision 24.0.2
# 33代表 SDK Platform Android 7.0, API 24, revision 2
$ android update sdk -u -a -t 6,33
{% endhighlight %}

*注：运行完后一定检查相应位置是否存在，如不存在就再运行一次*

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [How to install Android SDK Build Tools on the command line?][ref1]
</span>

[countly-sdk-android]: https://github.com/Countly/countly-sdk-android
[ref1]: http://stackoverflow.com/questions/17963508/how-to-install-android-sdk-build-tools-on-the-command-line
