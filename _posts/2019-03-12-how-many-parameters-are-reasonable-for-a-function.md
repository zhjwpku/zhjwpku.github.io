---
layout: post
title: 函数中最多定义多少个参数是合理的
date: 2019-03-12 20:00:00 +0800
tags:
- "C/C++"
---

依稀记得毕业面试的时候有考官问过: *函数最多定义多少个参数是合理的 ?*  印象中当时是回答了四个，但也只是凭感觉猜的，没有什么根据。

stackoverflow 上搜了几个类似的问题：

[Are there guidelines on how many parameters a function should accept?][145055] <br>
[How many parameters are too many?][174968]

一种观点认为，*一个函数应该定义不超过 3 到 4 个参数*，否则你的函数可能做的太多，并且不好维护，单元测试写起来也相对更加困难。

"Clean Code: A Handbook of Agile Software Craftsmanship" 一书中写到：

```
The ideal number of arguments for a function is zero (niladic). Next comes one (monadic), followed 
closely by two (dyadic). Three arguments (triadic) should be avoided where possible. More than 
three (polyadic) requires very special justification—and then shouldn’t be used anyway.
```

另一种观点认为，代码的组织更应该是一种艺术而非工程，因此不应给它制定固定规则而是根据实际情况来看待这个问题：

```
I hate making hard and fast rules like this because the answer changes not only depending on the 
size and scope of your project, but I think it changes even down to the module level. Depending 
on what your method is doing, or what the class is supposed to represent, it's quite possible 
that 2 arguments is too many and is a symptom of too much coupling.
```

注：*C 语言标准建议函数最少支持的参数个数为127个，而C++标准为256个，但在真正的实现上，可能并无此限制，而是根据栈的大小来决定参数的个数。*

下面笔者从更底层的角度来分析这个问题。

在 x86-64 指令集架构中，当调用一个函数的时候，如果参数不超过6个，会将它们依次存放于 **rdi, rsi, rdx, rcx, r8, r9** 6个寄存器中；超过6个的参数将依次反向压入到栈中。访问栈涉及到访存，速度远比不上访问寄存器。

在 ARM 架构中，参数不超过4个的依次存放到 r0 ~ r3，超过的也会压栈。

如果要求函数在不同的平台上都能有较高的性能，那不超过函数不超过四个应该是比较合理的。当然，由于寄存器的数量有限，编译器很可能将上述寄存器作于它用，因此函数参数个数还是应当越少越好。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [How many maximum arguments can be passed to a function in C?][ref1] <br>
2 [The 64 bit x86 C Calling Convention](/assets/pdf/x86-64bit-c-calling-convention.pdf) from [CS 2150: Program and Data Representation][cs2150]
</span>

[145055]: https://softwareengineering.stackexchange.com/questions/145055/are-there-guidelines-on-how-many-parameters-a-function-should-accept
[174968]: https://stackoverflow.com/questions/174968/how-many-parameters-are-too-many
[ref1]: https://www.quora.com/How-many-maximum-arguments-can-be-passed-to-a-function-in-C
[cs2150]: http://aaronbloomfield.github.io/pdr/readme.html
