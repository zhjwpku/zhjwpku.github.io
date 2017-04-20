---
layout: post
title: Nginx 日志格式简介
date: 2017-04-20 12:00:00 +0800
tags:
- nginx
- logrotate
---

Nginx 预定义的 log_format 名字为 combined，有时候它不能满足需求，因此根据需求定义我们需要的日志格式。

`combined` 的格式为：

{% highlight shell %}
# 注意，自己不能定义名为combined的log_format
log_format combined '$remote_addr - $remote_user [$time_local]  '
        '"$request" $status $body_bytes_sent '
        '"$http_referer" "$http_user_agent"';
{% endhighlight %}

**参数解释**

    $remote_addr          客户端地址
    $remote_user          客户端用户名
    $time_local           访问时间和时区
    $request              请求类型及请求的请求的url
    $status               响应状态
    $body_bytes_sent      响应数据大小
    $http_referer         url跳转来源
    $http_user_agent      客户端浏览器信息

对 combined 进行扩展就可以获得我们需要的日志格式，如：

{% highlight shell %}
# 定义日志格式main
log_format main '$remote_addr - $remote_user [$time_local]  '
        '"$request" $status $body_bytes_sent '
        '"$http_referer" "$http_user_agent" '
        'rt=$request_time urt=$upstream_response_time';

# 指定日志格式
access_log /var/log/nginx/access.log main;
{% endhighlight %}

**参数解释**

    $request_time           Nginx处理请求所用总时间
    $upstream_connect_time  Nginx与upstream server建立连接所用时间
    $upstream_header_time   从建立连接成功到接收第一个字节之间的时间
    $upstream_response_time 从连接upstream成功到接收upstream返回最后一个字节所用总时间

可以看出：

    $request_time = $upstream_connect_time +
                    $upstream_header_time +
                    $upstream_response_time

但版本较老的 Nginx 版本不支持 $upstream_connect_time 和 $upstream_header_time，因此上面的 main 只定义了 $request_time 和 $upstream_response_time。

日志的结果见下图：

![access_log](/assets/201704/access_log.png)

<h4>Log Rotation</h4>

日志的重要性怎么强调都不为过，通过日志可以分析用户行为，查找程序问题。但日志会占用磁盘，有时候我们不得不进行日志滚动。有些人喜欢写脚本进行处理，但 [logrotate][logrotate] 应该能做的更好。

一般的 Linux 发行版都安装了 logrotate 这个工具，而且自动配置了 cron.daily 任务：

{% highlight shell %}
# ubuntu
ubuntu@ip-172-31-1-163:~$ cat /etc/cron.daily/logrotate
#!/bin/sh

# Clean non existent log file entries from status file
cd /var/lib/logrotate
test -e status || touch status
head -1 status > status.clean
sed 's/"//g' status | while read logfile date
do
    [ -e "$logfile" ] && echo "\"$logfile\" $date"
done >> status.clean
mv status.clean status

test -x /usr/sbin/logrotate || exit 0
/usr/sbin/logrotate /etc/logrotate.conf

# centos
→ ~ # cat /etc/cron.daily/logrotate
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
{% endhighlight %}

也就是说，每天会运行一次 /usr/sbin/logrotate，/etc/logrotate.conf 中包含了 /etc/logrotate.d/ 下的所有配置文件，所以只要在 /etc/logrotate.d/ 目录下放置需要回滚日志的程序配置即可使用 logrotate 强大的功能。

很多程序在安装的时候会自动创建日志回滚配置文件，如 Nginx 安装后会创建 /etc/logrotate.d/nginx:

{% highlight shell %}
ubuntu@ip-172-31-1-163:~$ cat /etc/logrotate.d/nginx
/var/log/nginx/*.log {
	weekly
	missingok
	rotate 52
	compress
	delaycompress
	notifempty
	create 0640 www-data adm
	sharedscripts
	prerotate
		if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
			run-parts /etc/logrotate.d/httpd-prerotate; \
		fi \
	endscript
	postrotate
		[ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
	endscript
}
{% endhighlight %}

关于配置的具体说明运行 `man logrotate` 查看即可。需要强调的是，以下片段在 Rotation 完成后告诉 Nginx 重载日志文件。

    postrotate
      [ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
    endscript


推荐阅读： [logrotate机制和原理][ref2]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [CONFIGURING LOGGING][ref1]<br>
2 [HowTo: The Ultimate Logrotate Command Tutorial with 10 Examples][ref3]<br>
3 [How To Configure Logging and Log Rotation in Nginx on an Ubuntu VPS][ref4]
</span>

[ref1]: https://www.nginx.com/resources/admin-guide/logging-and-monitoring/
[logrotate]: https://github.com/logrotate/logrotate
[ref2]: http://blog.lightxue.com/how-logrotate-works/
[ref3]: http://www.thegeekstuff.com/2010/07/logrotate-examples/
[ref4]: https://www.digitalocean.com/community/tutorials/how-to-configure-logging-and-log-rotation-in-nginx-on-an-ubuntu-vps
