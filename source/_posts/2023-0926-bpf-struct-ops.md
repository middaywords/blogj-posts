---
title: bpf struct_ops
date: 2023-09-26 12:31:43
tags:
- bpf
---

## bpf struct_ops

bpf struct ops is a bpf prog type introduced in linux 5.6 [patch](https://lwn.net/ml/netdev/20191231062037.280596-1-kafai@fb.com/)

it is infra to allow implementing specific kernel function pointers in BPF,  e.g. tcp_congestion_ops in BPF

so we can move tcp cc to user space, faster to test!

### details

eample bpf: https://elixir.bootlin.com/linux/v5.15.96/source/tools/testing/selftests/bpf/progs/bpf_dctcp.c 

tcp_congestion_ops [https://elixir.bootlin.com/linux/v5.15.96/source/include/net/tcp.h#L1039](https://elixir.bootlin.com/linux/v5.15.96/source/include/net/tcp.h) 

bpf prog should fill the required field in ops struct, they happens when different TCP events occured.

```c
// include/net/tcp.h
struct tcp_congestion_ops {
/* fast path fields are put first to fill one cache line */

	/* return slow start threshold (required) */
	u32 (*ssthresh)(struct sock *sk);

	/* do new cwnd calculation (required) */
	void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);

	/* call before changing ca_state (optional) */
	void (*set_state)(struct sock *sk, u8 new_state);

	/* call when cwnd event occurs (optional) */
	void (*cwnd_event)(struct sock *sk, enum tcp_ca_event ev);

	/* call when ack arrives (optional) */
	void (*in_ack_event)(struct sock *sk, u32 flags);

	/* hook for packet ack accounting (optional) */
	void (*pkts_acked)(struct sock *sk, const struct ack_sample *sample);

	/* override sysctl_tcp_min_tso_segs */
	u32 (*min_tso_segs)(struct sock *sk);

	/* call when packets are delivered to update cwnd and pacing rate,
	 * after all the ca_state processing. (optional)
	 */
	void (*cong_control)(struct sock *sk, const struct rate_sample *rs);


	/* new value of cwnd after loss (required) */
	u32  (*undo_cwnd)(struct sock *sk);
	/* returns the multiplier used in tcp_sndbuf_expand (optional) */
	u32 (*sndbuf_expand)(struct sock *sk);

/* control/slow paths put last */
	/* get info for inet_diag (optional) */
	size_t (*get_info)(struct sock *sk, u32 ext, int *attr,
			   union tcp_cc_info *info);

	char 			name[TCP_CA_NAME_MAX];
	struct module		*owner;
	struct list_head	list;
	u32			key;
	u32			flags;

	/* initialize private data (optional) */
	void (*init)(struct sock *sk);
	/* cleanup private data  (optional) */
	void (*release)(struct sock *sk);
} ____cacheline_aligned_in_smp;
```

Some hooks are necessary to implement and some are optional.

support in kernel side:

```c
/* Manage refcounts on socket close. */
void tcp_cleanup_congestion_control(struct sock *sk)
{
	struct inet_connection_sock *icsk = inet_csk(sk);

	if (icsk->icsk_ca_ops->release)
		icsk->icsk_ca_ops->release(sk);
	bpf_module_put(icsk->icsk_ca_ops, icsk->icsk_ca_ops->owner);
}
```

在一些 TCP 里面特定点，会先请求加载模块，然后如果钩子存在的话，会去调钩子

### example test

* example bpf prog

  * https://elixir.bootlin.com/linux/v5.15.96/source/tools/testing/selftests/bpf/progs/bpf_dctcp.c 

  * [https://elixir.bootlin.com/linux/v5.15.96/source/tools/testing/selftests/bpf/prog_tests/bpf_tcp_ca.c#L190](https://elixir.bootlin.com/linux/v5.15.96/source/tools/testing/selftests/bpf/prog_tests/bpf_tcp_ca.c) 

* run simple tests here

```shell
$ cd tools/testing/selftests/bpf; make
$ strace ./test_progs -a bpf_tcp_ca/dctcp -vv
```

### More thoughts

这个能力很通用，但是到现在也只有 congestion control 这个应用。可能其他 ops 的复杂度还是太高了，bpf 不好支持，比如 file_ops 之类的。

如果只是 ondemand 测试，livepatching 也可以做同样的事情吧，而且 CC 这种肯定也需要 thorough 的 baking。

### reference 

PR link: https://github.com/torvalds/linux/commit/417759f7d4cf44a5fb526fbafcc9372e3dbfc0ae
