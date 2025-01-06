---
title: xdpdump
date: 2023-08-07 20:46:12
tags:
- bpf
- xdp
---

TBD

## xdpdump 源码解析

大概思路就是

1. 没有 xdp 加载在 hook 点上，则挂一个 xdp bpf 程序将 packet data 复制到用户态，用户态可以对抓取的数据 pcap trace 进行处理
2. 有 xdp 程序已经挂载的时候，使用 fexit 和 fentry 在 xdp 程序入口和出口分别挂载 bpf 程序，将数据包输出到用户态。
