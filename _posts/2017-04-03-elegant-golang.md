---
layout: post
title: 优雅的 golang
date: 2017-04-03 18:00:00 +0800
tags:
- golang
---

云计算领域的开源项目中，绝大多数使用 golang 作为开发语言。本文记录 golang 一些实用的语言特性。

<h4>Init function</h4>

有一个特殊的函数叫做init(), 这个函数不能被调用，会在 main 函数执行之前自动调用，另外，当含有 init() 函数的 package 被 import 时，其 init() 函数也会被自动调用。

more to read: [When is the init() function in Go (Golang) run?][ref3]

<h4>Elegant Contants</h4>

{% highlight go %}
type Stereotype int

const (
    TypicalNoob Stereotype = iota // 0
    TypicalHipster                // 1
    TypicalUnixWizard             // 2
    TypicalStartupFounder         // 3
)

type AudioOutput int

const (
    OutMute AudioOutput = iota // 0
    OutMono                    // 1
    OutStereo                  // 2
    _                          // skip value
    _
    OutSurround                // 5
)

type Allergen int

const (
    IgEggs Allergen = 1 << iota // 1 << 0 which is 00000001
    IgChocolate                         // 1 << 1 which is 00000010
    IgNuts                              // 1 << 2 which is 00000100
    IgStrawberries                      // 1 << 3 which is 00001000
    IgShellfish                         // 1 << 4 which is 00010000
)

type ByteSize float64

const (
    _           = iota             // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota) // 1 << (10*1)
    MB                             // 1 << (10*2)
    GB                             // 1 << (10*3)
    TB                             // 1 << (10*4)
    PB                             // 1 << (10*5)
    EB                             // 1 << (10*6)
    ZB                             // 1 << (10*7)
    YB                             // 1 << (10*8)
)

const (
    Apple, Banana = iota + 1, iota + 2  // Apple:      1, Banana: 2
    Cherimoya, Durian                   // Cherimoya:  2, Durian: 3
    Elderberry, Fig                     // Elderberry: 3, Fig:    4
)
{% endhighlight %}

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [iota: Elegant Constants in Golang][ref1]<br>
2 [Iota][ref2]
</span>

[ref1]: https://splice.com/blog/iota-elegant-constants-golang/
[ref2]: https://github.com/golang/go/wiki/Iota
[ref3]: http://stackoverflow.com/questions/24790175/when-is-the-init-function-in-go-golang-run
