---
title: rcu(2) - other rcu flavor
date: 2025-02-19 15:07:35
tags:
- kernel
- rcu
---

translated from: https://docs.kernel.org/RCU/Design/Requirements/Requirements.html#other-rcu-flavors

## RCU-bh: bottom-half flavor

Soft-irq disable RCU 最初是为了防止 DoS 网络攻击。这种攻击通过对系统施加巨大网络流量，导致系统无法退出 softirq 的执行逻辑，从而导致 RCU 无法执行 context switch，也导致 grace period 无法结束，最终系统 hang 住。

RCU-bh 则通过在 Read-side API 禁止 softirq 执行（lockdep 也会记录这一行为）。具体来说， `rcu_read_lock_bh()` 和 `rcu_read_unlock_bh()` 会 disable 和 re-enable softirq , 如果有要启用 softirq handler 的情况，会被 deferred 知道 re-enable。

The [RCU-bh API](https://lwn.net/Articles/609973/#RCU Per-Flavor API Table) 包括 [`rcu_read_lock_bh()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_read_lock_bh), [`rcu_read_unlock_bh()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_read_unlock_bh), [`rcu_dereference_bh()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_dereference_bh), [`rcu_dereference_bh_check()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_dereference_bh_check), and [`rcu_read_lock_bh_held()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_read_lock_bh_held). 不过旧的 RCU-bh 更新的 APIs 现在被移出了, 被 [`synchronize_rcu()`](https://docs.kernel.org/core-api/kernel-api.html#c.synchronize_rcu), [`synchronize_rcu_expedited()`](https://docs.kernel.org/core-api/kernel-api.html#c.synchronize_rcu_expedited), [`call_rcu()`](https://docs.kernel.org/core-api/kernel-api.html#c.call_rcu), and [`rcu_barrier()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_barrier) 替代了。

## RCU-sched: sched flavor

在可抢占的 RCU 之前，等待 RCU grace period 结束会有副作用，需要等待之前所有 interrupt 和 NMI handlers 结束。但有些 RCU 实现不具备这种性质（等 interrupt 和 NMI 结束），因为代码中 RCU 读取端临界区之外的任何点都可能处于静止状态。因此有了 RCU-sched。在 kernel 编译的时候，有 `CONFIG_PREEMPTION=n`，RCU 和RCU-sched 都有相同的实现，`CONFIG_PREEMPION=y` 的时候，则会有不同的实现。

`CONFIG_PREEMPION=y`的kernel，会使用[`rcu_read_lock_sched()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_read_lock_sched) and [`rcu_read_unlock_sched()`](https://docs.kernel.org/core-api/kernel-api.html#c.rcu_read_unlock_sched) 来 disable 和 re-enable 抢占。当中断发生的时候，正好 rcu_read_unlcok_sched 结束，导致它看起来执行很慢。

## Sleepable RCU

十多年来，如果有人说“我需要在 RCU 读端临界区内进行阻塞”，这肯定表明这个人并不了解 RCU。毕竟，如果您总是在 RCU 读端临界区内进行阻塞，那么您可能可以使用更高开销的同步机制。然而，随着 Linux 内核 notifier 的出现，这种情况发生了变化，其 RCU 读端临界区几乎从不休眠，但有时也需要休眠。这导致了[可休眠 RCU](https://lwn.net/Articles/202847/)或 *SRCU*的引入。
