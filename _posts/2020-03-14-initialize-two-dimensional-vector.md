---
layout: post
title: 初始化二维 vector 的几种方法
date: 2020-03-14 20:00:00 +0800
tags:
- C/C++
---

本文列举几种二维 vector 的初始化方法。

**Fill Constructor**

```C++
#define M 4
#define N 4

// one step, recommended
std::vector<std::vector<int>> matrix(M, std::vector<int>(N, 0));

// two steps
std::vector<int> row(N, 0);
std::vector<std::vector<int>> matrix2(M, row);
```

**resize function**

```C++
#define M 4
#define N 4

std::vector<std::vector<int>> matrix(M);
for (int i = 0 ; i < M ; i++)
    matrix[i].resize(N, 0);

std::vector<std::vector<int>> matrix2;
matrix2.resize(M, std::vector<int>(N, 0));
```

**C++ Initializer lists**

```C++
std::vector<std::vector<int>> matrix {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12},
    {13, 14, 15, 16}
};
```

**Print a vector**

```C++
template<typename T>
void printVector(const std::vector<std::vector<T>> &matrix) {
    for (auto row: matrix) {
        for (auto val: row) {
            std::cout << val << ' ';
        }
        std::cout << '\n';
    }
}
```