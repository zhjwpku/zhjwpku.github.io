---
layout: post
title: 面试准备
date: 2020-06-30 00:00:00 +0800
tags:
- interview
---

2020年6月30日，下决心换工作了。想换工作的原因不便多说。希望利用下半年的周末进行面试前的系统准备，本文记录一下准备的过程及后续面试遇到的高水平面试题，希望对你有所启发和帮助。

<h4>一、数据结构</h4>

**[Red–black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)**

考点1: 红黑树的特性

除了具有 BST 的特性之外，红黑树还具有以下特性:

- 每个节点非红即黑
- 根节点是黑色的
- 所有的叶子节点(nil)都是黑色的
- 红色节点的孩子都是黑色的
- 从任意节点出发，到其每一个叶子节点的路径所包含的黑色节点个数相同

*可以通过不多于三次翻转操作来保证上述的特性*

考点2: 每一棵红黑树都有一棵等同的 2-3-4 B-tree。C++ 的 map 和 set 是用红黑树实现的

[Red/Black Tree Visualization](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html)

**[AVL tree](https://en.wikipedia.org/wiki/AVL_tree)**

AVL tree 是最先发明的平衡二叉树，除具备 BST 的特性之外，它还具有以下特性:

- 任意节点的两个子树的高度差小于等于 1
- 需要额外两个 bit 存储 balance factor

考点1: 任意一棵 AVL tree 都能着色为一棵 red-black tree，但反过来不成立

考点2: 在读密集的应用中，AVL tree 效率要稍优于 red-black tree; 但由于 red-black tree 的约束更宽松（从*考点1*可以看出），red-black tree 在写密集的应用中效率要高于 AVL tree。

[AVL Tree Visualzation](https://www.cs.usfca.edu/~galles/visualization/AVLtree.html)

**Trie(Prefix Tree)**

**Radix Tree (Compact Trie)**

**B+ tree**

B tree 和 B+ tree 的考点大概会问 B tree 和 B+ tree 的差别，[LMDB][lmdb] 内部使用了 B+ tree，阅读 lmdb 源码、分析其内部实现以加深理解，并写不少于两篇源码分析的博文。

[B+ Tree Visualization](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

**lsm tree**

hbase、cassendra、leveldb 及 rocksdb 底层数据结构，看过相关论文，后面会把 leveldb 源码读一遍，并写不少于三篇源码分析的博文。

**Skip list**

[Skip List Visualization](https://people.ok.ubc.ca/ylucet/DS/SkipList.html)

<h4>二、算法</h4>

**上机或白板做题**

现在每天都会做一两道 leetcode，要特别加强如下几种题型的练习和总结:

- 动态规划
- 线段树
- 单调栈

**共识算法**

详见 [一致性(Consensus)][consensus]。

<h4>三、C++</h4>

[C/C++ Learning Resources](/2019/10/07/cpp-learning-resources.html) 记录了 C++ 学习过程。

<h4>四、网络</h4>

**tcp/ip**

**epoll**

[libevent 源码分析][libevent-source-code-analysis]

**aio**

**RDMA**

<h4>五、内存管理</h4>

**内存池**

**slab 分配算法**

<h4>六、并发</h4>

**线程池**

[Thread Pool](https://zhjwpku.com/category/2020/09/12/thread-pool.html)

<h4>七、文件系统</h4>

**NFS**

**ext3**

<h4>八、编译连接</h4>

**cmake**

**sanitizers**

<h4>九、Ceph</h4>

期望能找一个 Ceph 相关的工作，因为从 16 年做 GSoC 之后一直在关注这个项目，只是一直没有深度参与，现在极度渴望阅读 Ceph 源码并深入社区。Ceph 我会分如下几个方面进行准备:

1. 阅读 Ceph 相关的 Paper
2. 跟着 Ceph Code Walkthrough 熟悉 3 至 5 个模块
3. 把 msg 模块阅读并做源码分析

<h4>十、项目</h4>

在华为的工作比较琐碎，之后在做[简历][resume]的时候需要把项目这块好好整理一下。

<h4>十一、常用工具</h4>

常用的工具能体现一个程序员的工作方式，甚至能简介反映出其工作效率。

**Linux 常用命令行工具**

1. awk
2. sed
3. gdb
4. perf

[resume]: /resume
[lmdb]: https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database
[consensus]: https://github.com/zhjwpku/papers-notebook#%E4%B8%80%E8%87%B4%E6%80%A7consensus
[libevent-source-code-analysis]: https://zhjwpku.com/2020/08/20/libevent-source-code-analysis.html