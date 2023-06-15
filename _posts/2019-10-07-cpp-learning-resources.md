---
layout: post
title: C/C++ Learning Resources
date: 2019-10-07 08:00:00 +0800
tags:
- C/C++
---

C/C++ 学习资料，包括但不限于*文章*、*书籍*、*代码库*、*教程*、*视频*。

**[Compiler Explorer](https://godbolt.org/)**

**[Quick C++ Benchmark](https://quick-bench.com/)**

**[wandbox.org](https://wandbox.org/)**

**[cppinsights.io](https://cppinsights.io/)**

**[Latency Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)**

<h4>Articles</h4>

- [C++ Rvalue References Explained][rvalue_references] by Thomas Becker
- [Google C++ Style Guide][cppstyleguide]
- [C++ Core Guidelines][cppcoreguidelines]
- [What Every Programmer Should Know About Memory][cpumemory]
- [The Free Lunch Is Over: A Fundamental Turn Toward Concurrency in Software][freelunchover]
- [Why symbol visibility is good][why-symbol-visibility-is-good]
- [Executable and Linkable Format (ELF)][elf]
- [Modern dynamic linking infrastructure for PLT][3474]
- [The Log: What every software engineer should know about real-time data's unifying abstraction][the-log]
- [SIMD for C++ Developers][simd]
- [x86 Intrinsics Cheat Sheet][simd-cheat-sheet]
- [Data-Parallel Execution using SIMD Instructions](https://db.in.tum.de/teaching/ws1819/dataprocessing/chapter2.pdf)
- [Design Docs at Google][design-docs-at-google]
- [PRINCIPLES OF CHAOS ENGINEERING][principlesofchaos]

<h4>Blogs</h4>

- [Jonathan Boccara's Blog: Fluent {C++}](https://www.fluentcpp.com/)

<h4>Books</h4>

- [Linkers & Loaders][linker_and_loaders]
- [Bottom Up Computer Science][bottomupcs]

- **C**

  - [Modern C][modernc]
  - [C 语言编程透视][cbook]

- **C++**

  - [C++ Primer Plus 6th Edition][cpp_primer_plus_6ed]

  - [C++ Primer 5th Edition][cpp_primer_5ed]

    *Plus 适合 0 基础入门，Primer 则是学 C++ 必看书*

  - [Modern C++ Tutorial: C++11/14/17/20 On the Fly][modern-cpp-tutorial]

    *非常精炼的一本介绍 Modern C++ 的手册*

<h4>Libraries</h4>

- [spdlog][spdlog]: Fast C++ logging library.
- [Abseil Common Libraries (C++)][abseil-cpp].
- [cloudwu/coroutine](https://github.com/cloudwu/coroutine): 基于 getcontext、makecontext、swapcontext 实现的协程库.
- [LevelDB][leveldb]: a fast key-value storage library, under the hood is a LSM tree.

<h4>Standards</h4>

- [C - Project status and milestones](http://www.open-std.org/jtc1/sc22/wg14/www/projects)
- [C++ - Standards](http://www.open-std.org/jtc1/sc22/wg21/docs/standards)

<h4>Tutorial</h4>

- [cppbestpractices](https://github.com/lefticus/cppbestpractices)
- [Function Interposition in Linux](https://jayconrod.com/posts/23/tutorial-function-interposition-in-linux)

<h4>Video</h4>

- **Miscellaneous**

  - [Linux basic anti-debug][UTVp4jpJoyc]

- **YouTube Channel**

  - [**C++** from *The Cherno*][cpp_cherno]

    适合初学者的一个 C++ 教学视频系列，作者不定期更新。建议一天看完（截至 2020/04/19 共 86 集），后面跟起来比较省事。主要介绍一些 C++ 的基础知识，较新的视频有介绍 C++ 17 的新特性，如 async，optional，any 等。

  - [**C++ Weekly** from *Jason Turner*][cpp_weekly].

- **Talks**

  - *Scott Mayers*
    - [CppCon 2014: Scott Meyers "Type Deduction and Why You Care"][wQxj20X-tIU]

  - *Matt Godbolt*
    - [CppCon 2017: “What Has My Compiler Done for Me Lately? Unbolting the Compiler's Lid”][bSkpMdDe4g4]

  - *Jason Turner*
    - [CppCon 2018: “Applied Best Practices”][DHOlsEd0eDE]
    - [CppCon 2019: “The Best Parts of C++"][iz5Qx18H6lg]

  - *Chandler Carruth*
    - [CppCon 2015: "Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!"](https://www.youtube.com/watch?v=nXaxk27zwlk)

  - *John Lakos*
    - [C++Now 2018: “C++ Modules & Large-Scale Development”](https://www.youtube.com/watch?v=EglLjioQ9x0)

  - *Robert O'Callahan*
    - [Record and replay debugging with "rr"](https://www.youtube.com/watch?v=ytNlefY8PIE)

  - *Vince Bridgers*
    - [Using Clang-tidy for Customized Checkers and Large Scale Source Tree Refactoring](https://www.youtube.com/watch?v=UfLH7dORav8)

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Free Programming Books][free-programming-books]<br>
2 [Awesome C++][awesome-cpp]<br>
</span>

[free-programming-books]: https://github.com/EbookFoundation/free-programming-books/blob/master/free-programming-books.md#c-1
[awesome-cpp]: https://github.com/fffaraz/awesome-cpp
[cpp_primer_plus_6ed]: /assets/pdf/books/C++.Primer.Plus.6th.Edition.Oct.2011.pdf
[cpp_primer_5ed]: /assets/pdf/books/C++.Primer.5th.Edition_2013.pdf
[rvalue_references]: http://thbecker.net/articles/rvalue_references/section_01.html
[cpp_cherno]: https://www.youtube.com/watch?v=18c3MTX0PK0&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb
[bSkpMdDe4g4]: https://www.youtube.com/watch?v=bSkpMdDe4g4
[DHOlsEd0eDE]: https://www.youtube.com/watch?v=DHOlsEd0eDE
[spdlog]: https://github.com/gabime/spdlog
[abseil-cpp]: https://github.com/abseil/abseil-cpp
[cppstyleguide]: https://google.github.io/styleguide/cppguide.html
[cppcoreguidelines]: https://github.com/isocpp/CppCoreGuidelines
[iz5Qx18H6lg]: https://www.youtube.com/watch?v=iz5Qx18H6lg
[wQxj20X-tIU]: https://www.youtube.com/watch?v=wQxj20X-tIU
[modern-cpp-tutorial]: https://github.com/changkun/modern-cpp-tutorial
[cpumemory]: https://people.freebsd.org/~lstewart/articles/cpumemory.pdf
[cpp_weekly]: https://www.youtube.com/watch?v=EJtqHLvAIZE&list=PLs3KjaCtOwSZ2tbuV1hx8Xz-rFZTan2J1
[freelunchover]: http://www.gotw.ca/publications/concurrency-ddj.htm
[linker_and_loaders]: https://wh0rd.org/books/linkers-and-loaders/linkers_and_loaders.pdf
[modernc]: https://modernc.gforge.inria.fr/
[why-symbol-visibility-is-good]: https://www.technovelty.org/code/why-symbol-visibility-is-good.html
[elf]: https://www.cs.stevens.edu/~jschauma/631A/elf.html
[cbook]: https://tinylab-1.gitbook.io/cbook/
[3474]: http://lambda-the-ultimate.org/node/3474
[bottomupcs]: https://github.com/ianw/bottomupcs
[UTVp4jpJoyc]: https://www.youtube.com/watch?v=UTVp4jpJoyc
[the-log]: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
[simd]: http://const.me/articles/simd/simd.pdf
[simd-cheat-sheet]: assets/pdf/x86-intrin-cheatsheet-v2.1.pdf
[design-docs-at-google]: https://www.industrialempathy.com/posts/design-docs-at-google/
[principlesofchaos]: http://principlesofchaos.org/
[leveldb]: https://github.com/google/leveldb
