---
layout: post
title: 也答一下Java面试题前200页
date: 2017-11-24 8:00:00 +0800
tags:
- java
---

某公众号推送了一篇文章 👉 **Java 面试资源: 百度“Java面试题”前200页都在这里了**。相信很多Java大神小神都会在阅读的同时给出自己的答案，笔者在此也给出一份个人的解答，检测自己的同时也是一个学习的过程。

题目页面见: [java-baidu-200.md][java-baidu-200]

<h4>基本概念</h4>

**操作系统中heap和stack的区别**

stack(堆栈)用于静态内存分配，heap(堆)用于动态内存分配，都存储在计算机的RAM中。stack以LIFO顺序存储，从stack释放空间仅需调整一个指针;堆用于在运行时分配内存，其大小只受虚拟内存大小的限制，可以随时分配一个块，随时释放它，这使得在任何给定时间跟踪堆的哪些部分被分配或释放变得复杂。注意操作系统的堆与数据结构中的堆是两回事，分配方式类似链表。

[Differences between Stack and Heap][stack-heap]<br>
[What is the difference between the stack and the heap?][stack-heap-diff]

**什么是基于注解的切面实现**



[java-baidu-200]: https://github.com/tangyouhua/program-resource/blob/master/program-interview/java-baidu-200.md
[stack-heap]: http://net-informations.com/faq/net/stack-heap.htm
[stack-heap-diff]: https://www.quora.com/What-is-the-difference-between-the-stack-and-the-heap
