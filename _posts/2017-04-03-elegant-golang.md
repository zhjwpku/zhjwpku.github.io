---
layout: post
title: 优雅的 golang
date: 2017-04-03 18:00:00 +0800
tags:
- golang
---

云计算领域的开源项目中，绝大多数使用 golang 作为开发语言。本文记录 golang 一些实用的语言特性。

<h4>函数 function</h4>

golang 不允许函数重载，函数重载在运行时进行类型匹配，这会造成性能损失。

golang 函数可以赋值给变量，变量获取函数的引用，并且知道函数的 signature, 如果两个变量保存同一个函数的引用，那么两个变量相等。

和许多其他语言一样，golang 也分传值（pass by value）和传引用（pass by reference），类型 slices、maps、interfaces、channels 默认为传引用。

使用命名返回值(named return)可以是代码更简洁:

{% highlight go %}
func getX2AndX3_2(input int) (x2 int, x3 int) {
	x2 = 2 * input
	x3 = 3 * input
	// return x2, x3
	return
}
{% endhighlight %}

<h4>defer</h4>

将函数推迟到 return 或函数出差之后、 `}` 之前执行，类似 Java 或 C++ 的 finally，常用于释放资源。

带有参数的 defer 函数在执行执行的时候，参数值为 defer 当前行的变量值。多个 defer 执行的顺序满足 LIFO。

{% highlight go %}
func a() {
	i := 0
	defer fmt.Println(i)    // 打印0
	i++
	return
}

func f() {
	for i := 0; i < 5; i++ {
		defer fmt.Printf("%d ", i)  // 打印4 3 2 1 0
	}
}
{% endhighlight %}

defer 常见用法：

*关闭文件流*
{% highlight go %}
// open a file
defer file.Close()
{% endhighlight %}

*Unlock*
{% highlight go %}
mu.Lock()
defer mu.Unlock()
{% endhighlight %}

*打印 footer*
{% highlight go %}
printHeader()
defer printFooter()
{% endhighlight %}

*关闭数据库连接*
{% highlight go %}
// open a database connection
defer disconnectFromDB()
{% endhighlight %}

<h4>switch</h4>

{% highlight go %}
switch key: {
case val1: fallthrough 
case val2, val3:
    // do something
default:
    // do something
}

switch {
case condition1:
    ...
case condition2:
    ...
default:
    ...
}

switch initialization {
case val1:
    ...
case val2:
    ...
default:
    ...
}
{% endhighlight %}

上面的 `key` 可以是任意类型，但 val1, val2, val3 必须是同一类型。与 Java/C/C++ 不一样的是，golang switch 的 break 是隐式的，如果想要穿透效果，需要使用 `fallthrough`。

`switch` 后面如果没有变量，第一个为真的 case 将被执行。

`switch` 后面还可以包含初始化语句。

<h4>++ and --</h4>

golang 中的 ++ 和 -- 只有后缀操作符，并且只能在声明的时候用，不能用作表达式。

{% highlight go %}
n = i++         // validate
f(i++)          // 在golang中不允许
a[i] = b[i++]   // 在golang中不允许
{% endhighlight %}

这样的设计真是比 C/C++/Java 清爽了很多啊！

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
