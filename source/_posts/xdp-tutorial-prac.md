---
title: xdp-tutorial-prac
date: 2024-03-17 16:27:09
tags:
- xdp
- bpf
---

# xdp tutorial 

## basic01

尝试 load 和 unload 示例 xdp 程序。

```
➜  basic01-xdp-pass git:(master) ✗ sudo ./xdp_pass_user -d enp0s3   
libbpf: elf: skipping unrecognized data section(7) xdp_metadata
libbpf: prog 'xdp_pass': BPF program load failed: Invalid argument
libbpf: prog 'xdp_pass': failed to load: -22
libbpf: failed to load object 'xdp-dispatcher.o'
libbpf: elf: skipping unrecognized data section(7) xdp_metadata
libbpf: elf: skipping unrecognized data section(7) xdp_metadata
libbpf: elf: skipping unrecognized data section(7) xdp_metadata
Success: Loading XDP prog name:xdp_prog_simple(id:583) on device:enp0s3(ifindex:2)
➜  basic01-xdp-pass git:(master) ✗ sudo ./xdp_pass_user -d enp0s3 --unload-all  
Success: Unloading XDP prog name: xdp_prog_simple
```

这一堆的 warning 是因为 multiprog load 失败，fall back 到 single program 的 load 了。



## basic02

load 一个 xdp_abort 程序。

```c
/* Assignment#2: Add new XDP program section that use XDP_ABORTED */
SEC("xdp")
int  xdp_abort_func(struct xdp_md *ctx)
{
	return XDP_ABORTED;
}
```

`XDP_ABORTED` different from `XDP_DROP`, `XDP_ABORTED` can trigger `xdp:xdp_exception`



## basic03

实现一个 PerCPU map 读取不同 action 的 bytes/packets.



## basic04

这回把 loader 和 读取数据的 stats_poll 逻辑分开成了两个进程。

这种方式有时候有问题，因为 program load/unload 的时候可能用其他 map 了，所以这里需要使用 reuse map 来避免更换 map 后丢失数据。

candidate solution [basic04 reuse pinned map failed. · Issue #402 · xdp-project/xdp-tutorial (github.com)](https://github.com/xdp-project/xdp-tutorial/issues/402) 

据说后面 libbpf 也实现了 reuse map automatically, https://github.com/xdp-project/xdp-tutorial/issues/125#issuecomment-626614450





