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



## packet01

这一部分主要教学如何读包内容，进行解析。

XDP 需要解析 packet header，能看到这些参数，需要注意的是，这些数据是按照网络字节序来排列的。可以用 `bpf_ntohs` 来转化字节序。

```c
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex */
	__u32 rx_queue_index;  /* rxq->queue_index  */
};
```

这里面有几个 assingment

1. fixing the bouds checking error: 读写 [data, data_end] 的数据需要检查边界，否则过不了 verifier。
2. parsing the IP header
3. parsing the ICMPv6 header and reacting to it: 解析 icmp 包，对 request 且 sequence 为奇数的包予以通行，其他的都 drop， sequence 是 icmp 协议里面自带的，kernel struct 里面有定义 `include/uapi/linux/icmpv6.h`
4. adding vlan support: 加入 VLAN 的支持，能够识别 VLAN。
5. adding ipv4 support: 加入 ipv4 的支持。



## packet02

这一部分教学如何写包内容。

关于改包，有几个需要注意

1. 可能会 fail，因为内存不足，packet data 太小无法容纳 ethernet header。
2. Helper function 只能改 data pointer，XDP 需要做验证，之前 verifier 的验证信息讲全部失效。
3. 可能需要重新算 checksum
4. 改包: `bpf_xdp_adjust_tail()` 可以用来改 skb_tail. `bpf_xdp_adjust_head()` 可以用来改 skb_head。

Assginments 包括

1. 交换 src, dst port number，
2. 删除 vlan header
3. 插入 vlan header



## packet03



