---
layout: post
title: SMTP AUTH extension not supported by server
date: 2017-02-14 22:00:00 +0800
tags:
- python
- smtp
---

使用 Python 发送邮件的时候出现了 *SMTP AUTH extension not supported by server* 的错误，同样的代码换163邮箱地址发送就不会出错，于是想当然的认为是邮箱的问题。其实不然！

Python email Code:

{% highlight python %}
#!/usr/bin/python

import smtplib
from email.mime.text import MIMEText
from email.header import Header

# The gmail test, fails
from_addr = 'yourname@gmail.com'
from_pass = '!QAC@7SYcdxc'
smtp_host = 'smtp.gmail.com'
smtp_port = 587

# Another successful test
#from_addr = "yourname@163.com"
#from_pass = "hkrhtxxdzfjsjkqj"
#smtp_host = "smtp.163.com"
#smtp_port = 25

# Send to this email
to_addr = "yourname@receive.com"

msg = MIMEText('hello, send by superman', 'plain', 'utf-8')
msg['Subject'] = Header('Test email account %s' % from_addr, 'utf-8')
msg['from'] = from_addr
msg['to'] = to_addr

server = smtplib.SMTP(smtp_host, smtp_port)
server.set_debuglevel(1)
# Blow the tow line solve the problom
#server.ehlo()
#server.starttls()
server.login(from_addr, from_pass)
server.sendmail(from_addr, to_addr, msg.as_string())
server.quit()
{% endhighlight %}

程序运行报错：

{% highlight shell %}
→ ~ $ ./testemail.py 
send: 'ehlo [127.0.1.1]\r\n'
reply: '250-smtp.gmail.com at your service, [160.16.209.99]\r\n'
reply: '250-SIZE 35882577\r\n'
reply: '250-8BITMIME\r\n'
reply: '250-STARTTLS\r\n'   # Attention!
reply: '250-ENHANCEDSTATUSCODES\r\n'
reply: '250-PIPELINING\r\n'
reply: '250-CHUNKING\r\n'
reply: '250 SMTPUTF8\r\n'
reply: retcode (250); Msg: smtp.gmail.com at your service, [160.16.209.99]
SIZE 35882577
8BITMIME
STARTTLS # Attention!
ENHANCEDSTATUSCODES
PIPELINING
CHUNKING
SMTPUTF8
Traceback (most recent call last):
  File "./testemail.py", line 28, in <module>
    server.login(from_addr, from_pass)
  File "/usr/lib/python2.7/smtplib.py", line 585, in login
    raise SMTPException("SMTP AUTH extension not supported by server.")
smtplib.SMTPException: SMTP AUTH extension not supported by server.
{% endhighlight %}

解决的办法也很简单，即添加 server.ehlo() 和 server.starttls()。但这背后的原因是怎么样的，我们不妨来做个对比。添加 server.ehlo() 和 server.starttls() 再次运行：

{% highlight shell %}
→ ~ $ ./testemail.py 
send: 'ehlo [127.0.1.1]\r\n'
reply: '250-smtp.gmail.com at your service, [160.16.209.99]\r\n'
reply: '250-SIZE 35882577\r\n'
reply: '250-8BITMIME\r\n'
reply: '250-STARTTLS\r\n'   # Attention!
reply: '250-ENHANCEDSTATUSCODES\r\n'
reply: '250-PIPELINING\r\n'
reply: '250-CHUNKING\r\n'
reply: '250 SMTPUTF8\r\n'
reply: retcode (250); Msg: smtp.gmail.com at your service, [160.16.209.99]
SIZE 35882577
8BITMIME
STARTTLS # Attention!
ENHANCEDSTATUSCODES
PIPELINING
CHUNKING
SMTPUTF8
send: 'STARTTLS\r\n'  # Attention!
reply: '220 2.0.0 Ready to start TLS\r\n' # Attention!
reply: retcode (220); Msg: 2.0.0 Ready to start TLS # Attention!
send: 'ehlo [127.0.1.1]\r\n'
reply: '250-smtp.gmail.com at your service, [160.16.209.99]\r\n'
reply: '250-SIZE 35882577\r\n'
reply: '250-8BITMIME\r\n'
reply: '250-AUTH LOGIN PLAIN XOAUTH2 PLAIN-CLIENTTOKEN OAUTHBEARER XOAUTH\r\n'
reply: '250-ENHANCEDSTATUSCODES\r\n'
reply: '250-PIPELINING\r\n'
reply: '250-CHUNKING\r\n'
reply: '250 SMTPUTF8\r\n'
reply: retcode (250); Msg: smtp.gmail.com at your service, [160.16.209.99]
SIZE 35882577
8BITMIME
AUTH LOGIN PLAIN XOAUTH2 PLAIN-CLIENTTOKEN OAUTHBEARER XOAUTH
ENHANCEDSTATUSCODES
PIPELINING
CHUNKING
SMTPUTF8
send: 'AUTH PLAIN AGN93C9cbCVyc2VydmljZUBtYWlscy5adXeFdGltZXN0di5zb12EIEBSWkBXU1gzZWRj\r\n'
reply: '235 2.7.0 Accepted\r\n'
reply: retcode (235); Msg: 2.7.0 Accepted
send: 'mail FROM:<yourmail@gmail.com> size=280\r\n'
reply: '250 2.1.0 OK n87sm3405999pfi.122 - gsmtp\r\n'
reply: retcode (250); Msg: 2.1.0 OK n87sm3405999pfi.122 - gsmtp
send: 'rcpt TO:<yourname@receive.com>\r\n'
reply: '250 2.1.5 OK n87sm3405999pfi.122 - gsmtp\r\n'
reply: retcode (250); Msg: 2.1.5 OK n87sm3405999pfi.122 - gsmtp
send: 'data\r\n'
reply: '354  Go ahead n87sm3405999pfi.122 - gsmtp\r\n'
reply: retcode (354); Msg: Go ahead n87sm3405999pfi.122 - gsmtp
data: (354, 'Go ahead n87sm3405999pfi.122 - gsmtp')
send: 'Content-Type: text/plain; charset="utf-8"\r\nMIME-Version: 1.0\r\nContent-Transfer-Encoding: base64\r\nSubject: =?utf-8?q?Test_email_account_yourmail=40gmail=2Ecom?=\r\nfrom: yourmail@gmail.com\r\nto: yourname@receive.com\r\n\r\naGVsbG8sIHNlbmQgYnkgemhhb2p3\r\n.\r\n'
reply: '250 2.0.0 OK 1487123527 n87sm3405999pfi.122 - gsmtp\r\n'
reply: retcode (250); Msg: 2.0.0 OK 1487123527 n87sm3405999pfi.122 - gsmtp
data: (250, '2.0.0 OK 1487123527 n87sm3405999pfi.122 - gsmtp')
send: 'quit\r\n'
reply: '221 2.0.0 closing connection n87sm3405999pfi.122 - gsmtp\r\n'
reply: retcode (221); Msg: 2.0.0 closing connection n87sm3405999pfi.122 - gsmtp
{% endhighlight %}

我想大家看了两次输出的不同应该已经明白其中的缘由，当给 SMTP 服务器发送 `ehlo` 报文的时候，reply 里包含 *250-STARTTLS*，客户端需要发送 `STARTTLS` 报文（协议）进行应答，以建立 TLS 通道，之后发送的内容都会加密传输。

**我们在代码中添加 server.starttls() 就是为了模拟发送 `STARTTLS` 报文的功能以建立安全连接。**

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [SMTP AUTH extension not supported by server][ref1]<br>
2 [25, 465, 587... What port should I use?][ref2]
</span>

[ref1]: https://www.pythonanywhere.com/forums/topic/1961/
[ref2]: http://blog.mailgun.com/25-465-587-what-port-should-i-use/
