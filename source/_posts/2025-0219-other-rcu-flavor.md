---
title: rcu(2) - sleepable rcu
date: 2025-02-19 15:07:35
tags:
- kernel
- rcu
---

translated from: https://docs.kernel.org/RCU/Design/Requirements/Requirements.html#other-rcu-flavors

## Sleepable RCU

十多年来，如果有人说“我需要在 RCU 读端临界区内进行阻塞”，这肯定表明这个人并不了解 RCU。毕竟，如果您总是在 RCU 读端临界区内进行阻塞，那么您可能可以使用更高开销的同步机制。然而，随着 Linux 内核 notifier 的出现，这种情况发生了变化，其 RCU 读端临界区几乎从不休眠，但有时也需要休眠。这导致了[可休眠 RCU](https://lwn.net/Articles/202847/)或 *SRCU*的引入。
