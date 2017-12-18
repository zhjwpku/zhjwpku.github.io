---
layout: post
title: "C/C++ 及 Java 中 static 的含义总结"
date: 2017-12-15 12:00:00 +0800
tags:
- static
---

`static` 关键字在 C/C++ 及 Java 语言中都有其特定的含义，本文做一简单总结。

**static in C**

*1. static 全局变量*

static 全局变量和其它全局变量都是存储在.data段或者.bss段，但它只在定义它的源文件内有效，其它源文件无法访问。

*2. static 局部变量*

普通的局部变量在栈空间上分配，这个局部变量所在的函数被多次调用时，每次调用这个局部变量在栈上的位置都不一定相同。局部变量也可以在堆上动态分配，但是记得使用完这个堆空间后要释放。
static局部变量与普通的局部变量比起来有如下几个区别:

    a. 静态局部变量被编译器放在全局存储区.data，虽然是局部的，但是在程序的整个生命周期中存在
    b. 静态局部变量只能被其作用域内的变量或函数访问
    c. 静态局部变量如果没有被用户初始化，则会被编译器自动赋值为0，以后每次调用静态局部变量的时候都用上次调用后的值

*3. static 函数*

定义为static的函数只能在当前文件内部调用。

**static in C++**

// TODO

**static in Java**

_static方法就是没有this的方法。在static方法内部不能调用非静态方法，反过来是可以的。而且可以在没有创建任何对象的前提下，仅仅通过类本身来调用static方法。这实际上正是static方法的主要用途_ —— 引自《Java编程思想》

最常见的例子就是main函数，main函数被定义为static因为它必须在任何对象存在之前被调用。

*static 方法*

静态方法有如下限制:

    a. 只能直接调用其它静态方法
    b. 只能访问静态收
    c. 不能被this或super访问

**JDK8 为接口增加了一个新的特性，可以在接口中定义静态方法，在接口中定义静态方法不需要类来实现就可以直接调用。**

*static 变量*

静态变量被所有的对象所共享，在内存中只有一个副本，当且仅当在类初次加载时被初始化。

*static block*

static块可以置于类的任何地方，类中可以有多个static块。在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只执行一次。

Java 中的 static 还可以用来修饰 nested class，但由于 static nested class 不能直接访问包含它的类的非静态成员，这种用法不是很常见。常见的用法是 non-static nested class，又被称为 inner class。

Java 中的 static 不改变成员的访问权限，且不允许用来修饰局部变量。

**2017-12-18 更新**

*static import*

通过在 import 之后使用关键字 static，可以导入类或接口的静态成员，这称为静态导入。使用静态导入可以直接通过名称来引用静态成员，而不必使用类名来进行限定，这简化并缩短了使用静态成员所需的语法。但请不要过度使用。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [C语言中的 static 详细分析](http://blog.csdn.net/keyeagle/article/details/6708077/)<br>
2 [Java static keyword](https://www.javatpoint.com/static-keyword-in-java)<br>
3 Herbert Schildt. Java The Complete Reference Ninth Edition. McGraw-Hill. 2014
</span>
