---
title: 'kernel notes: process'
date: 2023-08-31 23:01:50
tags:
- kernel
- linux
---

## kernel notes 1: process

进程：父子进程共享代码段，有独立的堆栈。

轻量级进程： 能共享一些资源，如地址空间、文件，需要同步机制来看到共享区的修改。

