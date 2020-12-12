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
- [Parallel Programming](#parallel-programming)
- [Web Development](#web-development)

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

- [The Missing Semester of Your CS Education][missing-semester] *MIT*
    - [课程视频][missing-semester-video]
    - [Shell Tools and Scripting][ms-shell-tools] find/ripgrep/fd/broot/nnn/ranger
    - [Editors (Vim)][ms-editors] ctrlp.vim/ack.vim/nerdtree
    - [Data Wrangling][ms-data-wrangling] sed/regex/sort/paste/awk/R/gnuplot
    - [Command-line Environment][ms-command-line] tmux/dotfiles/ssh
    - [Version Control (Git)][ms-version-control] git add -p/git diff --cached/Gblame
    - [Debugging and Profiling][ms-debugging-profiling] flame grame/perf/ncdu/lsof/hyperfine/linter
    - [Metaprogramming][ms-metaprogramming] make/symver/ci
    - [Security and Cryptography][ms-security]
    - [Potpourri][ms-potpourri] FUSE/Hammerspoon/UNetbootin/Jupyter
    - [Q&A][ms-qa]

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

#### Parallel Programming

- [15-418][15418] **Parallel Computer Architecture and Programming** *CMU*
    - 课程涵盖并行体系结构（GPU / Multi-Core）、并行编程模型、缓存一致性的实现方式（Snooping-Based & Directory-Based）、同步机制、Lock-Free编程 等内容。
    - [课程主页 - 2015][15418]
    - [课程视频 - 2015][15418-video]
    - 授课者 [Kayvon Fatahalian][kayvonf] 从 CMU 换到 Stanford 后开设了一门对等的课程 [cs149][cs149]，课程 Lab 在 github 上

#### Web Development

- **[Web Programming, Technologies, and Applications][webapp] from Ideas to Systems to Real Impact** *National Tsing Hua University*
    - 台湾国立清华大学开设的一门网络编程课程。
    - Part I  : 介绍 HTTP，HTML，CSS 和 Javascript等网络基础知识
        - [SS-01: Web Development and HTML][ss-01] *[Slides][ss-01-slides]*
        - [SS-02: CSS][ss-02] *[Slides][ss-02-slides]*
            - Assigned Reading: [CSS tutorial][css-tutorial]
        - [SS-03: Landing Page & Bootstrap & CSS3][ss-03] *[Slides][ss-03-slides]*
        - [SS-04: Javascript - the Basics][ss-04] *[Slides][ss-04-slides]*
            - Assigned Reading: [A re-introduction to JavaScript][reintro-js]
    - Part II : 介绍现代Web开发技术，如响应式设计，Bootstrap，ES6/7，React/Redux
        - [SS-05: Modern Javascript][ss-05] *[Slides][ss-05-slides]*
        - [SS-06: React][ss-06] *[Slides][ss-06-slides]*
        - [SS-07: Redux][ss-07] *[Slides][ss-07-slides]*
    - Part III: 介绍 Node.js，PostgreSQL数据库系统等后端技术，及 AWS 和 React Native
        - [SS-08: Backend Development & Node.js & AWS][ss-08] *[Slides][ss-08-slides]*
        - [SS-09: Database Systems & AWS RDS][ss-09]
        - [SS-10: Mobile Development & React Native][ss-10]


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Awesome CS Courses][ref0]<br>
2 [MIT EECS Courses][ref1]<br>
3 [Bilibili 公开课目录][biliopen]<br>
</span>

[ref0]: https://github.com/prakhar1989/awesome-courses
[ref1]: http://catalog.mit.edu/subjects/6/
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
[webapp]: https://nthu-datalab.github.io/webapp/index.html
[ss-01]: https://www.youtube.com/playlist?list=PLlPcwHqLqJDlD86V7FNTP8d7JBvQmITrP
[ss-01-slides]: https://nthu-datalab.github.io/webapp/slides/web/01_Web_HTML.pdf
[ss-02]: https://www.youtube.com/playlist?list=PLlPcwHqLqJDkGpN5725ZP7jR2vTHayk4E
[ss-02-slides]: https://nthu-datalab.github.io/webapp/slides/web/02_CSS.pdf
[css-tutorial]: https://www.w3schools.com/css/
[ss-03]: https://www.youtube.com/watch?v=JGTk_7kaIQQ&list=PLlPcwHqLqJDlwNSyaBRQ3yao5UyhhxQuf
[ss-03-slides]: https://nthu-datalab.github.io/webapp/slides/web/03_Bootstrap.pdf
[ss-04]: https://www.youtube.com/watch?v=OuDZRFugiSQ&list=PLlPcwHqLqJDlKxQfaWR1apRR9EPLz2-yG
[ss-04-slides]: https://nthu-datalab.github.io/webapp/slides/web/04_Javascript.pdf
[reintro-js]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/A_re-introduction_to_JavaScript
[ss-05]: https://www.youtube.com/watch?v=O0tVDwTAI3E&list=PLlPcwHqLqJDkPXpTPlMh8NtCUC-LWYb-f
[ss-05-slides]: https://nthu-datalab.github.io/webapp/slides/web/05_Modern-Javascript.pdf
[ss-06]: https://www.youtube.com/watch?v=9H1oAOeldNI&list=PLlPcwHqLqJDm9ZW6n7f6JGwLXO9ctuW3R
[ss-06-slides]: https://nthu-datalab.github.io/webapp/slides/react/01_React.pdf
[ss-07]: https://www.youtube.com/watch?v=xaTjMXGw_PA&list=PLlPcwHqLqJDndwtjgYp0mYtllK83Eco1U
[ss-07-slides]: https://nthu-datalab.github.io/webapp/slides/react/02_Redux.pdf
[ss-08]: https://www.youtube.com/watch?v=laTipMRN31k&list=PLlPcwHqLqJDkxqZqqAfElde6eESUCz0YO
[ss-08-slides]: https://nthu-datalab.github.io/webapp/slides/backend/01_Node_Backend.pdf
[ss-09]: https://www.youtube.com/watch?v=7EI1wPfNko4&list=PLlPcwHqLqJDkLsH0P-TiAP_83XQaBWnAS
[ss-10]: https://www.youtube.com/watch?v=umH1M5v2Aqc&list=PLlPcwHqLqJDnhXHlgF4tiKYN-K39zYMtR
[missing-semester]: https://missing.csail.mit.edu/
[missing-semester-video]: https://www.youtube.com/watch?v=Z56Jmr9Z34Q&list=PLyzOVJj3bHQuloKGG59rS43e29ro7I57J
[ms-shell-tools]: https://missing.csail.mit.edu/2020/shell-tools/
[ms-editors]: https://missing.csail.mit.edu/2020/editors/
[ms-data-wrangling]: https://missing.csail.mit.edu/2020/data-wrangling/
[ms-command-line]: https://missing.csail.mit.edu/2020/command-line/
[ms-version-control]: https://missing.csail.mit.edu/2020/version-control/
[ms-debugging-profiling]: https://missing.csail.mit.edu/2020/debugging-profiling/
[ms-metaprogramming]: https://missing.csail.mit.edu/2020/metaprogramming/
[ms-security]: https://missing.csail.mit.edu/2020/security/
[ms-potpourri]: https://missing.csail.mit.edu/2020/potpourri/
[ms-qa]: https://missing.csail.mit.edu/2020/qa/
[15418]: http://15418.courses.cs.cmu.edu/spring2015/
[15418-video]: https://scs.hosted.panopto.com/Panopto/Pages/Sessions/List.aspx#folderID=%22a5862643-2416-49ef-b46b-13465d1b6df0%22
[kayvonf]: http://graphics.stanford.edu/~kayvonf/
[cs149]: http://cs149.stanford.edu/fall19/