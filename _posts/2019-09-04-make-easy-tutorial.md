---
layout: post
title: Make Easy Tutorial
date: 2019-09-04 08:00:00 +0800
categories: build
tags:
- make
---

虽然现在很多时候都不用手写 Makefile 了，例如 [Autotools][autotools]、[cmake][cmake]、[scons][scons]、[qmake][qmake] 等，都是很好的构建系统。但能够读懂 Makefile 在很多时候是非常必要的，因此本文列举一些基本的 Makefile 要点，方便笔者在需要的时候参考。

Makefile 是由一系列规则（Rule）组成的，一条规则包含三个部分：`工作目标`（target）、`必要条件`（prerequisites，或称依存对象 dependents）和要执行的`命令`（command）。

```
target: prereq 1 prereq2
    commands            # 命令一定要以 tab 字符开始，在 vim 下可用 set list 查看
```

`工作目标`是一个必须构建的文件或进行的事情；`必要条件`是`工作目标`被创建或执行之前，必须事先存在的文件；而`命令`则是将`必要条件`创建为`工作目标`需要执行的 shell 命令。

当 make 要处理某项`规则`时，它会找出`必要条件`和`工作目标`中指定的文件，如果`必要条件`中的文件有关联的`规则`，make 会尝试处理这些规则，之后才是`工作目标`。如果`必要条件`中文件有更改的文件，则 make 会执行`命令`以便重新构建`工作目标`。`命令`会被传递给 shell 并在 subshell 中执行。

由于有些`规则`的`工作目标`可以是另一个`规则`的`必要条件`，因此这些`工作目标`和`必要条件`形成了一个依赖关系图（dependency graph），建立并处理这个关系图以更新指定的`工作目标`就是 make 所要做的事情。

**假想（Phony）工作目标**

.PHONY 用来告诉 make，它的必要条件对应的工作目标不是一个真正的文件。常见的假想工作目标有：

```
工作目标            功能
all             执行编译应用程序的所有工作
install         从已编译的二进制文件进行应用程序的安装
clean           将产生自源代码的二进制文件删除
distclean       删除编译过程中所产生的任何文件
TAGS            建立可供编辑器使用能够的编辑表（ctags和etags）
info            从 Textinfo 源代码来创建 GNU info 文件
check           执行与应用程序相关的测试
```

**自动变量（Automatic Variables）**

make 中使用 `$(variable-name)`(现在更常用一点) 或 `${variable-name}` 标识变量，单字符的变量不需要加括号，通常 makefile 文件中会定义许多变量，不过其中有许多特殊变量是 make 自动定义的。当`规则`相符时， make 会设定自动变量，通过它们，你可以取用`工作目标`以及`必要条件`中的元素。

```
$@              工作目标的文件名
$%              档案文件成员结构中的文件名元素
$<              第一个必要条件的文件名
$?              时间戳在工作目标时间戳之后的所有必要条件，并以空格隔开这些必要条件。即上次执行之后变更过的必要条件
$^              以空格隔开的所有必要条件列表（重复的文件名会被删除）
$+              同 $^，但包含所有的重复文件名
$*              工作目标的主文件名。一个文件名称由两部分组成：主文件名（stem）和扩展名（suffix）
```

**变量类型**

make 的变量有两种类型：简单扩展（simply expanded）变量和递归扩展（recursively expanded）变量。前者使用 `:=` 来定义，而后者使用 `=` 定义。递归变量的扩展动作会被延迟到该变量被使用的时候才进行。

**函数**

具体见到了直接查手册。

**命令修饰符**

一个`命令`可以通过若干前缀加以修饰。

- `@` 修饰符告诉 make 不要输出`命令`本身
- `-` 用来指示 make 忽略被修饰行的结束状态，当命令出错的时候，不会终止运行
- `+` 用来要求 make 执行这条命令，即使用户是以 --just-print(-n) 选项来执行 make



<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [GNU Make Manual](https://www.gnu.org/software/make/manual/)<br>
2 [Managing Projects with GNU Make, Third Edition](/assets/pdf/books/Managing.Projects.With.Gnu.Make.3Rd.Edition.pdf)<br>
</span>

[cmake]: https://cmake.org/
[scons]: https://github.com/SCons/scons
[qmake]: https://doc.qt.io/qt-5/qmake-manual.html
[autotools]: https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html