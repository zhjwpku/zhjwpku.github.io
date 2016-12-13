---
layout: post
title: Gradle with Flyway
date: 2016-12-13 18:00:00 +0800
tags:
- gradle
- flyway
---

[Flyway][flyway] 用于管理数据库的版本，它使得数据库可以随着代码的变动而变动，从而让数据库的管理更加方便。Flyway 的使用方式多种多样，本文介绍在 Gradle 使用 Flyway 来管理数据库表。

**直接在 build 脚本中配置**

{% highlight groovy %}
flyway {
    url = 'jdbc:mysql://localhost:3306?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&useSSL=false'
    user = 'myUsr'
    password = 'mySecretPwd'
    schemas = ['schema1', 'schema2', 'schema3']

    // 禁止flywayClean，在生产环境中非常重要
    cleanDisabled = true

    placeholders = [ 
        'keyABC': 'valueXYZ',
        'otherplaceholder': 'value123'
    ]   

    // 指定数据库Schema的位置
    locations = ['filesystem:com.example.abc.migration/sqls']
}
{% endhighlight %}

**通过命令行参数**

{% highlight shell %}
./gradlew -Pflyway.user=myUsr -Pflyway.schemas=schema1,schema2 -Pflyway.placeholders.keyABC=valXYZ
{% endhighlight %}

**在 gradle.properties 文件配置**

{% highlight shell %}
flyway.user=myUser
flyway.password=mySecretPwd
flyway.locations=filesystem:com.example.abc.migration/sqls

# List are defined as comma-separated values
flyway.schemas=schema1,schema2,schema3

# Individual placeholders are prefixed by flyway.placeholders.
flyway.placeholders.keyABC=valueXYZ
flyway.placeholders.otherplaceholder=value123
{% endhighlight %}

数据库表管理工具还有一个 [MyBatis Migrations][mybatis_migrations]，在 Gradle 中集成可使用 [Gradle Migrations Plugin][gradle-migrations-plugin]。

[flyway]: https://flywaydb.org/
[mybatis_migrations]: http://www.mybatis.org/migrations/
[gradle-migrations-plugin]: https://github.com/marceloemanoel/gradle-migrations-plugin
