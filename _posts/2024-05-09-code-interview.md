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

<h4>删除链表中相邻重复的元素</h4>

比如: 3 -> 3 -> 3 -> 5 -> 1 -> 1 -> 4 删除之后结果为: 5 -> 4

```C++
#include <iostream>

struct ListNode {
    int val;
    ListNode *next;
    ListNode(int val_) : val(val_), next(nullptr) {}
    ListNode(int val_, ListNode *next_) : val(val_), next(next_) {}
};

ListNode* removeDuplicates(ListNode* head) {
    if (head == nullptr || head->next == nullptr) {
        return head;
    }

    ListNode dummy(0, head);
    ListNode *pre = &dummy;
    ListNode *cur = head;

    while (cur && cur->next) {
        bool hasDup = false;
        while (cur->next && cur->val == cur->next->val) {
            hasDup = true;
            cur = cur->next;
        }

        if (hasDup) {
            pre->next = cur->next;
        } else {
            // or pre = pre->next;
            pre = cur;
        }

        cur = cur->next;
    }

    return dummy.next;
}

void printList(ListNode* head) {
    while (head != nullptr) {
        std::cout << head->val << " ";
        head = head->next;
    }
    std::cout << std::endl;
}

int main() {
    // 3 3 3 5 1 1 4
    ListNode n1(4);
    ListNode n2(1, &n1);
    ListNode n3(1, &n2);
    ListNode n4(5, &n3);
    ListNode n5(3, &n4);
    ListNode n6(3, &n5);
    ListNode n7(3, &n6);


    printList(&n7);
    printList(removeDuplicates(&n7));
}

```

<h4>实现 LRUCache 类</h4>

题目描述:
- LRUCache(int capacity) 以正整数作为容量 capacity 初始化 LRU 缓存
- int get(int key) 如果关键字 key 存在于缓存中，则返回关键字的值，否则返回 -1。
- void put(int key, int value) 如果关键字已经存在，则变更其数据值；如果关键字不存在，则插入该组「关键字-值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

```C++
#include <iostream>
#include <list>
#include <unordered_map>
#include <utility>

class LRUCache {
public:
    LRUCache(int capacity) : capacity_(capacity) {}

    int get(int key) {
        auto it = cache_.find(key);
        if (it == cache_.end()) {
            return -1;
        } else {
            items_.splice(items_.begin(), items_, it->second.second);
            return it->second.first;
        }
    }

    void put(int key, int value) {
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            it->second.first = value;
            items_.splice(items_.begin(), items_, it->second.second);
        } else {
            if (cache_.size() == capacity_) {
                int temp = items_.back();
                items_.pop_back();
                cache_.erase(temp);
            }
            items_.push_front(key);
            cache_[key] = {value, items_.begin()};
        }
    }

private:
    int capacity_;
    std::list<int> items_;
    std::unordered_map<int, std::pair<int, std::list<int>::iterator>> cache_;
};

int main() {
    LRUCache lru(3);
    lru.put(1, 2);
    std::cout << lru.get(1) << std::endl; // expect 2
    lru.put(2, 3);
    lru.put(3, 4);
    lru.put(4, 5);
    std::cout << lru.get(1) << std::endl; // expect -1
    std::cout << lru.get(2) << std::endl; // expect 3
    lru.put(5, 6);
    std::cout << lru.get(3) << std::endl; // expect -1

    return 0;
}
```

<h4>实现 shared_ptr</h4>

references:

- [C++: shared_ptr and how to write your own](https://medium.com/analytics-vidhya/c-shared-ptr-and-how-to-write-your-own-d0d385c118ad)
- [Unique Pointer and implementation in C++](https://blog.devgenius.io/unique-pointer-and-implementation-in-c-ec6599a518e5)

```C++
#include <iostream>

template <typename T>
class my_shared_ptr {
private:
    T * ptr = nullptr;
    int * refCnt = nullptr;

public:
    // default construtor
    my_shared_ptr() : ptr(nullptr), refCnt(new int(0)) {}

    // constructor
    my_shared_ptr(T *ptr) : ptr(ptr), refCnt(new int(1)) {
        std::cout << "constructor called" << std::endl;
    }

    // copy constructor
    my_shared_ptr(const my_shared_ptr &obj) {
        std::cout << "copy constructor called" << std::endl;
        this->ptr = obj.ptr;
        this->refCnt = obj.refCnt;
        if (obj.ptr != nullptr) {
            (*this->refCnt)++;
        }
    }

    // copy assignment
    my_shared_ptr& operator=(const my_shared_ptr& obj) {
        std::cout << "copy assignment called" << std::endl;
        __cleanup__();
        this->ptr = obj.ptr;
        this->refCnt = obj.refCnt;
        if (obj.ptr != nullptr) {
            (*this->refCnt)++;
        }
    }

    // move constructor
    my_shared_ptr(my_shared_ptr && dyingObj) {
        this->ptr = dyingObj.ptr;
        this->refCnt = dyingObj.refCnt;

        dyingObj.ptr = nullptr;
        dyingObj.refCnt = nullptr;
    }

    // move assignment
    my_shared_ptr& operator=(my_shared_ptr && dyingObj) {
        __cleanup__();

        this->ptr = dyingObj.ptr;
        this->refCnt = dyingObj.refCnt;

        dyingObj.ptr = nullptr;
        dyingObj.refCnt = nullptr;
    }

    // destructor
    ~my_shared_ptr() {
        __cleanup__();
    }

    int get_count() const {
        return (*this->refCnt);
    }

    T* get() const {
        return this->ptr;
    }

    T* operator->() const {
		return this->ptr;
	}

    T& operator*() const {
        return *(this->ptr);
    }

private:

    void __cleanup__() {
        (*refCnt)--;
        if (*refCnt == 0) {
            if (ptr != nullptr) {
                delete ptr;
            }
            delete refCnt;
        }
    }
};

template <typename T>
class my_unique_ptr {
private:
    T * ptr;

public:
    // constructor
    explicit my_unique_ptr(T* ptr = nullptr) : ptr(ptr) {}

    // destructor
    ~my_unique_ptr() {
        delete ptr;
    }

    // copy constructor
    my_unique_ptr(const my_unique_ptr&) = delete;

    // copy assignment
    my_unique_ptr& operator=(const my_unique_ptr&) = delete;

    // move constructor
    my_unique_ptr(my_unique_ptr && other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }

    // move assignment
    my_unique_ptr& operator=(const my_unique_ptr&& other) {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
};

int main()
{
    my_shared_ptr<int> p(new int(10));
    std::cout << *p << ' ' << p.get_count() << std::endl;

    my_shared_ptr<int> pp = p;
    std::cout << *pp << ' ' << pp.get_count() << std::endl;
}
```
