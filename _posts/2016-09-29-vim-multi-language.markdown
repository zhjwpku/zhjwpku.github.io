---
layout: post
title: Vim多语言配置
date: 2016-09-29 22:00:00 +0800
tags:
- vim
---

Vim是一个很好用的编辑工具，可以通过很多插件来打造理想的IDE，但是有个问题可能困惑着一些人，如果在vimrc中设置了C++语言编码规范（如Tab对应的Space数），那么该如何设置python或xml的编码规范呢？

答案很简单，使用Filetype。比如你最常使用的语言是C++，那么你可以在vimrc文件中设置全局的编码规范：
{% highlight shell %}
set ai
set ts = 4
set sw = 4
set smarttab
{% endhighlight %}

现在如果你想在xml语言中使用不同的缩进规则，那么在vimrc中添加如下规则：
{% highlight shell %}
autocmd Filetype xml setlocal expandtab tabstop=2 shiftwidth=2
{% endhighlight %}
