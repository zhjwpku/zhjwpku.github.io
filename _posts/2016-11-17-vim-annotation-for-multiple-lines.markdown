---
layout: post
title: Vim多行注释
date: 2016-11-17 21:00:00 +0800
tags:
- vim
---

Vim很强大，下面介绍如何使用vim进行多行注释的添加、删除和替换。

<h4>删除</h4>
{% highlight shell %}
1. Ctrl + v
2. 选中要删除注释的行，方向键或hjkl(左下上右)
3. d
{% endhighlight %}

![Picture](/assets/201611/vim_d_anno.gif)

<h4>添加</h4>
{% highlight shell %}
1. Ctrl + v
2. 选中要增加注释的行，方向键或hjkl(左下上右)
3. Shift + i
4. 输入你想添加的字符，通常是‘#’或‘//’
5. 按Esc两次
{% endhighlight %}

![Picture](/assets/201611/vim_a_anno.gif)
<h4>替换</h4>
{% highlight shell %}
1. Ctrl + v
2. 选中要替换注释的行，方向键或hjkl(左下上右)
3. c
4. 输入你想替换的字符，通常是‘#’或‘//’
5. 按Esc两次
{% endhighlight %}

![Picture](/assets/201611/vim_rp_anno.gif)
