---
layout: post
title: Awesome CS Courses
date: 2018-09-16 10:00:00 +0800
tags:
- course
---

笔者业余时间学习的计算机课程，全部来自世界名校，列于此，共勉。

<h4>TOC</h4>
- [Computer System](#cs)
- [Language](#language)
- [Operating System](#os)

#### CS

- [CS 61C][cs61c] **Great Ideas in Computer Architecture (Machine Structures)** *UC Berkele*
    - 课程覆盖了 C 语言、汇编语言（MIPS 指令集）、CPU 设计（包括逻辑电路/Cache/流水线）、内存管理、并发编程、数据中心([Warehouse-Scale Computer][wsc])等话题，内容偏基础，老师讲得非常棒。
    - 教材: [Computer Organization And Design 5th Edition][cs61c-refbook]
    - [课程主页 - 2015][cs61c]
    - [课程视频 - 2015][cs61c-video]
    - [Labs - 2018][cs61c-labs]
    - [历届考试题][cs61c-exam]

- [15-213][15213] **Introduction to Computer Systems** *CMU*
    - ICS 能让你成为更高效的程序员，特别是在处理性能，可移植性和健壮性问题时。BTW，15213 是 CMU 的邮编。
    - 教材: [Computer Systems: A Programmer's Perspective, 3/E (CS:APP3e)][15213-textbook]
    - [课程主页][15213]
    - [课程视频][15213-video]
    - [Labs][15213-labs]

#### Language

- [CS 106A][cs106a] **Programming Methodology, Spring 2017** *Stanford*
    - 编程方法论是编程入门课程中最大的课程，也是斯坦福大学最大的课程之一。该课程使用 **Java** 语言着重介绍了现代软件工程原理：面向对象的设计，分解，封装，继承，多态等。课程相对简单，为了衔接之后的 [CS 106B][cs106b]，笔者把所有的课程视频浏览了一遍。
    - [课程主页][cs106a]
    - [课程视频][cs106a-video]
- [CS 106B][cs106b] **Programming Abstractions, Winter 2018** *Stanford*
    - 该课程是[编程方法论][cs106a]的自然继承，使用 **C++** 语言教学，涵盖了递归，算法分析和数据抽象等高级编程主题。
    - 教材: [Programming Abstractions in C++][cs106b-textbook]
    - [课程主页][cs106b]
    - [课程视频][cs106b-video]
- [CS 106X][cs106x] **Programming Abstractions (Accelerated)** *Stanford*
    - 课程 106X 覆盖的内容与 106B 相同，使用 **C++**，但教授的速度更快，层次更深。
    - [课程主页][cs106x]
    - [课程视频][cs106x-video]
- [CS 106L][cs106l] **Standard C++ Programming, Autumn 2019** *Stanford*
    - 教材: [CS106L Course Reader][cs106l-textbook] by Keith Schwarz
    - [课程主页](http://web.stanford.edu/class/cs106l/lectures.html)
    - [课程视频][cs106l-video]

#### OS

- [OSTEP][ostep] **Operating Systems: Three Easy Pieces(virtualization, concurrency, and persistence)** *University of Wisconsin*
    - [Homework][ostep-homework]
    - [Projects][ostep-projects]


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Awesome CS Courses][ref0]<br>
2 [Bilibili 公开课目录][biliopen]<br>
</span>

[ref0]: https://github.com/prakhar1989/awesome-courses
[biliopen]: https://github.com/wenhan-wu/OpenCourseCatalog#%E8%AE%A1%E7%AE%97%E6%9C%BA--computer-science
[ostep]: http://pages.cs.wisc.edu/~remzi/OSTEP/
[ostep-projects]: https://github.com/remzi-arpacidusseau/ostep-projects
[ostep-homework]: http://pages.cs.wisc.edu/~remzi/OSTEP/Homework/homework.html
[cs61c]: http://www-inst.eecs.berkeley.edu/~cs61c/sp15/
[cs61c-video]: https://archive.org/details/ucberkeley-webcast-PL-XXv-cvA_iCl2-D-FS5mk0jFF6cYSJs_?sort=titleSorter
[cs61c-labs]: https://github.com/61c-teach/fa18-lab-starter
[cs61c-exam]: https://hkn.eecs.berkeley.edu/exams/course/CS/61C
[cs61c-refbook]: /assets/pdf/ComputerOrganizationAndDesign5thEdition2014.pdf
[wsc]: /assets/pdf/TheDatacenterAsaComputer.pdf
[15213]: http://www.cs.cmu.edu/~213/index.html
[15213-textbook]: http://csapp.cs.cmu.edu/
[15213-video]: https://scs.hosted.panopto.com/Panopto/Pages/Sessions/List.aspx#folderID=%22b96d90ae-9871-4fae-91e2-b1627b43e25e%22&view=2
[15213-labs]: http://csapp.cs.cmu.edu/3e/labs.html
[cs106a]: http://web.stanford.edu/class/archive/cs/cs106a/cs106a.1176/index.shtml
[cs106a-video]: https://www.bilibili.com/video/av21133071
[cs106b]: http://stanford.edu/class/archive/cs/cs106b/cs106b.1184/index.shtml
[cs106b-video]: https://www.bilibili.com/video/av21620553
[cs106b-textbook]: /assets/pdf/books/Programming.Abstractions.in.CPP.pdf
[cs193a-video]: https://www.youtube.com/watch?v=iBBOUzGS8QU&list=PLBx6OgewIjRoF1eh017uRiPuV42piY_iP&index=2
[cs106x]: http://web.stanford.edu/class/archive/cs/cs106x/cs106x.1182/index.shtml
[cs106x-video]: https://www.bilibili.com/video/av21619854
[cs106l]: http://web.stanford.edu/class/cs106l/index.html
[cs106l-video]: https://www.youtube.com/playlist?list=PLCgD3ws8aVdolCexlz8f3U-RROA0s5jWA
[cs106l-video-bilibili]: https://www.bilibili.com/video/av76247001
[cs106l-textbook]: /assets/pdf/books/cs106l_course_reader.pdf