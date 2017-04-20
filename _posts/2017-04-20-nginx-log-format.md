---
layout: post
title: Nginx 日志格式简介
date: 2017-04-20 12:00:00 +0800
tags:
- nginx
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

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [CONFIGURING LOGGING][ref1]
</span>

[ref1]: https://www.nginx.com/resources/admin-guide/logging-and-monitoring/
