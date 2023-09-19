---
title: perf ring buf
date: 2023-09-17 10:10:13
tags:
- bpf
- kernel 
---

## perf ring buf

BPF 程序需要将数据发到用户空间，一般很多都用 BPF perf buffer，但它有些问题，比如 浪费内存（因为 per-CPU），无法保证事件顺序等。

因此，内核 5.8 引入了 BPF ring buffer，它有如下优势：

1. 比 perf buffer 内存效率更高，保证事件顺序，性能也不输 perf buffer。
2. 提供了与 perf buffer 类似的 API，方便用户迁移，也提供了新的 reserve  和 commit 的 API，以实现更好的性能。

实验和真实环境的结果都说明，从 BPF 程序发送数据给用户空间应该首选 BPF ring buffer。

### ring buf 的改进

perfbuf 是 per-CPU circular buffers，实现高效的内核和用户空间的交互，但 per-CPU 的设计导致 **内存使用效率低下** 和 **事件顺序无法保证** 两个问题。

因此引入 ringbuf 来解决问题，ringbuf 是**“多生产者、单消费者”** MPSC 队列，可在多个CPU 之间共享和操作，perfbuf 支持的它都支持，包括

1. 可变长数据
2. Mmapped region 来高效地从 userspace 读数据，避免内存拷贝和 系统调用。
3. 支持 Epoll notification 和 busy-loop 两种数据获取方式

它还解决了 perfbuf 的下列问题：

1. 内存开销（memory overhead）；
2. 数据乱序；
3. 无效的处理逻辑和不必要的数据复制（extra data copying）。

具体改进方法

1. 使用 shared ring buf for each CPU
   1. 缓解 内存效率（buffer使用率） v.s. 数据丢失（buffer 满了导致丢失数据） 的 tradeoff，在不同 CPU 处理不均衡的时候，更明显。
   2. 扩展性好：CPU数量增加，per-CPU buffer 的总量会翻倍，但是 ringbuf 可能不需要增大就够用了。
   3. 保证事件顺序。
2. 提供 reserve 和 submit API 来预留数据
   1. 之前使用 perfbuf ，BPF 程序现需要初始化事件数据，然后送到用户空间，意味着数据需要拷贝两次
      1. 第一次复制到局部变量
      2. 第二次复制到 perfbuf 中
      3. 如果 perfbuf 里面没有足够空间了，那么第一步的数据是浪费的
   2. BPF ringbuf 提供了可选地 reserve submit方式
      1. 首先申请为数据预留空间 reserve()
      2. 预留成功后，应用直接将准备发送的数据放到 ringbuf 中，节省了 perfbuf 中的一次复制
      3. 将数据提交到用户空间十分高效，不会失败，不涉及额外的内存复制
      4. 如果buffer 空间不够而预留失败，则BPF 程序马上知道，不会再执行 perfbuf 中的第一次复制。

Code 示例：

```c
// tools/testing/selftests/bpf/progs/ringbuf_bench.c
// SPDX-License-Identifier: GPL-2.0
// Copyright (c) 2020 Facebook

#include <linux/bpf.h>
#include <stdint.h>
#include <bpf/bpf_helpers.h>

char _license[] SEC("license") = "GPL";

struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);
} ringbuf SEC(".maps");

const volatile int batch_cnt = 0;
const volatile long use_output = 0;

long sample_val = 42;
long dropped __attribute__((aligned(128))) = 0;

const volatile long wakeup_data_size = 0;

static __always_inline long get_flags()
{
	long sz;

	if (!wakeup_data_size)
		return 0;

	sz = bpf_ringbuf_query(&ringbuf, BPF_RB_AVAIL_DATA);
	return sz >= wakeup_data_size ? BPF_RB_FORCE_WAKEUP : BPF_RB_NO_WAKEUP;
}

SEC("fentry/__x64_sys_getpgid")
int bench_ringbuf(void *ctx)
{
	long *sample, flags;
	int i;

	if (!use_output) {
		for (i = 0; i < batch_cnt; i++) {
			// reserve sizeof(sample_val) in ringbuf
			sample = bpf_ringbuf_reserve(&ringbuf,
					             sizeof(sample_val), 0);
			if (!sample) {
				// ring buf is full, failed to add data
				// increment dropped count
				__sync_add_and_fetch(&dropped, 1);
			} else {
				// commit the sample_val to ringbuf
				*sample = sample_val;
				flags = get_flags();
				bpf_ringbuf_submit(sample, flags);
			}
		}
	} else {
		// dont use reserve/commit API
        // output API 里面就是 reserve/commit API
        // 上面用 reserve/commit -> reserve 返回地址，用户直接往里面填数据
        // 下面用 output -> reserve 地址后，将用户的数据拷贝到 ringbuf 里面
        // 可能好处就是用户可以直接往 reserve 返回地址里面填东西
		for (i = 0; i < batch_cnt; i++) {
			flags = get_flags();
			if (bpf_ringbuf_output(&ringbuf, &sample_val,
					       sizeof(sample_val), flags))
				__sync_add_and_fetch(&dropped, 1);
		}
	}
	return 0;
}
```



### 测试 cases

1. 常规场景

https://patchwork.ozlabs.org/project/netdev/patch/20200529075424.3139988-5-andriin@fb.com/

实现了 4 种 benchmark，对 ringbuf 和 perfbuf 都实现了两种

> Also implement a set of benchmarks for new BPF ring buffer and existing perf
> buffer. 4 benchmarks were implemented: 2 variations for each of BPF ringbuf
> and perfbuf:,
>   - rb-libbpf utilizes stock libbpf ring_buffer manager for reading data;
>   - rb-custom implements custom ring buffer setup and reading code, to
>     eliminate overheads inherent in generic libbpf code due to callback
>     functions and the need to update consumer position after each consumed
>     record, instead of batching updates (due to pessimistic assumption that
>     user callback might take long time and thus could unnecessarily hold ring
>     buffer space for too long);
>   - pb-libbpf uses stock libbpf perf_buffer code with all the default
>     settings, though uses higher-performance raw event callback to minimize
>     unnecessary overhead;
>   - pb-custom implements its own custom consumer code to minimize any possible
>     overhead of generic libbpf implementation and indirect function calls.
>
> Benchmarks that have only one producer implement optional back-to-back mode,
> in which record production and consumption is alternating on the same CPU.
> This is the highest-throughput happy case, showing ultimate performance
> achievable with either BPF ringbuf or perfbuf.

```shell
~/Documents/jammy/tools/testing/selftests/bpf$ ./benchs/run_bench_ringbufs.sh
Single-producer, parallel producer
==================================
rb-libbpf            1.901 ± 1.391M/s (drops 0.002 ± 0.004M/s)
rb-custom            3.226 ± 3.353M/s (drops 0.018 ± 0.035M/s)
pb-libbpf            0.023 ± 0.021M/s (drops 0.000 ± 0.000M/s)
pb-custom            0.036 ± 0.034M/s (drops 0.000 ± 0.000M/s)

Single-producer, parallel producer, sampled notification
========================================================
rb-libbpf            19.028 ± 5.974M/s (drops 0.000 ± 0.000M/s)
rb-custom            13.982 ± 12.444M/s (drops 0.000 ± 0.000M/s)
pb-libbpf            5.780 ± 5.679M/s (drops 0.000 ± 0.000M/s)
pb-custom            2.544 ± 1.797M/s (drops 0.000 ± 0.000M/s)

Single-producer, back-to-back mode
==================================
rb-libbpf            11.322 ± 9.532M/s (drops 0.000 ± 0.000M/s)
rb-libbpf-sampled    7.206 ± 2.298M/s (drops 0.000 ± 0.000M/s)
rb-custom            8.772 ± 0.146M/s (drops 0.000 ± 0.000M/s)
rb-custom-sampled    8.707 ± 0.134M/s (drops 0.000 ± 0.000M/s)
pb-libbpf            0.034 ± 0.001M/s (drops 0.000 ± 0.000M/s)
pb-libbpf-sampled    4.468 ± 0.030M/s (drops 0.000 ± 0.000M/s)
pb-custom            0.034 ± 0.000M/s (drops 0.000 ± 0.000M/s)
pb-custom-sampled    4.511 ± 0.043M/s (drops 0.000 ± 0.000M/s)

Ringbuf back-to-back, effect of sample rate
===========================================
rb-sampled-1         0.030 ± 0.001M/s (drops 0.000 ± 0.000M/s)
rb-sampled-5         0.149 ± 0.006M/s (drops 0.000 ± 0.000M/s)
rb-sampled-10        0.293 ± 0.009M/s (drops 0.000 ± 0.000M/s)
rb-sampled-25        0.710 ± 0.015M/s (drops 0.000 ± 0.000M/s)
rb-sampled-50        1.419 ± 0.042M/s (drops 0.000 ± 0.000M/s)
rb-sampled-100       2.730 ± 0.031M/s (drops 0.000 ± 0.000M/s)
rb-sampled-250       5.410 ± 0.155M/s (drops 0.000 ± 0.000M/s)
rb-sampled-500       8.382 ± 0.105M/s (drops 0.000 ± 0.000M/s)
rb-sampled-1000      18.470 ± 14.494M/s (drops 0.000 ± 0.000M/s)
rb-sampled-2000      24.252 ± 14.943M/s (drops 0.000 ± 0.000M/s)
rb-sampled-3000      69.032 ± 26.645M/s (drops 0.000 ± 0.000M/s)

Perfbuf back-to-back, effect of sample rate
===========================================
pb-sampled-1         0.059 ± 0.060M/s (drops 0.000 ± 0.000M/s)
pb-sampled-5         0.299 ± 0.172M/s (drops 0.000 ± 0.000M/s)
pb-sampled-10        0.576 ± 0.524M/s (drops 0.000 ± 0.000M/s)
pb-sampled-25        1.466 ± 1.089M/s (drops 0.000 ± 0.000M/s)
pb-sampled-50        0.905 ± 0.975M/s (drops 0.000 ± 0.000M/s)
pb-sampled-100       2.817 ± 1.659M/s (drops 0.000 ± 0.000M/s)
pb-sampled-250       4.348 ± 2.437M/s (drops 0.000 ± 0.000M/s)
pb-sampled-500       15.098 ± 22.562M/s (drops 0.000 ± 0.000M/s)
pb-sampled-1000      6.877 ± 5.430M/s (drops 0.000 ± 0.000M/s)
pb-sampled-2000      4.876 ± 5.380M/s (drops 0.000 ± 0.000M/s)
pb-sampled-3000      9.280 ± 13.732M/s (drops 0.000 ± 0.000M/s)

Ringbuf back-to-back, reserve+commit vs output
==============================================
reserve              17.154 ± 5.395M/s (drops 0.000 ± 0.000M/s)
output               6.493 ± 7.867M/s (drops 0.000 ± 0.000M/s)

Ringbuf sampled, reserve+commit vs output
=========================================
reserve-sampled      7.559 ± 5.264M/s (drops 0.000 ± 0.000M/s)
output-sampled       5.649 ± 0.207M/s (drops 0.000 ± 0.000M/s)

Single-producer, consumer/producer competing on the same CPU, low batch count
=============================================================================
rb-libbpf            0.026 ± 0.001M/s (drops 0.000 ± 0.000M/s)
rb-custom            0.026 ± 0.001M/s (drops 0.000 ± 0.000M/s)
pb-libbpf            0.026 ± 0.001M/s (drops 0.000 ± 0.000M/s)
pb-custom            0.026 ± 0.001M/s (drops 0.000 ± 0.000M/s)

Ringbuf, multi-producer contention
==================================
rb-libbpf nr_prod 1  6.278 ± 0.226M/s (drops 0.000 ± 0.000M/s)
rb-libbpf nr_prod 2  9.459 ± 0.083M/s (drops 0.000 ± 0.000M/s)
rb-libbpf nr_prod 3  5.133 ± 0.134M/s (drops 0.000 ± 0.000M/s)
setting affinity to CPU #4 failed: 2
rb-libbpf nr_prod 4  Setting up benchmark 'rb-libbpf'... (drops Setting up benchmark 'rb-libbpf'...)
```



### 一些设计上的考虑 [看不懂]

参考 `Documentation/bpf/ringbuf.rst`

除了 single ring buf 被最终采用，还有两种候选方案被放弃了。

其中一个和 BPF_MAP_TYPE_PERF_EVENT_ARRAY 相近，将 BPF_MAP_TYPE_RINGBUF 表示为一个 ring buffers 的数组，但不强制要求 ”同一个 CPU“ 的规则，这和现有的 perf buffer的用法相近，但应用如果想要用任意的 Key 来在 ring buffer 里面查找的话，则不支持。``BPF_MAP_TYPE_HASH_OF_MAPS`` 解决了这个问题。

此外，考虑到BPF环缓冲区的性能，许多用例只会选择在所有 CPU 之间共享一个简单的单个环缓冲区，对于这种情况，当前的方法可能是多余的。

另一种方法可以与 BPF MAP 一起引入一个新概念来表示通用 “container” 对象，该对象不一定具有具有 CRUD 接口。 这种方法将增加许多额外的基础支持，必须为 observability 和 verify support 加一堆代码。 它还会使得 BPF 开发人员必须熟悉一些 libbpf 中的新语法等。 ``BPF_MAP_TYPE_RINGBUF`` 不支持查找/更新/删除操作，但其他映射类型也不支持（例如队列和堆栈；数组不支持删除等）。

