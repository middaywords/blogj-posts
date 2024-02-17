---
title: linux traffic control - fair queue(tc fq)
date: 2024-01-02 09:06:49
tags:
- linux
- tc
- network
---

# linux tc 系列(3) - fair queue

最近在研究 EDT，EDT 实现一般是通过 tc bpf + tc fq 实现。



## Parameters

```c
static int fq_init(struct Qdisc *sch, struct nlattr *opt,
		   struct netlink_ext_ack *extack)
{
	struct fq_sched_data *q = qdisc_priv(sch);
	int err;
    // 整个 sch fq qdisc 的 queue 长度
	sch->limit		= 10000;
    // 每个 flow 的 queue 长度
	q->flow_plimit		= 100;
	q->quantum		= 2 * psched_mtu(qdisc_dev(sch));
	q->initial_quantum	= 10 * psched_mtu(qdisc_dev(sch));
	q->flow_refill_delay	= msecs_to_jiffies(40);
	q->flow_max_rate	= ~0UL;
	q->time_next_delayed_flow = ~0ULL;
	q->rate_enable		= 1;
	q->new_flows.first	= NULL;
	q->old_flows.first	= NULL;
	q->delayed		= RB_ROOT;
	q->fq_root		= NULL;
	q->fq_trees_log		= ilog2(1024);
	q->orphan_mask		= 1024 - 1;
	q->low_rate_threshold	= 550000 / 8;

	q->timer_slack = 10 * NSEC_PER_USEC; /* 10 usec of hrtimer slack */

	q->horizon = 10ULL * NSEC_PER_SEC; /* 10 seconds */
	q->horizon_drop = 1; /* by default, drop packets beyond horizon */

	/* Default ce_threshold of 4294 seconds */
	q->ce_threshold		= (u64)NSEC_PER_USEC * ~0U;

	qdisc_watchdog_init_clockid(&q->watchdog, sch, CLOCK_MONOTONIC);

	if (opt)
		err = fq_change(sch, opt, extack);
	else
		err = fq_resize(sch, q->fq_trees_log);

	return err;
}
```

## Source code

### enqueue

```c
static int fq_enqueue(struct sk_buff *skb, struct Qdisc *sch,
                      struct sk_buff **to_free)
```



### dequeue

```c
static struct sk_buff *fq_dequeue(struct Qdisc *sch)
```



## atc paper on fair queue & multi-queue

https://www.usenix.org/conference/atc19/presentation/hedayati-queue

## references

1. LWN, pkt_sched: fq: Fair Queue packet scheduler https://lwn.net/Articles/564825/ 
2. https://ztex.medium.com/demystify-the-fair-queuing-fq-packet-scheduler-in-linux-kernel-6-7-060c2b2f6f1a
3. patch for horizon [[PATCH net-next\] net_sched: sch_fq: add horizon attribute (kernel.org)](https://lore.kernel.org/all/20200501055144.24346-1-edumazet@google.com/T/) 
4. https://www.usenix.org/conference/atc19/presentation/hedayati-queue
