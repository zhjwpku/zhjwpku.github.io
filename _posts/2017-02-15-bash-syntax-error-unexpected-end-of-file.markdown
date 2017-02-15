---
layout: post
title: "Bash syntax error: unexpected end of file"
date: 2017-02-15 17:30:00 +0800
tags:
- shell
- dos2unix
---

今天在处理一个shell脚本的时候，遇到了 *syntax error: unexpected end of file*，费了半天劲才发现脚本是在Windows下编辑的，罪魁祸首 <i class="fa fa-bolt" aria-hidden="true"></i> CRLF。

用`dos2unix`命令处理脚本：

{% highlight shell %}
# install dos2unix
→ ~ $ sudo apt-get install dos2unix

# convert file
→ ~ $ dos2unix filename.sh
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1. [Bash syntax error: unexpected end of file][ref]
</span>

[ref]: http://stackoverflow.com/questions/6366530/bash-syntax-error-unexpected-end-of-file
