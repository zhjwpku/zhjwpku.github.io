---
layout: post
title: "Effective Modern CMake 实践"
date: 2020-04-04 08:00:00 +0800
categories: category
tags:
- cmake
---

笔者去年有篇[文章][cmake_basic]介绍了将团队项目整改为 CMake 用到的一些命令，虽然尽力遵从 Modern CMake 的写法，但还是由于经验不足用到了一些非 Modern 的语法。今年三月初，公司层面开始推广 CMake 构建，项目群请了个专家带领某产品的各子系统进行 CMake 整改，我负责其中一个子系统，跟专家学到了很多规范和技巧。笔者结合最近的实践及网上的 CMake 资料写下本文，记录 CMake 的一些优秀实践。

<h4>CMake 不是构建系统</h4>

严格来讲，CMake 不是构建系统，而是一个跨平台的构建系统生成器。通过编写 CMakeLists.txt，告诉 CMake 生成相应的构建工程，如 Makefile，Ninja 或 MSBuild，从而增加了构建的可移植性。另外，CMake 拥有一个开发工具家族，包括 CMake、CTest、CPack 及 CDash。

<h4>Modern CMake?</h4>

CMake 自版本 2.8.12 引入 Modern CMake 的概念，3.12 之后的版本被称为 More Modern CMake，华为公司建议使用 3.14 及之后的版本。

**语法**

Modern CMake 围绕 **Targets** 和 **Properties** 展开，因此常用的命令如下:

*创建 target*

```
add_library()
add_exectutable()

target_sources()
```

*为 target 指定参数*

```
## 预处理头文件搜索路径 -I
target_include_directories()

## 编译宏 自动在指定的宏前添加 -D
target_compile_definitions()

## 编译选项
target_compile_options()

## 链接库搜索路径 -L
target_link_directories()

## 链接选项
target_link_options()

## 链接库
target_link_libraries()
```

*让 CMake 帮你解析出应该添加哪些编译选项 [cmake-compile-features][cmake-compile-features]*

```
target_compile_features(mylib PRIVATE cxx_constexpr)

target_compile_features(Foo
  PUBLIC
    cxx_strong_enums
  PRIVATE
    cxx_lambdas
    cxx_range_for
)
```

*推荐上面的用法而不是直接指定 c++ 标准*

<h4>实践</h4>

**Interface Library**

Interface Library 是一种不需要构建的库(或 Header-Only 的库)，由于具有传递性，可将其用于公共的依赖，如

```
add_library(project_options INTERFACE)
target_compile_features(project_options INTERFACE cxx_std_17)
```

静态库链接 Interface Library, 用 install 导出静态库会出现错误，可用如下方法解决

```
add_library(foo STATIC foo.cc)
target_link_libraries(foo PRIVATE $<BUILD_INTERFACE:project_options>)
```

*BUILD_INTERFACE 和 INSTALL_INTERFACE 的解释见 [Effective CMake](https://youtu.be/bsXLMQ6WgIk?t=2620) 中相应的讲解。*

**[Object Library][object_lib]**

使用如下方法能同时编静态库和动态库，且源文件只需编一次。

```
add_library(foo_objs OBJECT)
target_sources(foo_objs foo.cc)

add_library(foo_static STATIC)
target_link_libraries(foo_static PRIVATE project_options foo_objs)
set_target_properties(foo_static PROPERTIES ARCHIVE_OUTPUT_NAME foo)    # 更改静态库名字

add_library(foo_shared SHARED)
target_link_libraries(foo_shared PRIVATE project_options foo_objs)
set_target_properties(foo_shared PROPERTIES LIBRARY_OUTPUT_NAME foo)    # 更改动态库名字
```

**[Generator expressions][generator_expression]**

生成表达式能根据不用的信息生成不同的 Properties。

```
add_executable(myapp main.cpp foo.c bar.cpp zot.cu)
target_compile_options(myapp
  PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-fno-exceptions>
)
target_compile_definitions(myapp
  PRIVATE $<$<COMPILE_LANGUAGE:CXX>:COMPILING_CXX>
          $<$<COMPILE_LANGUAGE:CUDA>:COMPILING_CUDA>
)
```

**添加 Target**

类似 Makefile，CMake 中可以添加定制的 target 以满足不同的构建目标。

```
add_custom_target(my_target)
add_dependencies(my_target foo bar)
```

**更改目标的生成路径**

```
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib) # 静态库
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib) # 动态库
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin) # 可执行文件
```

**[find_package][find_package]**

```
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/")

find_package(GTest)

add_executable(foo foo.cc)

target_link_libraries(foo PRIVATE
    GTest::GTest GTest::Main
)
```

**将静态库中的所有对象文件都链接到目标中，并循环查找依赖**

```
## gcc -shared -o libfoo.so foo.o -Wl,--whole-archive libbar.a libfuzz.a -Wl,--no-whole-archive
target_link_libraries(foo PRIVATE
    -Wl,--whole-archive
    libbar.a
    libfuzz.a
    -Wl,--no-whole-archive
)
```

**按需链接静态库中的符号，并循环查找依赖**

```
## gcc -shared -o libfoo.so foo.o -Wl,--start-group libbar.a libfuzz.a -Wl,--end-group
target_link_libraries(foo PRIVATE
    -Wl,--start-group
    libbar.a
    libfuzz.a
    -Wl,--end-group
)

## gcc -shared -o libfoo.so foo.o -Xlinker "-(" libbar.a libfuzz.a -Xlinker "-)"
target_link_libraries(foo PRIVATE
    -Xlinker "-("
    libbar.a
    libfuzz.a
    -Xlinker "-)"
)
```

<h4>Tips</h4>

- CMake is code，像对待代码仓里的其它代码一样对待 CMakeLists.txt，如相同的代码要抽象为函数
- 运行完 cmake 命令生成构建工程后，可以去 build tree 中看一些生成的文件，如 flags.txt 和 link.txt
- 使用 `make VERBOSE=1` 能查看具体的编译参数或链接参数
- 使用正确的变量，如 `CMAKE_CURRENT_SOURCE_DIR`, `CMAKE_PROJECT_NAME`, `PROJECT_NAME`, `CMAKE_SOURCE_DIR`
- 如果只更改了多个子模块中的一个，使用 `make <target> -j` 来编译，避免错误日志刷屏
- 编译所有子系统，想查看编译日志，使用 `make -j | tee /tmp/build_log.txt`
- 在每个源文件对应的目录中都应该有相应的 CMakeLists.txt
- 不使用 file(GLOB)

<h4>TO-Learn</h4>

- [Professional CMake: A Practical Guide][professional-cmake]
- find_package & CPack
- Conan 如何跟 CMake 结合

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [CppCon 2017: Mathieu Ropert “Using Modern CMake Patterns to Enforce a Good Modular Design”][ref1].<br>
2 [C++Now 2017: Daniel Pfeifer “Effective CMake"][ref2]<br>
3 [More Modern CMake - Deniz Bahadir - Meeting C++ 2018][ref3] **[Slides][MoreModernCMake]**<br>
4 [Oh No! More Modern CMake - Deniz Bahadir - Meeting C++ 2019][ref4] **[Slides][OhNoMoreModernCMake]**<br>
5 [Awesome CMake][awesome_cmake].<br>
6 [The Architecture of Open Source Applications: CMake][aosa_cmake]<br>
7 [C++ Weekly - Ep 208 - The Ultimate CMake / C++ Quick Start][YbgH7yat]<br>
8 [How do I make CMake output into a 'bin' dir?][how-do-i-make-cmake-output-into-a-bin-dir]<br>
9 [Effective Modern CMake][effective_modern_cmake]<br>
10 [Embracing Modern CMake - Stephen Kelly][embracing_modern_cmake]<br>
</span>

[cmake_basic]: /2019/11/15/cmake-basic-commands-intro.html
[ref1]: https://www.youtube.com/watch?v=eC9-iRN2b04
[ref2]: https://www.youtube.com/watch?v=bsXLMQ6WgIk
[ref3]: https://www.youtube.com/watch?v=y7ndUhdQuU8
[MoreModernCMake]: https://meetingcpp.com/mcpp/slides/2018/MoreModernCMake.pdf
[ref4]: https://www.youtube.com/watch?v=y9kSr5enrSk
[OhNoMoreModernCMake]: https://github.com/Bagira80/More-Modern-CMake/blob/master/OhNoMoreModernCMake.pdf
[awesome_cmake]: https://github.com/onqtam/awesome-cmake
[aosa_cmake]: http://www.aosabook.org/en/cmake.html
[cmake-compile-features]: https://cmake.org/cmake/help/v3.14/manual/cmake-compile-features.7.html#manual:cmake-compile-features(7)
[YbgH7yat]: https://www.youtube.com/watch?v=YbgH7yat-Jo
[professional-cmake]: https://crascit.com/professional-cmake/
[how-do-i-make-cmake-output-into-a-bin-dir]: https://stackoverflow.com/questions/6594796/how-do-i-make-cmake-output-into-a-bin-dir
[generator_expression]: https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html
[find_package]: https://cmake.org/cmake/help/latest/command/find_package.html
[object_lib]: https://cmake.org/cmake/help/latest/command/add_library.html#object-libraries
[effective_modern_cmake]: https://gist.github.com/mbinna/c61dbb39bca0e4fb7d1f73b0d66a4fd1
[embracing_modern_cmake]: https://www.youtube.com/watch?v=mn1ZnO3MtVk
