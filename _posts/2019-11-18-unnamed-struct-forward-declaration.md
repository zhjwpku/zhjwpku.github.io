---
layout: post
title: "匿名 struct 的前向声明"
date: 2019-11-18 22:00:00 +0800
tags:
- "C/C++"
---

C/C++ 中的[前向声明](https://en.wikipedia.org/wiki/Forward_declaration)经常用于当头文件中的接口参数为指针时，做前向声明而非引用头文件。

但有时我们在使用别人提供的头文件时，可能会遇到匿名 struct。

我们先来看下为什么有人会定义匿名结构体。

**1. 当定义树的节点结构:**

```C
struct Node {
    int value;
    struct Node *left;
    struct Node *right;
};
```

**2. 有些人不想每次该结构时都输入 struct，于是:**

```C
struct Node_S {
    int value;
    struct Node_S *left;
    struct Node_S *right;
};
typedef struct Node_S Node;
```

**3. 然后有人觉得还可以让上面的代码更精简一些，于是:**

```C
typedef struct Node_S {
    int value;
    struct Node_S *left;
    struct Node_S *right;
} Node;
```

**4. 对于上面的 Node_S，由于结构体本身也使用了该结构，所有不能将 Node_S 省去，但是对于有些结构体，如 Person:**

```C
struct Person {
    bool sex;
    int age;
};
```

**5. 如果在使用该结构时不想敲 struct，于是:**

```C
typedef struct {
    bool sex;
    int age;
} Person;
```

这就是标题中所说的匿名结构体，对于这种结构体，乍一看好像做不到前向声明，但是:

*定义一个新的头文件 new_library.h*
```
#include "old_library.h"
struct PERSON: public Person {};
```

在之后的代码中，使用 PERSON，就可以进行前向声明了:

```
struct PERSON;
```

Bingo !!!

*注：当然这是在使用别人提供的头文件时不得已的做法，我们自己写的头文件还是尽量避免匿名结构体。*

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Forward declarations of unnamed struct][ref1]<br>
</span>

[ref1]: https://stackoverflow.com/questions/7256436/forward-declarations-of-unnamed-struct
