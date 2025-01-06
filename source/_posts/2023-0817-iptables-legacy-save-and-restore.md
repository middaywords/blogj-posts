---
title: iptables-legacy save and restore
date: 2023-08-17 22:33:44
tags:
- iptables
---

[TOC]

## iptables legacy save 和 restore 分析

写这篇东西的背景比较啰嗦。

公司的 gateway 节点用的 iptables-legacy 实现一些 Access control 之类的网络策略。iptables 数量很多，遍历会导致 si 比较高，CPU 较高，容易出问题，需要一些监控服务，监控 iptables 的变化来辅助诊断问题。

iptables 控制面相关的监控服务是我在公司做的第二个比较完整的工作。实现过程中，需要了解一些 iptables-save 和 iptables-restore 的相关实现，因此记录一下。

### iptables-save

iptables-save 会遍历所有的 table, 将每个 table 的所有 rule 打印出来。相关的 iptables rule 存储在内核的 iptables 模块中。用户态是通过 getsockopt 的系统调用来获取相关信息的。简要概括的话，包括两个步骤

1. GET_INFO：获取 iptables 模块中某个 table 的 metadata
2. GET_ENTRIES: 获取 iptables 模块中某个 table 的每一个条 entry

> 关于 entry 和 rule 的关系： 我理解是 entry 里面包括 xt_target， xt_match，match表示匹配条件，target 表示匹配后的具体操作，比如 DROP，ACCEPT之类的。 [1]

### iptables-restore

iptables-restore 会先将通过和 iptables-save 一样的方式，调用 getsockopt 来获取内核模块中的 iptables rules。然后在用户态进行改动，生成新的表，包含所有的 iptables 规则，之后再通过 setsockopt 系统调用，将表复制到内核中，将指定 table 进行替换

1. GET_INFO：获取 iptables 模块中某个 table 的 metadata
2. GET_ENTRIES: 获取 iptables 模块中某个 table 的每一个条 entry
3. SET_REPLACE: 将用户态修改后的表复制到内核态中，将表的指针通过原子操作修改，指向新的表。将 iptables counter 置为 0，并将现在的 counter 复制到 用户态。（Question： 为什么这里需要多一次 对 counter 的记录呢？对 counter 的修改为什么要拆成两次呢？个人猜想： 不能保持是因为规则发生了变化，之前的counter 对应的规则不一定存在了或者修改了。）
4. SET_ADD_COUNTERS: 在用户态计算了 counter 数值后，通过系统调用 setsockopt，使内核模块里的统计值原子加上 用户态提供的 counter 数值。

替换过程中的 counter 变化。（iptables -n 即可显示匹配的 pkts bytes，这个数据由 iptables counter 统计）

```c
static void counters_normal_map(STRUCT_COUNTERS_INFO *newcounters,
				STRUCT_REPLACE *repl, unsigned int idx,
				unsigned int mappos)
{
	/* Original read: X.
	 * Atomic read on replacement: X + Y.
	 * Currently in kernel: Z.
	 * Want in kernel: X + Y + Z.
	 * => Add in X + Y
	 * => Add in replacement read.
	 */
	newcounters->counters[idx] = repl->counters[mappos];
	DEBUGP_C("NORMAL_MAP => mappos %u \n", mappos);
}
```

### save 和 restore 过程中的锁分析

save 和 restore 调用 setsockopt 或者 getsockopt 进入 iptables 内核模块时，会进入 xt_table 模块中，其中每个 xt_table 在被操作时，都会加锁，这个锁是表级别的。

使用 save 和 restore 过程中，除了内核态，在用户态也会竞争用户态的一把锁，默认配置在 /run/xtables.lock。这把锁大概是对所有 iptables 相关操作的。

restore 的过程中，对数据面有影响，可以看到在如下代码中，会调用 bottom half disable，将下半部中断禁用，会导致网络协议栈暂停。此外，中间还包括一个原子替换和多核同步（IPI 中断）。

```c
struct xt_table_info *
xt_replace_table(struct xt_table *table,
	      unsigned int num_counters,
	      struct xt_table_info *newinfo,
	      int *error)
{
	struct xt_table_info *private;
	unsigned int cpu;
	int ret;

	ret = xt_jumpstack_alloc(newinfo);
	...

	/* Do the substitution. */
	local_bh_disable();
	private = table->private;

	/* Check inside lock: is the old number correct? */
	...

	newinfo->initial_entries = private->initial_entries;
	/*
	 * Ensure contents of newinfo are visible before assigning to
	 * private.
	 */
	smp_wmb();
	table->private = newinfo;

	/* make sure all cpus see new ->private value */
	smp_mb();

	/*
	 * Even though table entries have now been swapped, other CPU's
	 * may still be using the old entries...
	 */
	local_bh_enable();

	/* ... so wait for even xt_recseq on all cpus */
	...
}
EXPORT_SYMBOL_GPL(xt_replace_table);
```





## reference

[1] https://switch-router.gitee.io/blog/netfilter4/
