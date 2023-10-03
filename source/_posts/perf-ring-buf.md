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

因此引入 ringbuf 来解决问题，ringbuf 是**“多 producer 、单 consumer ”** MPSC 队列，可在多个CPU 之间共享和操作，perfbuf 支持的它都支持，包括

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

### 源码实现解析



* 测试框架代码 tools/testing/selftests/bpf/bench.c
* ringbuf 测试代码 
  * tools/testing/selftests/bpf/benchs/bench_ringbufs.c
* ringbuf bpf 代码
  * tools/testing/selftests/bpf/progs/ringbuf_bench.c
* perfbuf bpf 代码
  * tools/testing/selftests/bpf/progs/perfbuf_bench.c


提供了 commit/reserve/output/discard API ，我觉得想法就是，通过拆分 reserve 和 commit/discard 两步，尽可能减小竞争区。

reserve 实现

```c
// kernel/bpf/ringbuf.c
static void *__bpf_ringbuf_reserve(struct bpf_ringbuf *rb, u64 size)
{
	unsigned long cons_pos, prod_pos, new_prod_pos, flags;
	u32 len, pg_off;
	struct bpf_ringbuf_hdr *hdr;

	if (unlikely(size > RINGBUF_MAX_RECORD_SZ))
		return NULL;

	len = round_up(size + BPF_RINGBUF_HDR_SZ, 8);
	if (len > rb->mask + 1)
		return NULL;

	cons_pos = smp_load_acquire(&rb->consumer_pos);

	if (in_nmi()) {
		if (!spin_trylock_irqsave(&rb->spinlock, flags))
			return NULL;
	} else {
		spin_lock_irqsave(&rb->spinlock, flags);
	}

	prod_pos = rb->producer_pos;
	new_prod_pos = prod_pos + len;

	/* check for out of ringbuf space by ensuring producer position
	 * doesn't advance more than (ringbuf_size - 1) ahead
	 */
	if (new_prod_pos - cons_pos > rb->mask) {
		spin_unlock_irqrestore(&rb->spinlock, flags);
		return NULL;
	}
    
    // prod_pos 是单调递增的值，只有在使用的时候对 buf size 求模
	hdr = (void *)rb->data + (prod_pos & rb->mask);
    // 记录 bpf record 相对于 bpf_ringbuf 结构体的偏差，之后方便用 record 找到 ringbuf
	pg_off = bpf_ringbuf_rec_pg_off(rb, hdr);
    // set BPF_RINGBUF_BUSY_BIT in len field，表示这个 buffer 被占用了
	hdr->len = size | BPF_RINGBUF_BUSY_BIT;
	hdr->pg_off = pg_off;

	/* pairs with consumer's smp_load_acquire() */
	smp_store_release(&rb->producer_pos, new_prod_pos);

	spin_unlock_irqrestore(&rb->spinlock, flags);
    
    // 返回数据区域
	return (void *)hdr + BPF_RINGBUF_HDR_SZ;
}
```

commit 的实现

```c
// kernel/bpf/ringbuf.c
static void bpf_ringbuf_commit(void *sample, u64 flags, bool discard)
{
	unsigned long rec_pos, cons_pos;
	struct bpf_ringbuf_hdr *hdr;
	struct bpf_ringbuf *rb;
	u32 new_len;

	hdr = sample - BPF_RINGBUF_HDR_SZ;
	rb = bpf_ringbuf_restore_from_rec(hdr);
	// BPF_RINGBUF_BUSY_BIT 是用来同步的，需要配合，其中 len 共 32 位
    // 其中 31 位用来表示 BUSY BIT，BUSY BIT set 说明，producer 还未将
    // 数据准备好，还没有提交。
	new_len = hdr->len ^ BPF_RINGBUF_BUSY_BIT;
    // len 的第 30 位 BPF_RINGBUF_DISCARD_BIT 表示是否 producer 希望用户丢弃
    // 数据，这常用于 all or nothing，类似 packet 里一个 BIT 错了，整个包就没用了
	if (discard)
		new_len |= BPF_RINGBUF_DISCARD_BIT;
    // 基于以上，bpf ringbuf 最大为 2^30 - 1, about 256 MB

	/* update record header with correct final size prefix */
	xchg(&hdr->len, new_len);

	/* if consumer caught up and is waiting for our record, notify about
	 * new data availability
	 */
	rec_pos = (void *)hdr - (void *)rb->data;
    // 这里的 consumer pos 和 producer pos 都是单调递增的，只有用的时候才会执行
    // & rb->mask 操作，表示求模取余
	cons_pos = smp_load_acquire(&rb->consumer_pos) & rb->mask;

	// flags 用于表示是否需要唤醒用户消费者进程
    if (flags & BPF_RB_FORCE_WAKEUP)
		irq_work_queue(&rb->work);
    // 如果 consumer 在等新的数据，就唤醒它。
	else if (cons_pos == rec_pos && !(flags & BPF_RB_NO_WAKEUP))
		irq_work_queue(&rb->work);
}

```

bpf 程序逻辑

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

// batch_cnt 表示测试中一次提交多少个 recrod
const volatile int batch_cnt = 0;
// use_output 表示使用 reserve/commit 还是 reserve API
const volatile long use_output = 0;

// 填充数据的内容
long sample_val = 42;
// 填充数据失败的次数
long dropped __attribute__((aligned(128))) = 0;

// 表示累积一定量数据则唤醒用户进程
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

用户态 bpf ringbuf  的处理

```c
// tools/lib/bpf/ringbuf.c
static int64_t ringbuf_process_ring(struct ring* r)
{
	int *len_ptr, len, err;
	/* 64-bit to avoid overflow in case of extreme application behavior */
	int64_t cnt = 0;
	unsigned long cons_pos, prod_pos;
	bool got_new_data;
	void *sample;

	cons_pos = smp_load_acquire(r->consumer_pos);
	do {
		got_new_data = false;
		prod_pos = smp_load_acquire(r->producer_pos);
		while (cons_pos < prod_pos) {
			len_ptr = r->data + (cons_pos & r->mask);
			len = smp_load_acquire(len_ptr);

			/* sample not committed yet, bail out for now */
            // 这里处理逻辑是顺序处理，数据没准备好就会一直阻塞
			if (len & BPF_RINGBUF_BUSY_BIT)
				goto done;

			got_new_data = true;
			cons_pos += roundup_len(len);
            // 如果设置了 DISCARD_BIT 就不处理了
			if ((len & BPF_RINGBUF_DISCARD_BIT) == 0) {
				sample = (void *)len_ptr + BPF_RINGBUF_HDR_SZ;
                // 调用用户定义的处理函数，可以看到这里是直接基于 sample 地址修改，没有拷贝
                // 然后这里是 record 逐个处理，如果处理时间太长，阻塞太久，可能导致提交的数据
                // 把 ringbuf 都占满了
				err = r->sample_cb(r->ctx, sample, len);
				if (err < 0) {
					/* update consumer pos and bail out */
					smp_store_release(r->consumer_pos,
							  cons_pos);
					return err;
				}
				cnt++;
			}

			smp_store_release(r->consumer_pos, cons_pos);
		}
	} while (got_new_data);
done:
	return cnt;
}
```

相对于 perf buf 的实现

```c
// tools/testing/selftests/bpf/progs/perfbuf_bench.c
// SPDX-License-Identifier: GPL-2.0
// Copyright (c) 2020 Facebook

#include <linux/bpf.h>
#include <stdint.h>
#include <bpf/bpf_helpers.h>
#include "bpf_misc.h"

char _license[] SEC("license") = "GPL";

struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(value_size, sizeof(int));
	__uint(key_size, sizeof(int));
} perfbuf SEC(".maps");

const volatile int batch_cnt = 0;

long sample_val = 42;
long dropped __attribute__((aligned(128))) = 0;

SEC("fentry/" SYS_PREFIX "sys_getpgid")
int bench_perfbuf(void *ctx)
{
	int i;

	for (i = 0; i < batch_cnt; i++) {
		if (bpf_perf_event_output(ctx, &perfbuf, BPF_F_CURRENT_CPU,
					  &sample_val, sizeof(sample_val)))
			__sync_add_and_fetch(&dropped, 1);
	}
	return 0;
}
```

bpf 内核态的拷贝: bpf_perf_event_output ->bpf_perf_event_output ->  perf_event_output -> perf_output_sample -> memcpy_common

用户态 libbpf 实现 perf_buffer__poll -> bpf_perf_event_read_simple -> memcpy

### More details on implementation

* mapping to handle wrap arround

```c
// kernel/bpf/ringbuf.c
	/* Each data page is mapped twice to allow "virtual"
	 * continuous read of samples wrapping around the end of ring
	 * buffer area:
	 * ------------------------------------------------------
	 * | meta pages |  real data pages  |  same data pages  |
	 * ------------------------------------------------------
	 * |            | 1 2 3 4 5 6 7 8 9 | 1 2 3 4 5 6 7 8 9 |
	 * ------------------------------------------------------
	 * |            | TA             DA | TA             DA |
	 * ------------------------------------------------------
	 *                               ^^^^^^^
	 *                                  |
	 * Here, no need to worry about special handling of wrapped-around
	 * data due to double-mapped data pages. This works both in kernel and
	 * when mmap()'ed in user-space, simplifying both kernel and
	 * user-space implementations significantly.
	 */
```

一个物理 page 映射到了两个虚拟 page，这样在读取 ringbuf 末端的连续地址的时候，也不需要特殊处理，而且实际也没有用两个 物理 page，只是页表里面多了一次映射。

* Producer_pos & consumer_pos: 两个 u64，每个独占一个 page，这是为了给用户和内核态的 page 赋予不同的权限。

```c
// kernel/bpf/ringbuf.c
struct bpf_ringbuf {
	wait_queue_head_t waitq;
	struct irq_work work;
	u64 mask;
	struct page **pages;
	int nr_pages;
	spinlock_t spinlock ____cacheline_aligned_in_smp;
	/* Consumer and producer counters are put into separate pages to allow
	 * mapping consumer page as r/w, but restrict producer page to r/o.
	 * This protects producer position from being modified by user-space
	 * application and ruining in-kernel position tracking.
	 */
	unsigned long consumer_pos __aligned(PAGE_SIZE);
	unsigned long producer_pos __aligned(PAGE_SIZE);
	char data[] __aligned(PAGE_SIZE);
};
```

* 后面在 6.1 提出了  user ringbuf，没有详细研究，大致功能就是可以从用户往内核态通过 bpf userringbuf 传数据

User ringbuf patch: [bpf: Define new BPF_MAP_TYPE_USER_RINGBUF map type · torvalds/linux@583c1f4 · GitHub](https://github.com/torvalds/linux/commit/583c1f420173f7d84413a1a1fbf5109d798b4faa) 

```c
We want to support a ringbuf map type where samples are published from
user-space, to be consumed by BPF programs. BPF currently supports a
kernel -> user-space circular ring buffer via the BPF_MAP_TYPE_RINGBUF
map type.  We'll need to define a new map type for user-space -> kernel,
as none of the helpers exported for BPF_MAP_TYPE_RINGBUF will apply
to a user-space producer ring buffer, and we'll want to add one or
more helper functions that would not apply for a kernel-producer
ring buffer.

This patch therefore adds a new BPF_MAP_TYPE_USER_RINGBUF map type
definition. The map type is useless in its current form, as there is no
way to access or use it for anything until we one or more BPF helpers. A
follow-on patch will therefore add a new helper function that allows BPF
programs to run callbacks on samples that are published to the ring
buffer.

Signed-off-by: David Vernet <void@manifault.com>
Signed-off-by: Andrii Nakryiko <andrii@kernel.org>
Acked-by: Andrii Nakryiko <andrii@kernel.org>
Link: https://lore.kernel.org/bpf/20220920000100.477320-2-void@manifault.com
```



### 测试 cases

我把测试在我的 intel mac i9 + Virtual box(ubuntu 22.04 VM，4 核) 上跑了一下

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



后面都是翻译 kernel 文档的，虽然看不太懂，就干脆记录下

### 一些设计上的考虑 [看不懂]

参考 `Documentation/bpf/ringbuf.rst`

除了 single ring buf 被最终采用，还有两种候选方案被放弃了。

其中一个和 BPF_MAP_TYPE_PERF_EVENT_ARRAY 相近，将 BPF_MAP_TYPE_RINGBUF 表示为一个 ring buffers 的数组，但不强制要求 ”同一个 CPU“ 的规则，这和现有的 perf buffer的用法相近，但应用如果想要用任意的 Key 来在 ring buffer 里面查找的话，则不支持。``BPF_MAP_TYPE_HASH_OF_MAPS`` 解决了这个问题。

此外，考虑到BPF环缓冲区的性能，许多用例只会选择在所有 CPU 之间共享一个简单的单个环缓冲区，对于这种情况，当前的方法可能是多余的。

另一种方法可以与 BPF MAP 一起引入一个新概念来表示通用 “container” 对象，该对象不一定具有具有 CRUD 接口。 这种方法将增加许多额外的基础支持，必须为 observability 和 verify support 加一堆代码。 它还会使得 BPF 开发人员必须熟悉一些 libbpf 中的新语法等。 ``BPF_MAP_TYPE_RINGBUF`` 不支持查找/更新/删除操作，但其他映射类型也不支持（例如队列和堆栈；数组不支持删除等）。

### 一些实现细节

这种 commit / reserve 允许多个 producer 以自然的方式（无论是在不同的 CPU 上，还是在同一 CPU 上/在同一个 BPF 程序中）保留独立的记录并使用它们，而不会阻塞其他 producer 。 这意味着，如果 BPF 程序被共享同一 ringbuffer 的另一个 BPF 程序中断，它们都将获得 reserve 的 record，并且可以使用它并独立 commit 。 这也适用于 NMI 上下文，除了由于在 reserve 使用自旋锁之外，在 NMI 上下文中，bpf_ringbuf_reserve() 可能无法获取锁，在这种情况下，即使环形缓冲区未满，reserve 也会失败。

ringbuf 内部实现为 2 的幂大小的循环缓冲区，具有两个逻辑且不断增加的计数器：

consumer_pos 显示 consumer 消耗了数据的逻辑位置；

producer_pos 表示所有 producer 保留的数据量。

每次保留记录时，“拥有”该记录的 producer 将成功地推进 producer pos。 不过，此时数据还没有准备好被使用。 每个记录都有 8 个字节的 record，其中包含 reserved record 的长度，以及两个额外 bits：BPF_RINGBUF_BUSY_BIT，表示该记录仍在处理中；BPF_RINGBUF_DISCARD_BIT，如果记录被丢弃，则可能在提交时设置。 在后一种情况下， consumer 应该跳过该记录并继续下一个 record。 record header 还对记录 ring buf data area 的 offset（in pages）进行编码。 这允许 bpf_ringbuf_commit()/bpf_ringbuf_discard() 仅接受指向 record 本身的指针，而不需要指向 ringbuf 本身的指针。 

 producer 计数器增量在自旋锁下序列化，因此 reserve 之间有严格的顺序。 另一方面，提交是完全无锁且独立的。 所有记录都按预订顺序可供 consumer 使用，但前提是所有先前的记录都已提交。 因此，速度慢的 producer 可以暂时推迟提交的记录，这些记录是稍后保留的。

一个有趣的实现是**如何在虚拟内存中连续两次连续映射数据区域**，从而显着简化（并因此加快） producer 和 consumer 的实现。 这使得不需要对必须在 ringbuf 数据区域末尾 wrap 的样本采取任何特殊措施，因为最后一个数据页之后的下一页将再次成为第一个数据页，因此样本仍然会显得完全连续 在虚拟内存中。 请参阅 bpf_ringbuf_area_alloc() 中的注释和简单的 ASCII 图以直观方式显示这一点。

```c
	/* Each data page is mapped twice to allow "virtual"
	 * continuous read of samples wrapping around the end of ring
	 * buffer area:
	 * ------------------------------------------------------
	 * | meta pages |  real data pages  |  same data pages  |
	 * ------------------------------------------------------
	 * |            | 1 2 3 4 5 6 7 8 9 | 1 2 3 4 5 6 7 8 9 |
	 * ------------------------------------------------------
	 * |            | TA             DA | TA             DA |
	 * ------------------------------------------------------
	 *                               ^^^^^^^
	 *                                  |
	 * Here, no need to worry about special handling of wrapped-around
	 * data due to double-mapped data pages. This works both in kernel and
	 * when mmap()'ed in user-space, simplifying both kernel and
	 * user-space implementations significantly.
	 */
```



BPF Ringbuf 与 Perf Ring Buffer 的另一个区别是新数据可用时的 custom notification通知。 仅当 consumer 已经赶上正在提交的记录时，bpf_ringbuf_commit() 会发送 new record 在 comit 后 available notification。 如果没有， consumer 仍然必须赶上，因此无论如何都会看到新数据，而不需要 extra poll notification。 基准测试（参见tools/testing/selftests/bpf/benchs/bench_ringbufs.c）表明，这可以实现非常高的吞吐量，而不必求助于“仅通知每个第N个样本”之类的技巧，而这对于perf缓冲区来说是必需的。 对于极端情况，当 BPF 程序需要更多手动控制通知时，commit/discard/output helper function 接受 BPF_RB_NO_WAKEUP 和 BPF_RB_FORCE_WAKEUP 标志，这些标志可以完全控制数据可用性的通知，但在使用此 API 时需要格外小心和勤勉。

consumer page + producer page + data pages

### Reference 

* https://www.kernel.org/doc/html/next/bpf/ringbuf.html kernel 文档
* https://arthurchiao.art/blog/bpf-ringbuf-zh/ 


