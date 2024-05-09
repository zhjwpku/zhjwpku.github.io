---
layout: post
title: Code Interview
date: 2024-05-09 00:00:00 +0800
tags:
- interview
---

面试中常见的算法题

<h4>单链表反转</h4>

```C++
#include <iostream>

template <typename T>
struct ListNode {
    T val_;
    ListNode *next_;
    ListNode(T val) : val_(val), next_(nullptr) {};
    ListNode(T val, ListNode *next) : val_(val), next_(next) {};
};

template <typename T>
void printList(ListNode<T> *head) {
    while (head) {
        std::cout << head->val_ << ' ';
        head = head->next_;
    }
    std::cout << std::endl;
}

template <typename T>
ListNode<T>* reverse(ListNode<T> *head) {
    if (head == nullptr || head->next_ == nullptr) {
        return head;
    }

    ListNode<T> *pre = nullptr;
    while (head) {
        ListNode<T> *cur = head;
        head = head->next_;
        cur->next_ = pre;
        pre = cur;
    }

    return pre;
}

int main() {
    ListNode<int> a(1, nullptr);
    ListNode<int> b(2, &a);
    ListNode<int> c(3, &b);
    printList(&c);

    auto p = reverse(&c);

    printList(p);
}

```




