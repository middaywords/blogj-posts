---
title: linux tc mq
date: 2024-01-08 08:44:47
tags:
- tc
- linux
- network
---

# linux tc 系列(4) - mq

Kernel version: 5.15

## tc multiqueue

Tc 中有 multiqueue，还有 mq，两个其实是不同的东西，关于 tc multiqueue，`Documentation/networking/multiqueue.rst` 有相关叙述。

简单来说，qdisc support for multiqueue device，一种是给每个 NIC queue 配一个 pfifo_fast qdisc，另一种是一个 NIC queue 对应一个 band，用来 avoid head-of-line blocking（#可是一个 NIC 可能有 80 个queue呀，真的需要这么多 band）

```
// Documentation/networking/multiqueue.rst
// ...
Section 2: Qdisc support for multiqueue devices
===============================================

Currently two qdiscs are optimized for multiqueue devices.  The first is the
default pfifo_fast qdisc.  This qdisc supports one qdisc per hardware queue.
A new round-robin qdisc, sch_multiq also supports multiple hardware queues. The
qdisc is responsible for classifying the skb's and then directing the skb's to
bands and queues based on the value in skb->queue_mapping.  Use this field in
the base driver to determine which queue to send the skb to.

sch_multiq has been added for hardware that wishes to avoid head-of-line
blocking.  It will cycle though the bands and verify that the hardware queue
associated with the band is not stopped prior to dequeuing a packet.

On qdisc load, the number of bands is based on the number of queues on the
hardware.  Once the association is made, any skb with skb->queue_mapping set,
will be queued to the band associated with the hardware queue.


Section 3: Brief howto using MULTIQ for multiqueue devices
==========================================================

The userspace command 'tc,' part of the iproute2 package, is used to configure
qdiscs.  To add the MULTIQ qdisc to your network device, assuming the device
is called eth0, run the following command::

    # tc qdisc add dev eth0 root handle 1: multiq

The qdisc will allocate the number of bands to equal the number of queues that
the device reports, and bring the qdisc online.  Assuming eth0 has 4 Tx
queues, the band mapping would look like::

    band 0 => queue 0
    band 1 => queue 1
    band 2 => queue 2
    band 3 => queue 3

Traffic will begin flowing through each queue based on either the simple_tx_hash
function or based on netdev->select_queue() if you have it defined.
```



## tc mq

`net/sched/sch_mq.c` 中定义了  相关的行为。总的来说它定义了  qdisc ops 和 class ops，

```c
static const struct Qdisc_class_ops mq_class_ops = {
	.select_queue	= mq_select_queue, // 绑定 queue
	.graft		= mq_graft, // 嫁接 qdisc
	.leaf		= mq_leaf,
	.find		= mq_find,
	.walk		= mq_walk,
	.dump		= mq_dump_class,
	.dump_stats	= mq_dump_class_stats,
};

struct Qdisc_ops mq_qdisc_ops __read_mostly = {
	.cl_ops		= &mq_class_ops,
	.id		= "mq",
	.priv_size	= sizeof(struct mq_sched),
	.init		= mq_init,
	.destroy	= mq_destroy,
	.attach		= mq_attach,
	.change_real_num_tx = mq_change_real_num_tx,
	.dump		= mq_dump,
	.owner		= THIS_MODULE,
};
```

具体看一下 init 过程：

```c
static int mq_init(struct Qdisc *sch, struct nlattr *opt,
		   struct netlink_ext_ack *extack)
{
    // ...
    // 非 root qdisc 不支持
	if (sch->parent != TC_H_ROOT)
		return -EOPNOTSUPP;
    
    // 没有 multiqueue 不支持
	if (!netif_is_multiqueue(dev))
		return -EOPNOTSUPP;

	/* pre-allocate qdiscs, attachment can't fail */
    // 先分配
	priv->qdiscs = kcalloc(dev->num_tx_queues, sizeof(priv->qdiscs[0]),
			       GFP_KERNEL);
	if (!priv->qdiscs)
		return -ENOMEM;

	for (ntx = 0; ntx < dev->num_tx_queues; ntx++) {
        // 获取 netdev 的 NIC queue
		dev_queue = netdev_get_tx_queue(dev, ntx);
        // 创建 qdisc，它的 child 是 default qdisc
		qdisc = qdisc_create_dflt(dev_queue, get_default_qdisc_ops(dev, ntx),
					  TC_H_MAKE(TC_H_MAJ(sch->handle),
						    TC_H_MIN(ntx + 1)),
					  extack);
		if (!qdisc)
			return -ENOMEM;
		priv->qdiscs[ntx] = qdisc;
		qdisc->flags |= TCQ_F_ONETXQUEUE | TCQ_F_NOPARENT;
	}
    
    // TCQ_F_MQROOT 是 multiqueue + ROOT 的标志
	sch->flags |= TCQ_F_MQROOT;

	mq_offload(sch, TC_MQ_CREATE);
	return 0;
}
```

关于 class ops，其他都是一些增删查改的操作，其中 select queue 比较特别，可以看一下：

```c
static struct netdev_queue *mq_queue_get(struct Qdisc *sch, unsigned long cl)
{
	struct net_device *dev = qdisc_dev(sch);
	unsigned long ntx = cl - 1;

	if (ntx >= dev->num_tx_queues)
		return NULL;
	return netdev_get_tx_queue(dev, ntx);
}

static struct netdev_queue *mq_select_queue(struct Qdisc *sch,
					    struct tcmsg *tcm)
{
	return mq_queue_get(sch, TC_H_MIN(tcm->tcm_parent));
}
```

以上可以看到 它是根据 ` TC_H_MIN(tcm->tcm_parent)`，即 minor ID 来与 queue 一一对应的。

## other tips: select queue

看到这里，还是有疑问，发送 skb 的时候是怎么映射到 NIC queue 的呢？怎么知道走 tc mq 的哪个 slave 呢？

在 cilium 的 EDT 里面有 comments 说参考 `netdev_pick_tx` 这个函数，我们梳理调用路径可以知道

```
__dev_queue_xmit
	-> netdev_core_pick_tx
		-> netdev_pick_tx
```

在 netdev_core_pick_tx 中

```c
struct netdev_queue *netdev_core_pick_tx(struct net_device *dev,
					 struct sk_buff *skb,
					 struct net_device *sb_dev)
{
	int queue_index = 0;

#ifdef CONFIG_XPS
	u32 sender_cpu = skb->sender_cpu - 1;

	if (sender_cpu >= (u32)NR_CPUS)
		skb->sender_cpu = raw_smp_processor_id() + 1;
#endif

	if (dev->real_num_tx_queues != 1) {
		const struct net_device_ops *ops = dev->netdev_ops;

		if (ops->ndo_select_queue)
            // 优先根据设备配置的 net_device_ops select queue 的方法
			queue_index = ops->ndo_select_queue(dev, skb, sb_dev);
		else
            // 没有配置的话，调内部函数 netdev_pick_tx 来选 queue
			queue_index = netdev_pick_tx(dev, skb, sb_dev);

		queue_index = netdev_cap_txqueue(dev, queue_index);
	}

	skb_set_queue_mapping(skb, queue_index);
	return netdev_get_tx_queue(dev, queue_index);
}
```

每个网络 driver 可以配置其选 tx queue  的算法

```c
/*
* u16 (*ndo_select_queue)(struct net_device *dev, struct sk_buff *skb,
 *                         struct net_device *sb_dev);
 *	Called to decide which queue to use when device supports multiple
 *	transmit queues.
 */
struct net_device_ops {
    // ...
	u16			(*ndo_select_queue)(struct net_device *dev,
						    struct sk_buff *skb,
						    struct net_device *sb_dev);
    // ...
}
```

后面我们来分析一下 `netdev_pick_tx` 的实现：

```c

u16 netdev_pick_tx(struct net_device *dev, struct sk_buff *skb,
		     struct net_device *sb_dev)
{
	struct sock *sk = skb->sk;
	int queue_index = sk_tx_queue_get(sk);

	sb_dev = sb_dev ? : dev;

	if (queue_index < 0 || skb->ooo_okay ||
	    queue_index >= dev->real_num_tx_queues) {
		int new_index = get_xps_queue(dev, sb_dev, skb);

		if (new_index < 0)
			new_index = skb_tx_hash(dev, sb_dev, skb);

		if (queue_index != new_index && sk &&
		    sk_fullsock(sk) &&
		    rcu_access_pointer(sk->sk_dst_cache))
			sk_tx_queue_set(sk, new_index);

		queue_index = new_index;
	}

	return queue_index;
}
EXPORT_SYMBOL(netdev_pick_tx);
```

`sk_tx_queue_get` 分析 skb queue_mapping 是否已经设置，如果设置了就直接使用，跳过。否则设置为 -1。

否则会进入 if 选择 queue，queue_index 不合法或者 ooo_okay(out of order?) 会进行 xps 选 queue，首先调用 `get_xps_queue`，它会使用一个由用户配置的 TX queue 到 CPU 的映射，这称为 XPS（Transmit Packet Steering ，发送数据包控制）。

如果内核不支持 XPS，或者系统管理员未配置 XPS，或者配置的映射引用了无效队列， `get_xps_queue` 返回-1，则代码将继续调用 `skb_tx_hash`。

关于 XPS: [linux/Documentation/networking/scaling.txt at v3.13 · torvalds/linux · GitHub](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L364-L422)

XPS 也没有配置的话，则根据 `skb_tx_hash` 来计算。



