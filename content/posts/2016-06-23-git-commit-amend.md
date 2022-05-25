---
title: 深入浅出'Git commit --amend'
description: "深入浅出Git单步悔棋操作"
categories:
    - Git
tags: [Git]
date: 2016-06-23 10:26:12
---

Git能够提供悔棋的奥秘在于Git的reset命令。`Git commit --amend` 可以理解为`Git reset` 的alias。
``` bash
Git commit --amend ‘this is amend commit’
```
等价于
``` bash
Git reset --soft HEAD^
```
