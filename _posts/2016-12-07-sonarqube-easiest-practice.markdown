---
layout: post
title: SonarQube Easiest Practice
date: 2016-12-07 17:15:00 +0800
tags:
- sonarqube
- gradle
---

[SonarQube][sonarqube] 是一个开源的代码质量管理平台，覆盖了代码质量的7个维度：Potential bugs，Complexity，Unit tests、Dupliactions，Architecture & Design、Comments、Coding rules。由于本人刚接触SonarQube，本文仅介绍SonarQube平台的搭建及在Gradle构建脚本中使用sonarqube插件的简单实践，更深入的使用方法请参考[SonarQube in Action](/assets/pdf/SonarQube_in_Action.pdf)。

<h4>安装MySQL</h4>

CentOS 7默认的仓库不包含MySQL数据库索引（默认为MariaDB），虽然MariaDB和MySQL的上层接口一致，但实践证实，SonarQube的确不支持MariaDB，于是这里介绍一下安装MySQL的过程。

{% highlight shell %}
$ wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
$ sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
$ sudo yum install mysql-server

# Start MySQL Server
$ sudo systemctl enable mysqld
$ sudo systemctl start mysqld

# Reset MySQL root password
$ mysql_secure_installation
{% endhighlight %}

<h4>安装SonarQube</h4>

SonarQube最新的LTS版本为5.6.3，需要系统上安装OpenJDK 8，在笔者的“[同一系统安装多个Java版本][diffjava]”一文中对Java环境的配置做了介绍，此处不赘述。

**创建SonarQube数据库账户**

{% highlight shell %}
# 首先使用root账户连接到MySQL数据
$ mysql -u root -p
Enter password:
...
# 然后使用如下命令创建sonar账户
mysql> CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci;
mysql> CREATE USER 'sonar' IDENTIFIED BY 'sonar';
mysql> GRANT ALL ON sonar.* TO 'sonar'@'%' IDENTIFIED BY 'sonar';
mysql> GRANT ALL ON sonar.* TO 'sonar'@'localhost' IDENTIFIED BY 'sonar';
mysql> FLUSH PRIVILEGES;
{% endhighlight %}

**安装SonarQube**

{% highlight shell %}
$ wget https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-5.6.3.zip
$ sudo unzip -d /usr/share sonarqube-5.6.3.zip
$ sudo ln -s /usr/share/sonar-5.6.3 /usr/bin/sonar
$ sudo ln -s /usr/bin/sonar/bin/linux-x86-64/sonar.sh /etc/init.d/sonar
{% endhighlight %}

**配置并启动**

编辑`/usr/share/sonar-5.6.3/conf/sonar.properties`文件，如下：
{% highlight shell %}
# User credentials.
# Permissions to create tables, indices and triggers must be granted to JDBC user.
# The schema must be created first.
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar

#----- MySQL 5.6 or greater
# Only InnoDB storage engine is supported (not myISAM).
# Only the bundled driver is supported. It can not be changed.
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
{% endhighlight %}

启动SonarQube：

{% highlight shell %}
$ sudo service sonar start
{% endhighlight %}

<h4>Gradle and SonarQube</h4>

编辑项目的`build.gradle`文件，添加如下内容：

{% highlight groovy %}
plugins {
    id "org.sonarqube" version "2.2"
}

sonarqube {
    properties {
        // 更多配置参考Reference 1
        property "sonar.projectName", project.name
        property "sonar.projectKey", "$project.group:$project.name"
        property "sonar.sourceEncoding", "UTF-8"
    }   
}
{% endhighlight %}

添加`gradle.properties`文件，内容如下：

{% highlight shell %}
# SonarQube服务器地址
systemProp.sonar.host.url=http://10.0.63.202:9000

# 下面的配置已经弃用了，即使加上也会被忽略
#systemProp.sonar.jdbc.url=sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
#systemProp.sonar.jdbc.username=sonar
#systemProp.sonar.jdbc.password=sonar

# 如果SonarQube设置了 Force user authentication，则需要提供用户名密码将结果同步给SonarQube Server
systemProp.sonar.login=admin
systemProp.sonar.password=admin
{% endhighlight %}

<h4>LDAP 认证配置</h4>

在 Update Center 下载下载 LDAP 插件并配置`/usr/share/sonar-5.6.3/conf/sonar.properties`，添加如下内容：

{% highlight shell %}
# LDAP Configuration
sonar.security.realm=LDAP
sonar.security.savePassword=true
sonar.security.localUsers=admin
ldap.url=ldap://xxx.xxx.xxx.xxx:389
ldap.bindDn=uid=root,cn=users,dc=example,dc=com
ldap.bindPassword=secret

# User Configuration
ldap.user.baseDn=cn=users,dc=startimes,dc=me
# shodowExpire=-1 用来禁止失效用户登陆
ldap.user.request=(&(uid={login})(memberof=cn=gitlab-users,cn=groups,dc=example,dc=com)(shadowExpire=-1))
# sAMAccountName 用于 AD登陆
#ldap.user.request=(&(sAMAccountName={login})(memberof=cn=gitlab-users,cn=groups,dc=example,dc=com))
ldap.user.realNameAttribute=cn
ldap.user.emailAttribute=mail

# Group Configuration
#ldap.group.baseDn=cn=groups,dc=example,dc=com
#ldap.group.request=(&(memberUid={uid}))
#ldap.group.idAttribute=cn
{% endhighlight %}

<h4>Email 配置</h4>

Administration => General Settings => Email

几个值得注意的问题：

- `SMTP username`需要与`From address`相同
- `User secure connection`可能是`plaintext`

以上问题都可通过设置`sonar.log.level=DEBUG`来查看，日志在 /usr/share/sonar-5.6.3/logs/sonar.log 中。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Analyzing with SonarQube Scanner for Gradle][analyzesonar]<br>
2 [Sonar Examples][sonar-examples]<br>
3 [LDAP Plugin][ldap]
</span>

[analyzesonar]: http://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Gradle
[sonarqube]: http://www.sonarqube.org/
[sonar-examples]: https://github.com/SonarSource/sonar-examples
[diffjava]: http://zhjwpku.com/2016/12/03/different-java-version-on-same-os.html
[ldap]: http://docs.sonarqube.org/display/SONARQUBE45/LDAP+Plugin
