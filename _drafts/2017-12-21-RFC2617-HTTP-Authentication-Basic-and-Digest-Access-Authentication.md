---
layout: post
title: "RFC2617 HTTP Authentication: Basic and Digest Access Authentication"
date: 2017-12-21 22:00:00 +0800
categories: rfc
tags:
- rfc
- http
---

现在的认证方式种类繁多，如 **Bearer Token(jwt)、Basic Auth、Digest Auth、OAuth、Hawk Authentication、AWS Signature、NTLM Authentication** 等，笔者最近阅读 [rfc2617](https://tools.ietf.org/html/rfc2617) 学习了其中的两种 —— Basic Auth & Digest Auth。

HTTP 提供了一个简单的`挑战-响应`认证机制，服务器可以用该机制挑战客户端挺的认证请求。

```
    auth-scheme    = token
    auth-param     = token "=" ( token | quoted-string )
```

源服务器使用 401 (未授权) 响应来挑战客户端的授权。这个响应必须包含一个 WWW-Authenticate 头字段，该字段至少包含一个适用于请求资源的挑战。

```
    challenge   = auth-scheme 1*SP 1#auth-param
```

所有认证方案都定义了认证参数领域，realm 对于发出挑战的所有认证方案都是必需的:

```
    realm       = "realm" "=" realm-value
    realm-value = quoted-string
```

这些 realms 允许将服务器上受保护的资源划分为不同的保护空间，每个保护空间都有自己的认证方案和/或授权数据库。用户代理希望通过源服务器进行身份验证 —— 在收到 401 之后,通常但不是必须的 —— 可以通过在请求中包含一个 `Authorization` 头来实现，该字段由包含所请求资源域的客户端身份验证信息的凭证组成。

```
    credentials = auth-scheme #auth-param
```

<h4>Basic Authentication Scheme</h4>

Basic 认证模式需要客户端必须为每个 realm 提供 user-ID 和 password，Basic 对应的 `挑战-响应` 机制的 ABNF 如下:

```
    challenge   = "Basic" realm
    credentials = "Basic" basic-credentials
```

当在保护空间内接收到对 URI 未经授权的请求，源服务器可以用如下的挑战来响应:

```
    WWW-Authenticate: Basic realm="WallyWorld"
```

其中 `WallyWorld` 是服务器分配的字符串，用于标识 Request-URI 的保护空间。

为了获得权限，客户端发送 userid 和 password 的 base64 编码:

```
    basic-credentials = base64-user-pass
    base64-user-pass  = <base64 encoding of user-pass, except not limited to 76 char/line>
    user-pass   = userid ":" password
    userid      = *<TEXT excluding ":">
    password    = *TEXT
```

所以一个可能的认证请求的 Header 可能如下(userid = Aladdin, password = "open sesame"):

```
    Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

<h4>Digest Access Authentication Scheme</h4>



| Name | ADDRESS | HOSTNAME
|:-|:-|:-
| infra0 | 10.0.63.202/172.0.63.202 | anakin
| infra1 | 10.0.63.203/172.0.63.203 | boba
| infra2 | 10.0.63.204/172.0.63.204 | c3po

![Picture](/assets/201611/picture.png){: width="700px" style="padding-left: 45px"}

{% highlight shell %}
代码，可以为不同语言显示不同的风格，如shell、ruby、python、yaml等
{% endhighlight %}

```shell
同上，用于代码段
```

[链接][url_name]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 吴龙辉. Kubernetes实战. 电子工业出版社. 2016.<br>
2 蒋鑫. [Git 权威指南](/assets/pdf/GotGit.pdf). 机械工业出版社. 2011.<br>
3 [helloworld][url_name]
</span>

[url_name]: url
