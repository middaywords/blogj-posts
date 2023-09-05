---
title: skb details
date: 2023-09-01 11:40:18
tags:
- linux
- network
---

## skb details

Skb，就是 sk_buff，存网络包的数据结构，它有三个组成部分, sk_buff, linear-data buffer, paged-data(struct skb_shared_info)。申请 sk_buff 的时候，传入 linear data 长度。

linear data 就是 skb->data，一般来说，一个 skb 只需要一个 page，而对于比 IP 分段比一个 page 还长的情况，我们有两种做法

1. 可以增大 linear 区域，来容纳整个 IP 分段。
2. 可以拿一个 paged data area 来存放剩余的包里的数据，后面这种情况，只在 device 不支持 scatter-gather 的时候执行。



### reference

* https://zhuanlan.zhihu.com/p/626514905



