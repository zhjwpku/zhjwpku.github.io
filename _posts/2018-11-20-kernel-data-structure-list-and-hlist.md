---
layout: post
title: Linux 内核结构体之 list & hlist
date: 2018-11-20 20:00:00 +0800
tags:
- kernel
---

Linux 内核中提供了[链表][list]的实现，这其中包括了双向链表和用于哈希表的 hash list（hlist）。双向链表的实现采用侵入式的方式，链表节点不保存任何数据内容，而是将链表结构作为具体数据结构的成员；hlist 虽然有 pprev 和 next 成员，但它并不是双向链表，因为 pprev 指向的是前一个节点的 next 指针。

<h4>Linked List</h4>

此种链表只需一种结构体，即：

```
struct list_head {
    struct list_head *next, *prev;
};
```

常用的函数和宏列举如下：

```
#define LIST_HEAD_INIT(name) { &(name), &(name) }

// 初始化链表头
#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)

// 初始化一个链表节点
static inline void INIT_LIST_HEAD(struct list_head *list)
{
    WRITE_ONCE(list->next, list);
    list->prev = list;
}

// 在链表头后增加一个元素，即将新增元素插入到链表第一个位置，可用于实现栈结构
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}

// 在链表头前插入一个元素，即将新元素插入到链表最末，可用于实现队列结构
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}

// 获取包含链表节点 ptr 的 type 结构体指针，member 为 list_head 在 type 结构重的定义
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

// 获取链表的第一个结构体，ptr 为链表头
#define list_first_entry(ptr, type, member) \
    list_entry((ptr)->next, type, member)

// 获取链表的最后一个结构体，ptr 为链表头
#define list_last_entry(ptr, type, member) \
    list_entry((ptr)->prev, type, member)

// 遍历链表
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)
```

关于双向链表如果非要说一点需要注意的地方的话，那就是*双向链表的头不存储真实数据，它通常作为链表遍历的入口存在于系统的全局变量*。一个完整的使用例子见：[Linux Kernel Programming–Linked List][linkedlist]。

<h4>Hash List</h4>

Hash List 是为 Hash Table 设计的一种链表，其实上述的双向链表也能实现哈希表，但由于双向链表的头跟其它节点一样，都有两个指针，因此当 bucket 很大的时候，会浪费内存。hlist 实现为以下两种结构体：

```
struct hlist_head {
    struct hlist_node *first;
};

struct hlist_node {
    struct hlist_node *next, **pprev;
};
```

可以看出，相对于 list_head 的两个指针，hlist_head 仅需保存一个指针。hlist_node 的 next 指向下一个节点，但 pprev（指针的指针） 并不指向上一个节点，而是指向上一个节点的 next 指针，这样设计的好处是当要删除的节点是第一个节点，也可以通过 \*pprev = next 直接修改指针的指向（参见[2][htable]）。

hlist 常见的函数及宏列举如下：

```
#define HLIST_HEAD_INIT { .first = NULL }
// 初始化链表头
#define HLIST_HEAD(name) struct hlist_head name = {  .first = NULL }
#define INIT_HLIST_HEAD(ptr) ((ptr)->first = NULL)
static inline void INIT_HLIST_NODE(struct hlist_node *h)
{
    h->next = NULL;
    h->pprev = NULL;
}

// 在节点头之后添加节点 n，即操作后 n 为链表的第一个节点
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
    struct hlist_node *first = h->first;
    n->next = first;
    if (first)
        first->pprev = &n->next;
    WRITE_ONCE(h->first, n);
    n->pprev = &h->first;
}

// 在节点 next 之前添加节点 n
/* next must be != NULL */
static inline void hlist_add_before(struct hlist_node *n,
                    struct hlist_node *next)
{
    n->pprev = next->pprev;
    n->next = next;
    next->pprev = &n->next;
    WRITE_ONCE(*(n->pprev), n);
}

// 在节点 prev 之后增加节点 n
static inline void hlist_add_behind(struct hlist_node *n,
                    struct hlist_node *prev)
{
    n->next = prev->next;
    WRITE_ONCE(prev->next, n);
    n->pprev = &prev->next;

    if (n->next)
        n->next->pprev  = &n->next;
}

// 同 list_entry
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)

// 遍历 Hash List
#define hlist_for_each(pos, head) \
    for (pos = (head)->first; pos ; pos = pos->next)

```

由于 hlist 的这种设计，导致它不能在 O(1) 时间获取到尾节点，也不能进行反向遍历。这可能就是计算机中常见的 Trade Off 吧。

<h4>Hash List with bit lock</h4>

[hlist_bl][list_bl] 是 hlist 一个特殊版本。一般在使用 hlist 用作 hashtable 的时候，会给每个 hlist 定义一个 spinlock，而为了减少这种内存开销，hlist_bl 利用 hlist_bl_head->first 的地址最后一位来代替 spinlock，这种方式可行的原因是因为指针通常为4字节8字节对齐，最后一位一定为0。

其关键函数如下：

```
static inline struct hlist_bl_node *hlist_bl_first(struct hlist_bl_head *h)
{
    return (struct hlist_bl_node *)
        ((unsigned long)h->first & ~LIST_BL_LOCKMASK);
}

static inline void hlist_bl_set_first(struct hlist_bl_head *h,
                    struct hlist_bl_node *n)
{
    LIST_BL_BUG_ON((unsigned long)n & LIST_BL_LOCKMASK);
    LIST_BL_BUG_ON(((unsigned long)h->first & LIST_BL_LOCKMASK) !=
                            LIST_BL_LOCKMASK);
    h->first = (struct hlist_bl_node *)((unsigned long)n | LIST_BL_LOCKMASK);
}

static inline bool hlist_bl_empty(const struct hlist_bl_head *h)
{
    return !((unsigned long)READ_ONCE(h->first) & ~LIST_BL_LOCKMASK);
}

static inline void hlist_bl_lock(struct hlist_bl_head *b)
{
    bit_spin_lock(0, (unsigned long *)b);
}

static inline void hlist_bl_unlock(struct hlist_bl_head *b)
{
    __bit_spin_unlock(0, (unsigned long *)b);
}

static inline bool hlist_bl_is_locked(struct hlist_bl_head *b)
{
    return bit_spin_is_locked(0, (unsigned long *)b);
}
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Data Structures in the Linux Kernel —— Doubly linked list][dlinkedlist]<br>
2 [List, HList, and Hash Table][htable]<br>
3 [Use of double pointer in linux kernel Hash list implementation][hlistdpointer]<br>
4 [[01/52] kernel: add bl_list][hlist_bl_patch]<br>
5 [[02/35] kernel: add bl_list][hlist_bl_patch_2]
</span>

[list]: https://github.com/torvalds/linux/blob/master/include/linux/list.h
[list_bl]: https://github.com/torvalds/linux/blob/master/include/linux/list_bl.h
[linkedlist]: http://www.roman10.net/2011/07/28/linux-kernel-programminglinked-list/
[dlinkedlist]: https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html
[htable]: https://danielmaker.github.io/blog/linux/list_hlist_hashtable.html
[hlistdpointer]: https://stackoverflow.com/questions/3058592/use-of-double-pointer-in-linux-kernel-hash-list-implementation
[hlist_bl_patch]: https://lore.kernel.org/patchwork/patch/204448/
[hlist_bl_patch_2]: https://lore.kernel.org/patchwork/patch/220389/
