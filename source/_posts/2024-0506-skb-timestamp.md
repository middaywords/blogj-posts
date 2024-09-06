---
title: skb timestamps
date: 2024-05-06 18:25:50
tags:
- linux
- kernel
- network
---

最近在看 fathom 的实现，

* https://github.com/grpc/grpc/blob/fb72f1df08e4bbdad6eba93412135856ea45abe8/src/core/lib/event_engine/posix_engine/posix_endpoint.cc#L842
* https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=618896e6d00773d6d50e0b19f660af22fa26cd61
* tools/testing/selftests/net/txtimestamp.c 

发现其中用到了一些 kernel 里的 timestamp，这里做一些记录

tx 相关的 timestamp 如下，注释已经说明了其大概位置。

```c

/* Definitions for tx_flags in struct skb_shared_info */
enum {
	/* generate hardware time stamp */
	SKBTX_HW_TSTAMP = 1 << 0,

	/* generate software time stamp when queueing packet to NIC */
	SKBTX_SW_TSTAMP = 1 << 1,

	/* device driver is going to provide hardware time stamp */
	SKBTX_IN_PROGRESS = 1 << 2,

	/* generate hardware time stamp based on cycles if supported */
	SKBTX_HW_TSTAMP_USE_CYCLES = 1 << 3,

	/* generate wifi status information (where possible) */
	SKBTX_WIFI_STATUS = 1 << 4,

	/* determine hardware time stamp based on time or cycles */
	SKBTX_HW_TSTAMP_NETDEV = 1 << 5,

	/* generate software time stamp when entering packet scheduling */
	SKBTX_SCHED_TSTAMP = 1 << 6,
};
```

初版实现中，还使用了 `SKBTX_ACK_TSTAMP` ，后面替代为了 `txstamp_ack`。



