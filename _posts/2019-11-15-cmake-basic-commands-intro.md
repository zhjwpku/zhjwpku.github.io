---
layout: post
title: "CMake 基本用法介绍"
date: 2019-11-15 22:00:00 +0800
tags:
- cmake
- build
---

最近着手把部门项目从手写 makefile 改为了 [cmake][cmake] 构建，由于之前使用了不知道哪年哪个不一定在职的员工写了一个貌似通用的构建框架，每个模块内部写一个 makefile.d 并 include 公共 makefile，然后使用一个 python 脚本调用 `make -f` 来执行最终的构建，虽然也 work，但是丑陋且难以维护。改为 cmake 之后一是构建逻辑更加清晰，二是能够指导优化代码结构。本文主要介绍一些常用的 cmake 命令，并列举一些有助于学习 cmake 的资源。

*本文不是 CMake 教程，但后面推荐了一些不错的[教程](/2019/11/15/cmake-basic-commands-intro.html#tutorials)*

<h4>Basic Variables</h4>

如下变量可以简化 CMakeLists.txt 的编写，见字如面。

- [CMAKE_PROJECT_NAME](https://cmake.org/cmake/help/v3.12/variable/CMAKE_PROJECT_NAME.html)
- [PROJECT_NAME](https://cmake.org/cmake/help/v3.12/variable/PROJECT_NAME.html)
- [PROJECT_VERSION](https://cmake.org/cmake/help/v3.12/variable/PROJECT_VERSION.html)
- [CMAKE_SOURCE_DIR](https://cmake.org/cmake/help/v3.12/variable/CMAKE_SOURCE_DIR.html)
- [PROJECT_SOURCE_DIR](https://cmake.org/cmake/help/v3.12/variable/PROJECT_SOURCE_DIR.html)
- [CMAKE_CURRENT_SOURCE_DIR](https://cmake.org/cmake/help/v3.12/variable/CMAKE_CURRENT_SOURCE_DIR.html)
- [CMAKE_BINARY_DIR](https://cmake.org/cmake/help/v3.12/variable/CMAKE_BINARY_DIR.html)
- [PROJECT_BINARY_DIR](https://cmake.org/cmake/help/v3.12/variable/PROJECT_BINARY_DIR.html)
- [CMAKE_CURRENT_BINARY_DIR](https://cmake.org/cmake/help/v3.12/variable/CMAKE_CURRENT_BINARY_DIR.html)
- [CMAKE_CURRENT_LIST_FILE](https://cmake.org/cmake/help/v3.12/variable/CMAKE_CURRENT_LIST_FILE.html)
- [CMAKE_CURRENT_LIST_DIR](https://cmake.org/cmake/help/v3.12/variable/CMAKE_CURRENT_LIST_DIR.html)
- [CMAKE_ARCHIVE_OUTPUT_DIRECTORY](https://cmake.org/cmake/help/v3.12/variable/CMAKE_ARCHIVE_OUTPUT_DIRECTORY.html) Linux 上 .a 的输出路径
- [CMAKE_LIBRARY_OUTPUT_DIRECTORY](https://cmake.org/cmake/help/v3.12/variable/CMAKE_LIBRARY_OUTPUT_DIRECTORY.html) Linux 上 .so 输出路径
- [CMAKE_RUNTIME_OUTPUT_DIRECTORY](https://cmake.org/cmake/help/v3.12/variable/CMAKE_RUNTIME_OUTPUT_DIRECTORY.html)

更多变量的使用方法请查阅 [cmake-variables](https://cmake.org/cmake/help/v3.12/manual/cmake-variables.7.html)。

<h4>Basic Commands</h4>

cmake 的命令很多，本文只简单介绍笔者在项目改造时用到的命令。

**1. [cmake_minimum_required](https://cmake.org/cmake/help/v3.12/command/cmake_minimum_required.html)**

设置项目需要的最小 cmake 版本，因为较新的版本会有老版本没有的命令，如果 cmake 版本号小于该命令指定的版本，cmake 会报错。另外在后面的视频里会提到版本 2.8.12 之后的 cmake 为 **modern cmake**，更加注重 **modular design**，因此项目使用的版本不应低于 2.8.12。

```
cmake_minimum_required(VERSION 3.12)
```

**2. [project](https://cmake.org/cmake/help/v3.12/command/project.html)**

设置项目名称、版本及项目使用的语言等。

```
project(Helloworld VERSION 1.0.0 LANGUAGES C CXX)
```

**3. [set](https://cmake.org/cmake/help/v3.12/command/set.html)**

设置正常变量、缓存变量或环境变量的值

```
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")
# 设置打包.a文件使用的参数
set(CMAKE_C_ARCHIVE_CREATE "<CMAKE_AR> crus <TARGET> <LINK_FLAGS> <OBJECTS>")
# file.c 编译生成的对象文件名为 file.o 而不是 file.c.o
set(CMAKE_C_OUTPUT_EXTENSION_REPLACE 1)
```

**4. [option](https://cmake.org/cmake/help/v3.12/command/option.html)**

可供用户选择的选项，默认为OFF，可以通过 `ccmake` 选择或使用 `cmake -D` 参数设定

```
option(BUILD_SHARED_LIBS "BUILD the shared library" OFF)
option(DEBUG "DEBUG BUILD" ON)
```

**5. [add_compile_options](https://cmake.org/cmake/help/v3.12/command/add_compile_options.html)**

虽然我知道应该尽量少用这个命令，但是在改造项目的时候还是用了，原因是因为项目下的七八个模块使用的几乎都是相同的编译选项，而且非常多，后面如果找到更好的方式会改掉。

```
if(DEBUG)
  add_compile_options(-O0 -g -ggdb -fno-omit-frame-pointer -fprofile-arcs -ftest-coverage)
endif()
```

*-fprofile-arcs 和 -ftest-coverage 开启 gcov，编译之后会在生成 .o 的目录下生成一个同名的 .gcno 文件，可执行文件中记录了这些文件所在的位置，在执行完之后会将代码执行的结果写到每个 .gcno 文件对应的目录下，后缀为 .gcda，通过 lcov 可以将 .gcno 和 .gcda 的结果进行统计，便于覆盖率的统计*

当定义了 target 之后应尽可能用 [target_compile_options](https://cmake.org/cmake/help/v3.13/command/target_compile_options.html)。

**6. [add_compile_definitions](https://cmake.org/cmake/help/v3.12/command/add_compile_definitions.html)**

```
# 等同于 add_compile_options(-DMACRO_FEATURE_A)
add_compile_definitions(MACRO_FEATURE_A)
```

当定义了 target 之后应尽可能用 [target_compile_definitions](https://cmake.org/cmake/help/v3.13/command/target_compile_definitions.html)。

**7. [include_directories](https://cmake.org/cmake/help/v3.12/command/include_directories.html)**

这个命令也不建议使用了，但是在我们的项目中顶级目录并没有定义任何 target，因此也没有想到更好的办法。

```
# 编译时从添加的路径寻找头文件
include_directories(${PROJECT_SOURCE_DIR}/include)
```

**8. [add_subdirectory](https://cmake.org/cmake/help/v3.12/command/add_subdirectory.html)**

添加一个子目录到构建，该目录下必须有 CMakeLists.txt 文件。

**9. [file](https://cmake.org/cmake/help/v3.12/command/file.html)**

file 的用法很多种，这里仅说明如何使用它来查找C文件。

```
file(GLOB_RECURSE SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "*.c")
```

**10. [add_library](https://cmake.org/cmake/help/v3.12/command/add_library.html)**

指定要生成的静态库（STATIC）或动态库（SHARED），如果不指定，则根据 [BUILD_SHARED_LIBS](https://cmake.org/cmake/help/v3.12/variable/BUILD_SHARED_LIBS.html) 选项是否打开来生成默认的版本，ON 生成动态库，OFF 生成静态库。

```
# 生成静态库 libmylib.a
add_library(mylib STATIC ${SOURCES})
```

**11. [target_include_directories](https://cmake.org/cmake/help/v3.12/command/target_include_directories.html)**

指定编译 target 所需要的头文件。PUBLIC 和 INTERFACE 具有依赖传递性，而 PRIVATE 没有。

```
target_include_directories(mylib
                        PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src
                        PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
                        )
```

|  | &nbsp; &nbsp; &nbsp; PUBLIC &nbsp; &nbsp; &nbsp; | &nbsp; &nbsp; &nbsp; PRIVATE &nbsp; &nbsp; &nbsp;| &nbsp; &nbsp; &nbsp; INTERFACE &nbsp; &nbsp; &nbsp;
|:-|:----------:|:-----------:|:----------:
| Needed By Me | YES | YES | NO
| Needed By Dependers | YES | NO | YES

<br>

**12. [target_link_libraries](https://cmake.org/cmake/help/v3.12/command/target_link_libraries.html)**

如果 target 是一个 library，该命令可以用来指定依赖本仓库的 target 还需要链接另外的仓库，用于解决循环依赖。

```
target_link_libraries(mylib INTERFACE anotherlib)
```

如果 target 是可执行文件，则该命令用于指定其需要链接的库。

```
list(APPEND EXTRA_LIBS gcov mylib)
target_link_libraries(mymain ${EXTRA_LIBS})
```

**13. [add_executable](https://cmake.org/cmake/help/v3.12/command/add_executable.html)**

指定可执行文件。

```
add_executable(mymain ${SOURCES})
```

**14. [target_link_directories](https://cmake.org/cmake/help/v3.13/command/target_link_directories.html)**

指定链接路径，该命令在 3.12 以下版本没有，笔者在项目中使用如下命令设置库的路径，其实更好的方法是用 [find_library](https://cmake.org/cmake/help/v3.13/command/find_library.html)。

```
# bad way
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${CMAKE_CURRENT_SOURCE_DIR}/../dependentLibs"
# good way
find_library(mylib ../dependentLibs)
```

**15. [target_compile_features](https://cmake.org/cmake/help/v3.13/command/target_compile_features.html)**

定义 target 所需的编译特性，让 cmake 解决编译选项。

**16. [add_custom_target](https://cmake.org/cmake/help/v3.12/command/add_custom_target.html)**

*cmake/modules/cleangcov.cmake*
```
add_custom_target(cleangcov @echo cleaning for gcov)

add_custom_command(
    COMMENT "clean .gcda files"
    COMMAND find
    ARGS    . -name "*.gcda" -type f -delete
    TARGET cleangcov
    )
```

*CMakeLists.txt*

```
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")
include(cleangcov)
```

**17. [cmake_parse_arguments](https://cmake.org/cmake/help/v3.12/command/cmake_parse_arguments.html)**

这个命令稍微复杂一点，后面补充。

**Legacy commands**

modern cmake 尽量少用如下命令，见 [Legacy CMake Commands][legacy]。

- include_directories
- add_definitions
- add_dependencies
- add_compile_options
- link_libraries
- link_directories

**Tips**

- Modern CMake is all about `Targets` and `Properties`
- 使用 `cmake -P <script>.cmake` 调试 cmake 脚本
- else()/endif() 括号中无需写任何语句
- CMakeLists.txt 一定不能少写 s
- 使用 ccmake 来交互配置 CMakeCache.txt
- 使用 make VERBOSE=1 来查看具体编译时的编译选项及链接选项
- 编译选项和链接选项可以在 CMakeFiles 目录下的 flags.txt 和 link.txt 文件中查看
- Get your hands off CMAKE_CXX_FLAGS/CMAKE_C_FLAGS

更多命令的使用方法请查阅 [cmake-commands](https://cmake.org/cmake/help/v3.12/manual/cmake-commands.7.html)。

<h4>Tutorials</h4>

- **[CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)** && [源码](https://github.com/Kitware/CMake/tree/master/Help/guide/tutorial)
- **[C++ Weekly - Ep 78 - Intro to CMake](https://www.youtube.com/watch?v=HPMvU64RUTY)**
- **[How to CMake Good](https://www.youtube.com/playlist?list=PLK6MXr8gasrGmIiSuVQXpfFuE1uPT615s)**
- **[cmake-examples](https://github.com/ttroy50/cmake-examples)** 这个我还没看，不知道好不好

<h4>Books</h4>

- **[Mastering CMake](https://github.com/Akagi201/learning-cmake/blob/master/docs/mastering-cmake.pdf)** 这本书里中的 cmake 版本较低，没看。
- **[Professional CMake: A Practical Guide][professional-cmake]** 推荐度很高的一本书。

<h4>Talks</h4>

这部分列出了 Youtube 上一些对学习 cmake 有所帮助的演讲，基本都来自 [CppCon](https://www.youtube.com/user/CppCon/featured) 和 [BoostCon](https://www.youtube.com/user/BoostCon/featured)。

**[Daniel Pfeifer "Effective CMake"][video-1]**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/bsXLMQ6WgIk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<br>

**[Using Modern CMake Patterns to Enforce a Good Modular Design][video-2]**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/eC9-iRN2b04" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<br>

**[Git, CMake, Conan - How to ship and reuse our C++ projects][video-3]**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/S4QSKLXdTtA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [CMake Gcov c++ creating wrong .gcno files](https://stackoverflow.com/questions/37978016/cmake-gcov-c-creating-wrong-gcno-files).<br>
2 [cmake-buildsystem](https://cmake.org/cmake/help/v3.12/manual/cmake-buildsystem.7.html).<br>
</span>

[cmake]: https://cmake.org/
[meson]: https://mesonbuild.com/
[ninja]: https://github.com/ninja-build/ninja
[video-1]: https://www.youtube.com/watch?v=bsXLMQ6WgIk
[video-2]: https://www.youtube.com/watch?v=eC9-iRN2b04
[video-3]: https://www.youtube.com/watch?v=S4QSKLXdTtA
[legacy]: https://youtu.be/jt3meXdP-QI?t=2254
[professional-cmake]: https://crascit.com/professional-cmake/