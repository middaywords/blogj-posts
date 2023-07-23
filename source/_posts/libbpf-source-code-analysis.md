---
title: libbpf 源码解析 (1)
date: 2023-07-23 13:04:47
tags:
- bpf
- libbpf
---

## libbpf 源码解析 (1)

这一篇主要是 overview，从总体上介绍 libbpf 的加载 prog/map 等 object 的过程。（由于 map type, object type 过多，所以一篇文章也难以囊括所有，所以计划做成一个博客系列。）

这个系列的环境信息：

OS: linux 5.15.0-26-generic

libbpf version: commit [f7eb43b](https://github.com/libbpf/libbpf/commit/f7eb43b90f4c8882edf6354f8585094f8f3aade0)

### 如何使用 libbpf 编译加载 bpf 程序

深入源码之前，我们先简单说明下如何使用。

总的来说，使用 libbpf 程序加载分为几个步骤 (refer to [Liz Rice's book](https://github.com/lizrice/learning-ebpf/blob/main/chapter5/Makefile))

```makefile
TARGET = hello-buffer-config
ARCH = $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/')

BPF_OBJ = ${TARGET:=.bpf.o}
USER_C = ${TARGET:=.c}
USER_SKEL = ${TARGET:=.skel.h}

all: $(TARGET) $(BPF_OBJ) find-map
.PHONY: all 

$(TARGET): $(USER_C) $(USER_SKEL) 
	gcc -Wall -o $(TARGET) $(USER_C) -L../libbpf/src -l:libbpf.a -lelf -lz

%.bpf.o: %.bpf.c vmlinux.h
	clang \
	    -target bpf \
        -D __TARGET_ARCH_$(ARCH) \
	    -Wall \
	    -O2 -g -o $@ -c $<
	llvm-strip -g $@

$(USER_SKEL): $(BPF_OBJ)
	bpftool gen skeleton $< > $@

vmlinux.h:
	bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h

clean:
	- rm $(BPF_OBJ)
	- rm $(TARGET)
	- rm find-map

find-map: find-map.c
	gcc -Wall -o find-map find-map.c -L../libbpf/src -l:libbpf.a -lelf -lz
```

以上 Makefile 将按照几个步骤进行

1. 使用 bpftool 根据系统的 `sys/kernel/btf/vmlinux` 生成 vmlinux.h。bpf 程序中头文件包含 vmlinux.h



2. 使用 clang 编译 bpf 程序。生成 .bpf.o



3. 使用 bpftool 将 .bpf.o 解析，生成 skel.h 头文件。



4. 使用 gcc 和 libbpf 库，以及步骤 3 中生成的 skel 头文件，编译用户态 bpf 程序。



