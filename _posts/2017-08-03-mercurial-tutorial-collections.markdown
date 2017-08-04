---
layout: post
title: Mercurial Tutorial Collections
date: 2017-08-03 22:00:00 +0800
tags:
- hg
---

[Mercurial](https://www.mercurial-scm.org/) 是一个分布式版本控制系统，虽不如 git 使用广泛, 但有些项目的代码仓库使用它进行代码管理，如 [Nginx](https://www.nginx.com/resources/wiki/)，因此了解一下 Mercurial 是必要的。本文收集了一些 Mercurial 学习资料以便后续查看。

<h4>Quick Start</h4>

```shell
# Clone a project and push changes
$ hg clone https://www.mercurial-scm.org/repo/hello
$ cd hello
$ (edit files)
$ hg add (new files)
$ hg commit -m 'My changes'
$ hg push

# Create a project and commit
$ hg init (project-directory)
$ cd (project-directory)
$ (add some files)
$ hg add
$ hg commit -m 'Initial commit'
```

<h4>资料列表</h4>

- [Learning Mercurial in Workflows](https://www.mercurial-scm.org/guide)
- [A Tutorial on Using Mercurial](https://www.mercurial-scm.org/wiki/Tutorial)
- [Mercurial: the definitive guide](https://book.mercurial-scm.org/)
- [Understanding Mercurial](https://www.mercurial-scm.org/wiki/UnderstandingMercurial)
