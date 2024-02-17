---
title: linux tc (1) - overview
date: 2023-12-16 18:01:49
tags:
- linux
- tc
---

# linux tc 系列(1) - overview

TC 可以先读 https://tldp.org/HOWTO/Traffic-Control-HOWTO/ 有一些 general 的概念。

## tc framework

tc 是在 L2 实现的，实现为 queue discipline，简称 qdisc。

以下内容参考 [Linux-Traffic-Control-Classifier-Action-Subsystem-Architecture.pdf (netfilter.org)](https://people.netfilter.org/pablo/netdev0.1/papers/Linux-Traffic-Control-Classifier-Action-Subsystem-Architecture.pdf)

关于 qdisc，有几个相关概念

* qdisc： 可以是 classful 或者 classless 的。 classful qdisc 会有多个 classes，通过 qdisc 的 filter 进行分类。classful 的 qdisc 会包含其他的 qdisc，形成一个层级结构。每个 qdisc 由一个 32-bit 的 id 来确定，高 16 bit 称为 major id， 低 16 位称为 minor id。
* Class: 可能是 queue 可能是 qdisc，queue 是叶子(classless qdisc)，可以执行排队规则， qdisc 则进入
* Filter：根据 packet，依据算法分类到下一级 class
* Action: 当 classifier filter 结果 match 的时候，执行对应的 action

一个 port(netdev)会有两个 default qdisc points，一个是 egress path 的 root qdisc ，一个是 ingress 的 qdisc。ingress 的 qdisc 一般不执行 traffic control，只做一些 processing。

example qdisc:

1. prio. Flows selected internally based on TOS or skb->priority fields. Work conserving scheduling algorithm based on strict priority sorting (meaning low prio packets may be starved). 
2. Pfifo. 单个 fifo queue
3. Red. Classless qdisc with scheduling based on the RED algorithm.  
4. tbf 令牌桶. Classless qdisc, non-work conserving scheduling algorithm for shaping that is based on token bucket algorithm. 
5. htb. Hierarchical(classful) qdisc extension to TBF. 
6. Sfq. Stochastic fair queueing.  
7. codel. Based on controlled delay algorithm. 
8. fq-codel. Extending codel with a sfq flavoring.
9. Netem. Provides a variety of network emulation functionality for testing protocols by allowing mucking around with the protocol properties and semantics 

我们以一条 tc 指令来说明

```shell
$ tc filter add dev $DEV parent 1:0 protocol ip prio 10 \ 
u32 match ip protocol 1 0xff \ 
classid 1:10 \ 
action ok
```

将 filter attach 到 $DEV 的端口，filter 放在 egress qdisc 上，classid 是 1:0。匹配 ipv4 协议数据包，filter priority 为 10，使用 u32 classifier，检查是否 match icmp 协议(protocol 1 就是 ICMP)，一旦 match 了，那么就分类到选择 qdisc 1:10，接受数据包。

关于 **filter**: 一个端口的qdisc的一种协议可能配置多个 filter，根据 priority 来按顺序使用。

比如上面的例子里面用的是 u32 classifier，也会有其他的 classifier，u32，fw（用 skb mark metadata 来 match），route（用 ip route 规则来 match），rsvp，basic（由多个 其他类型 classifier 组合），BPF，Flow（packet conntrack,user_id,group_id）组合使用，OpenFlow classifier（由 OpenFlow spec 定义），还有其他的。

每个 classifier/filter 都有数据结构，给一个包（skb），返回一个 判定结果，给出 action。

关于**action**：上面的例子中，当 match 的时候，就会选择 action: accept，于是可以送进 协议栈中进行处理。action 还包括 reschedule，重新 filter 一遍，于是会有 pipeline 式的执行过程 A|B|C|D。actions 包括:(nat, checksum, TBF policer, gact, pedit, mirred, vlan, skbedit, connmark, 等)

gact: generic action，包括 accept/drop 包

Pedit: packet editor,可以对包做一些 xor, or, and 之类的

Mirred: 重定向包到另一个端口

vlan：encap/decap VLAN tags

Skbedit: 修改 skb 的 metadata

Connmark: 将 netfilter connection tracking 的一些details 和 skb marks 联系起来。



## L2 进入 qdisc 前处理过程

* `__dev_queue_xmit`

```c
int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
{
	struct net_device *dev = skb->dev;
	struct netdev_queue *txq = NULL;
	struct Qdisc *q;
    
    // ...
    
	// 根据 net cgroup 设置 skb 优先级，/sys/fs/cgroup/net_prio 相关实现
	skb_update_prio(skb);

	// 计算 GSO 情况下的 bytes sent on wire
	qdisc_pkt_len_init(skb);
	// tcx 区分是 ingress/egress
	tcx_set_ingress(skb, false);
#ifdef CONFIG_NET_EGRESS
	if (static_branch_unlikely(&egress_needed_key)) {
		// L2 egress 的 netfilter
		if (nf_hook_egress_active()) {
			skb = nf_hook_egress(skb, &rc, dev);
			if (!skb)
				goto out;
		}

		netdev_xmit_skip_txqueue(false);

		nf_skip_egress(skb, true);
		// tc cls_act 在这里运行
		skb = sch_handle_egress(skb, &rc, dev);
		if (!skb)
			goto out;
		nf_skip_egress(skb, false);
		// 根据 bpf 程序设置的 queue_mapping 来决定
		if (netdev_xmit_txqueue_skipped())
			txq = netdev_tx_queue_mapping(dev, skb);
	}
#endif
	// 检查 IFF_XMIT_DST_RELEASE 标志，此标志允许内核释放 skb 的目标缓存。如果标志已禁用，将强制对 skb 进行引用计数
	if (dev->priv_flags & IFF_XMIT_DST_RELEASE)
		skb_dst_drop(skb);
	else
		skb_dst_force(skb);

	// 如果 bpf 已经设置了，那么不需要继续操作了，否则根据 driver 和 xps 或者 hash 来决定
	if (!txq)
		txq = netdev_core_pick_tx(dev, skb, sb_dev);

	// 获取网卡 netdev_queue 上绑定的 qdisc
	q = rcu_dereference_bh(txq->qdisc);

	trace_net_dev_queue(skb);
	if (q->enqueue) {
		rc = __dev_xmit_skb(skb, q, dev, txq);
		goto out;
	}

	// 隧道设备和 loopback device 没有 qdisc 直接发送
	if (dev->flags & IFF_UP) {
        // ...
        skb = dev_hard_start_xmit
	}
    // ...
}
EXPORT_SYMBOL(__dev_queue_xmit);
```

1. `skb_update_prio`: /sys/fs/cgroup/net_prio 相关实现  https://www.spinics.net/lists/netdev/msg179826.html
2. `qdisc_pkt_len_init`: 计算 GSO 情况下的 bytes sent on wire，对于 bandwidth estimation 会更准
3. `skb = sch_handle_egress(skb, &rc, dev);` : 运行 cls_act bpf
4. `netdev_tx_queue_mapping`: 如果 cls_act bpf 设置 queue_mapping，后面就不用再设置了。这里支持了用户自己执行 txq 的绑定。可以实现 pod mapping 到 txq 上
5. `netdev_core_pick_tx`: 如果 bpf 设置了 queue mapping，则这一步不需要再选 queue，否则走内核自己的选 queue 流程， 根据 xps, `ndo_select_queue()` of netdev driver, or skb hash  in `netdev_core_pick_tx()` 来选择 queue
6. `q = rcu_dereference_bh(txq->qdisc);` : 获取网卡 netdev_queue 上绑定的 qdisc，这个是可能控制面要修改之类的
7. 如果有 enqueue 规则，则执行 `__dev_xmit_skb(skb, q, dev, txq);` 发送逻辑



---

* `__dev_xmit_skb`

```c
static inline int __dev_xmit_skb(struct sk_buff *skb, struct Qdisc *q,
				 struct net_device *dev,
				 struct netdev_queue *txq)
{
	// 拿到 qdisc 的 lock
	spinlock_t *root_lock = qdisc_lock(q);
    // ...

	// NOLOCK 的 qdisc，如 pfifo_fast
	if (q->flags & TCQ_F_NOLOCK) {
        // bypass qdisc
		if (q->flags & TCQ_F_CAN_BYPASS && nolock_qdisc_is_empty(q) &&
		    qdisc_run_begin(q)) {
            // ...
        }
		// 不能 bypass，还是得走 qdisc，调用 qdisc->enqueue，不过这是 NOLOCK 的情况
		rc = dev_qdisc_enqueue(skb, q, &to_free, txq);
		qdisc_run(q);
        // ...
    }

    contended = qdisc_is_running(q) || IS_ENABLED(CONFIG_PREEMPT_RT);
	if (unlikely(contended))
		spin_lock(&q->busylock);

	// 给 qdisc 上锁
	spin_lock(root_lock);
	if (unlikely(test_bit(__QDISC_STATE_DEACTIVATED, &q->state))) {
		// 未启用 qdisc，直接drop
		__qdisc_drop(skb, &to_free);
		rc = NET_XMIT_DROP;
	} else if ((q->flags & TCQ_F_CAN_BYPASS) && !qdisc_qlen(q) &&
		   qdisc_run_begin(q)) {
        // bypass 的情况
        if (sch_direct_xmit(skb, q, dev, txq, root_lock, true)) {
			if (unlikely(contended)) {
				spin_unlock(&q->busylock);
				contended = false;
			}
			__qdisc_run(q);
		}

		qdisc_run_end(q);
    } else {
        // 正常逻辑传输
		rc = dev_qdisc_enqueue(skb, q, &to_free, txq);
		if (qdisc_run_begin(q)) {
			if (unlikely(contended)) {
				spin_unlock(&q->busylock);
				contended = false;
			}
			__qdisc_run(q);
			qdisc_run_end(q);
		}
	}
    return rc;
}
```

这里分为几种情况，

1. `if (q->flags & TCQ_F_NOLOCK)`: 是否支持 TCQ_F_NOLOCK，支持说明不需要 qdisc lock 上锁。

   1. 对于不需要上锁的情况，也需要判断 qdisc 是否能够被 bypass，可以的话直接调用 `sch_direct_xmit` 进行传输
   2. 不能 bypass 则还是得走 qdisc，`dev_qdisc_enqueue()` 会调用 enqueue，`qdisc_run()` 中会调用 dequeue。

2. 不支持 TCQ_F_NOLOCK

   1. `spin_lock(root_lock);` 给 qdisc 上锁

   2. `test_bit(__QDISC_STATE_DEACTIVATED, &q->state)`: 未启用 qdisc，则直接 drop

   3. `(q->flags & TCQ_F_CAN_BYPASS) && !qdisc_qlen(q) &&qdisc_run_begin(q)`: 可以 bypass，则直接调用 `sch_direct_xmit` 进行传输

   4. 不能 bypass 则走 `dev_qdisc_enqueue` 逻辑传输

      

---

我们这里看到往后面进入 qdisc 之前，主要有几种情况，

1. bypass 直接传输: `sch_direct_xmit()`

```c
			qdisc_bstats_cpu_update(q, skb);
			if (sch_direct_xmit(skb, q, dev, txq, NULL, true) &&
			    !nolock_qdisc_is_empty(q))
                // 判断成立说明 feel free to send more pkts，可以继续发包
				__qdisc_run(q);
            // 完成后将状态恢复
			qdisc_run_end(q);
```

```c
bool sch_direct_xmit(struct sk_buff *skb, struct Qdisc *q,
		     struct net_device *dev, struct netdev_queue *txq,
		     spinlock_t *root_lock, bool validate) {
    // ..
    if (likely(skb)) {
    HARD_TX_LOCK(dev, txq, smp_processor_id());
    if (!netif_xmit_frozen_or_stopped(txq))
        skb = dev_hard_start_xmit(skb, dev, txq, &ret);
    // ...
}
```

`dev_hard_start_xmit()` 中调用网络驱动发送逻辑

2.  enqueue: `dev_qdisc_enqueue()`，里面即调用了 qdisc->enqueue

```c
		rc = dev_qdisc_enqueue(skb, q, &to_free, txq);
```

```c
static int dev_qdisc_enqueue(struct sk_buff *skb, struct Qdisc *q,
			     struct sk_buff **to_free,
			     struct netdev_queue *txq)
{
    // ...
	rc = q->enqueue(skb, q, to_free) & NET_XMIT_MASK;
    // ...
}
```

`q->enqueue(skb, q, to_free)` 进入 qdisc 的 enqueue 方法。

3. dequeue:  `qdisc_run_begin()` + `__qdisc_run()` + `qdisc_run_end()`

```c
		if (qdisc_run_begin(q)) {
			if (unlikely(contended)) {
				spin_unlock(&q->busylock);
				contended = false;
			}
			__qdisc_run(q);
			qdisc_run_end(q);
		}
```

`qdisc_run_begin()` + `qdisc_run_end()` 是设置 qdisc  的运行状态，主要的处理逻辑在 `__qdisc_run()` 中。

```c
void __qdisc_run(struct Qdisc *q)
{
	int quota = READ_ONCE(dev_tx_weight);
	int packets;

	while (qdisc_restart(q, &packets)) {
		quota -= packets;
		if (quota <= 0) {
			if (q->flags & TCQ_F_NOLOCK)
				set_bit(__QDISC_STATE_MISSED, &q->state);
			else
				__netif_schedule(q);

			break;
		}
	}
}
```

这里会一直调用 `qdisc_restart` 直到 quota 耗尽，再调用 `sch_direct_xmit()` 进行发送。

```c
/*
 * NOTE: Called under qdisc_lock(q) with locally disabled BH.
 *
 * running seqcount guarantees only one CPU can process
 * this qdisc at a time. qdisc_lock(q) serializes queue accesses for
 * this queue.
 *
 *  netif_tx_lock serializes accesses to device driver.
 *
 *  qdisc_lock(q) and netif_tx_lock are mutually exclusive,
 *  if one is grabbed, another must be free.
 *
 * Note, that this procedure can be called by a watchdog timer
 *
 * Returns to the caller:
 *				0  - queue is empty or throttled.
 *				>0 - queue is not empty.
 *
 */
static inline bool qdisc_restart(struct Qdisc *q, int *packets)
{
	spinlock_t *root_lock = NULL;
	struct netdev_queue *txq;
	struct net_device *dev;
	struct sk_buff *skb;
	bool validate;

	/* Dequeue packet */
    // 从 qdisc 中获取 skb
	skb = dequeue_skb(q, &validate, packets);
	if (unlikely(!skb))
		return false;

	if (!(q->flags & TCQ_F_NOLOCK))
		root_lock = qdisc_lock(q);

	dev = qdisc_dev(q);
	txq = skb_get_tx_queue(dev, skb);

	return sch_direct_xmit(skb, q, dev, txq, root_lock, validate);
}
```





## reference 

1. TC 介绍
   1. arch overview https://people.netfilter.org/pablo/netdev0.1/papers/Linux-Traffic-Control-Classifier-Action-Subsystem-Architecture.pdf
   2. LINUX TC HOWTO https://tldp.org/HOWTO/Traffic-Control-HOWTO/
   3. https://arthurchiao.art/blog/traffic-control-from-queue-to-edt-zh/#51-cilium-pod-egress-%E9%99%90%E9%80%9F
2. tc 源码分析
   1. https://abcdxyzk.github.io/blog/2020/05/22/kernel-qdisc/
   1. https://www.iteye.com/blog/cxw06023273-867318
   1. https://guanjunjian.github.io/2017/11/20/study-10-hierachical-token-bucket-source-code-1/
3. 关于 EDT
   1. 2018
      * Netdev: https://www.youtube.com/watch?v=MAni0_lN7zE
      * oct: https://documents.pub/document/oct-2018-david-wetherall-presenter-nandita-dukkipati-talks2018davidwetherall.html?page=1
      * http://vger.kernel.org/lpc_bpf2018_talks/lpc-bpf-2018-shaping.pdf
   2. 2020
      * https://legacy.netdevconf.info/0x14/pub/papers/55/0x14-paper55-talk-paper.pdf
   3. cilium 的 EDT 设计
      * https://isovalent.com/blog/post/addressing-bandwidth-exhaustion-with-cilium-bandwidth-manager/
      * https://cilium.io/use-cases/bandwidth-optimization/
   4.  talks & papers
      1. https://netdevconf.info/0x17/sessions/talk/ebpf-qdisc-a-generic-building-block-for-traffic-control.html
      2. 
