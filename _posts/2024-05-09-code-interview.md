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

<h4>两个链表相加</h4>

```C++
#include <iostream>

struct ListNode {
    int val;
    ListNode *next;

    ListNode(int val_) : val(val_), next(nullptr) {}
    ListNode(int val_, ListNode *next_) : val(val_), next(next_) {}
};

void printList(ListNode *head) {
    while (head) {
        std::cout << head->val << ' ';
        head = head->next;
    }

    std::cout << std::endl;
}

ListNode *reverse(ListNode *head) {
    if (head == nullptr || head->next == nullptr) {
        return head;
    }

    ListNode *pre = nullptr;
    while (head) {
        ListNode *cur = head;
        head = head->next;
        cur->next = pre;
        pre = cur;
    }

    return pre;
}

ListNode *add2list(ListNode *a, ListNode *b) {
    a = reverse(a);
    b = reverse(b);
    int carry = 0;
    ListNode dummy(-1);
    ListNode *pre = &dummy;

    while (a && b) {
        int tmp = a->val + b->val + carry;
        pre->next = new ListNode(tmp % 10);
        carry = tmp / 10;
        pre = pre->next;
        a = a->next;
        b = b->next;
    }

    ListNode *c = a != nullptr ? a : b;

    while (c) {
        int tmp = c->val + carry;
        pre->next = new ListNode(tmp % 10);
        carry = tmp / 10;
        pre = pre->next;
        c = c->next;
    }

    if (carry != 0) {
        pre->next = new ListNode(carry);
    }

    reverse(a);
    reverse(b);

    return reverse(dummy.next);
}

int main() {
    ListNode *a = new ListNode(1);
    ListNode *b = new ListNode(2, a);
    ListNode *c = new ListNode(3, b);

    ListNode *aa = new ListNode(8);
    ListNode *bb = new ListNode(9, aa);
    ListNode *cc = new ListNode(7, bb);

    ListNode *sum = add2list(c, cc);

    printList(sum);

    while (c) {
        ListNode *cur = c;
        c = c->next;
        delete cur;
    }

    while (cc) {
        ListNode *cur = cc;
        cc = cc->next;
        delete cur;
    }

    while (sum) {
        ListNode *cur = sum;
        sum = sum->next;
        delete cur;
    }
}

```

