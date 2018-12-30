---
layout: post
title: "C 语言头文件11条"
date: 2018-12-25 20:00:00 +0800
tags:
- "C/C++"
---

*一个大型C语言项目中的头文件应该包含什么内容？*想必做C/C++开发的程序员都有过这样的疑问。密歇根大学的[EECS 381][eecs381]课程中的[C Header File Guidelines][cheaderfile] 和 [C++ Header File Guidelines][cppheaderfile] 是两个不错的总结。如下11条规则摘取自 [C Header File Guidelines][cheaderfile]。

**Rule 1**

每个模块的 .h 和 .c 文件都应该有明确的功能点。不要把具有不同功能函数揉在一个模块，也不要把属于同一模块的函数分开。

**Rule 2**

在所有头文件中都使用 *include guards*。且定义的 guard 不要以下划线开头。

**Rule 3**

使用一个模块所需的所有声明必须出现在模块的头文件中，并且该文件始终用于访问模块。

**Rule 4**

头文件应该只包含函数声明（structure type declarations, function prototypes, and global variable extern declarations），函数的定义（function definitions and global variable definitions and initializations）应该放在该模块的 .c 文件中。.c 文件必须 include .h 文件以便编译器检测是否一致。

**Rule 5**

对于整个程序都可见的全局变量，使用 extern 在头文件件中进行声明，其定义声明放在 .c 文件中。

```c
.h
extern int g_number_of_entities;

.c
int g_number_of_entities = 0;  // 既是定义又是声明
```

**Rule 6**

仅在本模块中使用的声明应该避免放在头文件中。如果只是在 .c 文件中使用的结构体、全局变量以及函数，应该将它们的声明或定义放在 .c 文件的顶部，另外可以在全局变量和函数前加上 static，以便赋予它们内部链接的特性。

**Rule 7**

头文件应该只包含能使它通过编译的最少的其他头文件，如果 A.h 中定义的结构体 A 中使用了 X.h 中定义的结构体 X 作为它的成员变量，那么应该在 A.h 中应该 #include "X.h"。不要包含只被 .c 文件需要的头文件，如 "math.h" 就应该只被 .c 文件包含。

**Rule 8**

如果前向声明（incomplete declaration / "forward" declaration）能搞定，就用它代替头文件。例如在结构体 A 中只用 X 结构体的指针作为它的成员变量，那么就不用在 A.h 中包含 X.h。

```c
struct X;   /* incomplete ("forward") declaration */

struct A {
    int i;
    struct X* x_ptr;    
};
```

通常情况下，在函数的具体实现中才会去访问 X 中的成员变量，因此需要在 A.c 中包含 X.h。

**Rule 9**

头文件中包含的内容应该能保证本身能够通过编译。通过编译一个只包含该头文件的 test.c 文件可以检验这点。

**Rule 10**

A.c 文件应该最先 #include "A.h"，然后在包含其它头文件。

**Rule 11**

永远不要 include .c 文件。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [C Header File Guidelines](/assets/pdf/CHeaderFileGuidelines.pdf)<br>
</span>

[eecs381]: http://umich.edu/~eecs381/
[cheaderfile]: http://umich.edu/~eecs381/handouts/CHeaderFileGuidelines.pdf
[cppheaderfile]: http://umich.edu/~eecs381/handouts/CppHeaderFileGuidelines.pdf
