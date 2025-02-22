---
title: rcu(1) - basics
date: 2025-02-19 15:03:20
tags:
- rcu
- kernel
---

这篇文章聚焦于 rcu 底层实现的一些基本概念，不会具体谈 rcu 是如何使用的。

关于 rcu 基本使用，可以参考 https://lwn.net/Articles/262464/, Paul E. McKenney 写的，rcu maintainer，写的很好。（尽管他写了很多文章，代码注释也写的很详细，但 rcu 还是挺难懂的。

另外，这篇文章关于源码的记录，是基于 kernel 6.8，图片不是，因为图片基本上都是盗的... -_-||

## 0. rcu usage

>  这一部分抄的 https://www.cnblogs.com/LoyenWang/p/12681494.html，这篇图文并茂，写的很好，大篇参考（侵删

`RCU`的基本思想是将更新`Update`操作分为两个部分：1）`Removal`移除；2）`Reclamation`回收。
直白点来理解就是，临界资源被多个读者读取，写者在拷贝副本修改后进行更新时，第一步需要先把旧的临界资源数据移除（修改指针指向），第二步需要把旧的数据进行回收（比如`kfree`）。

因此，从功能上分为以下三个基本的要素：`Reader/Updater/Reclaimer`，三者之间的交互如下图：

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200411183349989-1834656562.png)

1. Reader
   - 使用`rcu_read_lock`和`rcu_read_unlock`来界定读者的临界区，访问受`RCU`保护的数据时，需要始终在该临界区域内访问；
   - 在访问受保护的数据之前，需要使用`rcu_dereference`来获取`RCU-protected`指针；
   - 当使用不可抢占的`RCU`时，`rcu_read_lock/rcu_read_unlock`之间不能使用可以睡眠的代码；
2. Updater
   - 多个Updater更新数据时，需要使用互斥机制进行保护；
   - Updater使用`rcu_assign_pointer`来移除旧的指针指向，指向更新后的临界资源；
   - Updater使用`synchronize_rcu`或`call_rcu`来启动`Reclaimer`，对旧的临界资源进行回收，其中`synchronize_rcu`表示同步等待回收，`call_rcu`表示异步回收；
3. Reclaimer
   - Reclaimer回收的是旧的临界资源；
   - 为了确保没有读者正在访问要回收的临界资源，Reclaimer需要等待所有的读者退出临界区，这个等待的时间叫做宽限期（`Grace Period`）；

后面我们会从 rcu data structures, rcu 读接口，rcu 更新接口，rcu 宽限期处理等分析起源码实现。

## 1. rcu data structures

> 这一部分参考 https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/Data-Structures.html

RCU 出于所有意图和目的是一个大型状态机，其数据结构以允许 RCU 读者极快地执行的方式维护状态，同时还以高效且极具可扩展性的方式处理更新程序请求的 RCU 宽限期 .
RCU updaters 的效率和可扩展性主要由组合树提供，如下所示：

![TreeLevel.svg](https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/TreeLevel.svg)

此图显示了一个包含“rcu_node”结构树的封闭“rcu_state”结构。 ``rcu_node`` 树的每个叶节点**最多有 16 个** ``rcu_data`` 结构与之关联，因此有 ``NR_CPUS`` 数量的 ``rcu_data`` 结构，
 如果需要，此结构会在启动时进行调整，以处理 ``nr_cpu_ids`` 远小于 ``NR_CPUs`` 的常见情况。 

此组合树的目的是允许高效且可扩展地处理每个 CPU 事件，例如静态、dyntick-idle 转换和 CPU 热插拔操作。
静态由每个 CPU 的“rcu_data”结构记录，和其他事件由叶级 ``rcu_node`` 结构记录。
所有这些事件都在树的每一层进行组合，直到最后在树的根“rcu_node”结构中完成宽限期。
一旦每个 CPU（或者，在 ``CONFIG_PREEMPT_RCU`` 的情况下，任务）已经通过静止状态，就可以在根节点完成宽限期。 一旦宽限期结束，该事实的记录就会沿着树向下传播。

如果您的系统有超过 1,024 个 CPU（或在 32 位系统上超过 512 个 CPU），那么 RCU 会自动向树中添加更多层级。 例如，如果你足够疯狂地构建一个具有 65,536 个 CPU 的 64 位系统，RCU 将配置
``rcu_node`` 树如上图所示。

这种多级组合树使我们能够获得分区的大部分性能和可伸缩性优势，即使 RCU 宽限期检测本质上是一个全局操作。 这里的技巧是，只有最后一个 CPU 将静止状态报告给给定的 ``rcu_node`` 结构需要前
进到树的下一层的 ``rcu_node`` 结构。 这意味着在叶级 ``rcu_node`` 结构中，16 个访问中只有一个会在树上进行。 对于内部的 ``rcu_node`` 结构，情况更加极端：64 次访问中只有一次会在树上进
行。 因为绝大多数 CPU 没有在树中向上移动，所以锁争用在树中大致保持不变。 无论系统中有多少个 CPU，每个宽限期最多 64 个静态报告将一直进行到根 ``rcu_node`` 结构，从而确保该根 ``rcu_node``
结构上的锁争用仍然处于可接受的低水平。

RCU updaters 通过注册 RCU 回调来等待正常的宽限期，可以直接通过 ``call_rcu()`` 或间接通过 ``synchronize_rcu()`` 或其朋友函数。 RCU 回调由 ``rcu_head`` 结构表示，它们在等待宽限期结束时在
``rcu_data`` 结构上排队，如下图所示：

![BigTreePreemptRCUBHdyntickCB.svg](https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/BigTreePreemptRCUBHdyntickCB.svg)

此图显示了 ``TREE_RCU`` 和 ``PREEMPT_RCU`` 的主要数据结构之间的关系。 较小的数据结构将与使用它们的算法一起引入。

注意上图中的每个数据结构都有自己的同步：

* 每个 ``rcu_state`` 结构都有一个锁和一个互斥量，一些字段由相应的根 ``rcu_node`` 结构的锁保护。
* 每个 ``rcu_node`` 结构都有一个自旋锁。
* ``rcu_data`` 中的字段是相应 CPU 私有的，尽管有一些字段可以被其他 CPU 读取和写入。 

重要的是要注意，不同的数据结构在任何给定时间对 RCU 的状态可能有非常不同的想法。 举一个例子，对给定 RCU 宽限期开始或结束的意识通过数据结构缓慢传播。 这种缓慢的传播对于 RCU 具有良好
的读取端性能是绝对必要的。 如果您觉得这种割裂的实现方式很陌生，一个有用的技巧是将这些数据结构的每个实例都视为不同的人，每个人对现实的看法通常略有不同。

这些数据结构的一般作用如下：

* ``rcu_state``：该结构形成``rcu_node``和``rcu_data``结构之间的互连，跟踪宽限期，作为CPU热插拔事件孤立的回调的短期存储库，维护``rcu_barrier()`` 状态，跟踪加速宽限期状态，并在宽限
  期延长太长时维护用于强制静态状态的状态，

* ``rcu_node``：这个结构形成了组合树，它将静止状态信息从叶子传播到根，并且还将宽限期信息从根传播到叶子。它提供宽限期状态的本地副本，以允许以同步方式访问此信息，而不会受到全局锁定强加的
  可伸缩性限制。 在 ``CONFIG_PREEMPT_RCU`` 内核中，它管理在当前 RCU 读端临界区中阻塞的任务列表。 在 ``CONFIG_PREEMPT_RCU`` 和 ``CONFIG_RCU_BOOST`` 中，它管理每 ``rcu_node`` 的优先级提
  升内核线程（kthreads）和状态。 最后，它记录 CPU 热插拔状态以确定在给定的宽限期内应忽略哪些 CPU。

* ``rcu_data``：这个每 CPU 的结构是静态检测和 RCU 回调排队的重点。 它还跟踪它与相应的叶``rcu_node``结构的关系，以允许更有效地将静态状态传播到``rcu_node``组合树上。 与 rcu_node 结构
  一样，它提供了宽限期信息的本地副本，以允许从相应的 CPU 中访问此信息，而不需要任何同步操作。 最后，该结构记录了相应 CPU 过去的 dyntick-idle 状态并跟踪统计信息。

* ``rcu_head``：这个结构代表RCU回调，并且是RCU用户分配和管理的唯一结构。 ``rcu_head`` 结构通常嵌入在受 RCU 保护的数据结构中。

### 1.1 rcu_state 结构

``rcu_state`` 结构是表示系统中 RCU 状态的基本结构。 这个结构形成了 ``rcu_node`` 和 ``rcu_data`` 结构之间的互连，跟踪宽限期，包含用于与 CPU 热插拔事件同步的锁，并维护用于在宽限期太长时强制进入静止状态的状态。

* 与 rcu_node 和 rcu_data 结构的关系

``rcu_state`` 结构的这一部分声明如下：

```c
struct rcu_state {
	struct rcu_node node[NUM_RCU_NODES];	/* Hierarchy. */
	struct rcu_node *level[RCU_NUM_LVLS + 1];
  /* Hierarchy levels (+1 to */
	/*  shut bogus gcc warning) */
```

* 宽限期跟踪

``rcu_state`` 结构的这一部分声明如下：

```c
struct rcu_state {
	unsigned long gp_seq ____cacheline_internodealigned_in_smp;
	/* Grace-period sequence #. */

}
```

RCU 宽限期是被编号的，``->gp_seq`` 字段包含当前宽限期序列号。 低两位是当前宽限期的状态，可以为 0 表示尚未开始，也可以为 1 表示进行中。换句话说，如果 ``->gp_seq`` 的低两位为零，则 RCU 空闲。
底部两位中的任何其他值都表示有东西坏了。 该字段受根 ``rcu_node`` 结构的 ``->lock`` 字段保护。

``rcu_node`` 和 ``rcu_data`` 结构中也有 ``->gp_seq`` 字段。 ``rcu_state`` 结构中的字段表示最新值，并且比较其他结构中的字段，以便以分布式方式检测宽限期的开始和结束。
值从 ``rcu_state`` 流向 ``rcu_node``（从根到叶的树）到 ``rcu_data``。

``rcu_state`` 结构的这一部分声明如下：

```c
struct rcu_state {
	unsigned long gp_max;/* Maximum GP duration in jiffies*/
	const char *name;/* Name of structure. */
	char abbr;/* Abbreviated name. */
```

``->gp_max`` 字段以 jiffies 为单位跟踪最长宽限期的持续时间。 它受根 ``rcu_node`` 的 ``->lock`` 保护。

``->name`` 和 ``->abbr`` 字段区分抢占式 RCU（“rcu_preempt”和“p”）和非抢占式 RCU（“rcu_sched”和“s”）。 这些字段用于诊断和跟踪目的。



### 1.2 rcu_node 结构

``rcu_node`` 结构形成了组合树，它将静止状态信息从叶子传播到根，并将宽限期信息从根向下传播到叶子。 它们提供宽限期状态的本地副本，以允许以同步方式访问此信息，而不会受到全局锁定强加的可伸缩性限制。 在 ``CONFIG_PREEMPT_RCU`` 内核中，它们管理在当前 RCU 读端临界区中阻塞的任务列表。在 ``CONFIG_PREEMPT_RCU`` 和 ``CONFIG_RCU_BOOST`` 中，它们管理每个 ``rcu_node`` 优先级提升内核线程（kthreads）和状态。 最后，他们记录 CPU 热插拔状态以确定在给定的宽限期内应忽略哪些 CPU。

* 连接到组合树

```c
struct rcu_node {
	struct rcu_node *parent;
	unsigned long grpmask;	/* Mask to apply to parent qsmask. */
				/*  Only one bit will be set in this mask. */
	int	grplo;		/* lowest-numbered CPU here. */
	int	grphi;		/* highest-numbered CPU here. */
	u8	grpnum;		/* group number for next level up. */
	u8	level;		/* root is at level 0. */
}
```

``->parent`` 指针引用树中上一层的``rcu_node``，对于根``rcu_node`` 为``NULL``。 RCU 实现大量使用此字段将静止状态推到树上。 ``->level`` 字段给出了树中的级别，根为零级，
其子级为一级，依此类推。 ``->grpnum`` 字段给出了该节点在其父节点的子节点中的位置，因此该数字在 32 位系统上可以介于 0 到 31 之间，在 64 位系统上可以介于 0 到 63 之间。
``->level`` 和 ``->grpnum`` 字段仅在初始化和tracing期间使用。 ``->grpmask`` 字段是 ``->grpnum`` 的对应位掩码，因此总是只有一位设置。此掩码用于清除其父级位掩码中与此
``rcu_node`` 结构对应的位，稍后将对此进行描述。 最后，``->grplo`` 和``->grphi`` 字段分别包含此 ``rcu_node`` 结构服务的编号最低和最高的 CPU。

* 同步

``rcu_node`` 结构的这个字段声明如下：

```c
struct rcu_node {
	raw_spinlock_t __private lock;
  /* Root rcu_node's lock protects */
	/*  some rcu_state fields as well as */
	/*  following. */
```

除非另有说明，否则此字段用于保护此结构中的其余字段。 也就是说，出于跟踪目的，无需锁定即可访问此结构中的所有字段。

* 宽限期跟踪

```c
struct rcu_node {
	unsigned long gp_seq;	/* Track rsp->gp_seq. */
	unsigned long gp_seq_needed; /* Track furthest future GP request. */
```

``rcu_node`` 结构的``->gp_seq`` 字段是``rcu_state`` 结构中同名字段的对应项。 他们每个人都可能落后于他们的 ``rcu_state`` 一步。 如果给定 ``rcu_node`` 结构的 ``->gp_seq`` 字段的底部两位为零，则此 ``rcu_node`` 结构认为 RCU 空闲。

每个“rcu_node”结构的“>gp_seq”字段在每个宽限期的开始和结束时更新。

``->gp_seq_needed`` 字段记录相应 ``rcu_node`` 结构看到的最远的未来宽限期请求。 当 ``->gp_seq`` 字段的值等于或超过 ``->gp_seq_needed`` 字段的值时，请求被认为已完成。

* 静止状态的跟踪

这些字段管理静态在组合树上的传播。``rcu_node`` 结构的这一部分具有如下字段：

```c
	unsigned long qsmask;	/* CPUs or groups that need to switch in */
  /*  order for current grace period to proceed.*/
  /*  In leaf rcu_node, each bit corresponds to */
  /*  an rcu_data structure, otherwise, each */
  /*  bit corresponds to a child rcu_node */
  /*  structure. */
	unsigned long qsmaskinit;
  /* Per-GP initial value for qsmask. */
  /*  Initialized from ->qsmaskinitnext at the */
  /*  beginning of each grace period. */
	unsigned long expmask;	/* CPUs or groups that need to check in */
  /*  to allow the current expedited GP */
  /*  to complete. */
	unsigned long expmaskinit;
  /* Per-GP initial values for expmask. */
  /*  Initialized from ->expmaskinitnext at the */
  /*  beginning of each expedited GP. */
```

``->qsmask`` 字段跟踪此 ``rcu_node`` 结构的哪些子结构仍需要报告当前正常宽限期的静止状态。 这样的子结构在其相应位中的值为 1。 请注意，叶 ``rcu_node`` 结构应该被视为具有 ``rcu_data``结构作为它们的子结构。 类似地，``->expmask`` 字段跟踪此 ``rcu_node`` 结构的哪些子结构仍需要报告当前加速宽限期的静止状态。 加速宽限期与正常宽限期具有相同的概念属性，但加速实施接受极端的 CPU 开销以获得更低的宽限期延迟，例如，消耗几十微秒的 CPU 时间来减少持续时间从几毫秒到几十微秒的宽限期。 ``->qsmaskinit`` 字段跟踪这个 ``rcu_node`` 结构的哪个子结构覆盖了至少一个在线 CPU。 此掩码用于初始化``->qsmask``，``->expmaskinit`` 用于初始化``->expmask`` 以及正常和加速宽限期的开始。

* 阻塞任务管理

```c

	struct list_head blkd_tasks;
  /* Tasks blocked in RCU read-side critical */
  /*  section.  Tasks are placed at the head */
  /*  of this list and age towards the tail. */
	struct list_head *gp_tasks;
  /* Pointer to the first task blocking the */
  /*  current grace period, or NULL if there */
  /*  is no such task. */
	struct list_head *exp_tasks;
  /* Pointer to the first task blocking the */
  /*  current expedited grace period, or NULL */
  /*  if there is no such task.  If there */
  /*  is no current expedited grace period, */
  /*  then there can cannot be any such task. */
```

``->blkd_tasks`` 字段是阻塞和抢占任务列表的列表头。 当任务在 RCU 读端临界区内进行上下文切换时，它们的 ``task_struct`` 结构被排入队列（通过 ``task_struct`` 的 ``->rcu_node_entry`` 字段）到 ``-> blkd_tasks`` 中，它是叶``rcu_node``结构的列表，对应于执行传出上下文切换的CPU。

 当这些任务稍后退出它们的 RCU 读端临界区时，它们将自己从列表中移除。 因此这个列表是时间倒序的，所以如果其中一个任务阻塞了当前的宽限期，那么所有后续任务也必须阻塞同一个宽限期。 因此，指向此列表的单个指针足以跟踪所有阻塞给定宽限期的任务。 

对于正常的宽限期，该指针存储在 ``->gp_tasks`` 中，对于加速的宽限期存储在 ``->exp_tasks`` 中。 如果没有进行中的宽限期或者没有阻止宽限期完成的阻塞任务，最后两个字段为“NULL”。 如果这两个指针中的任何一个正在引用一个从``->blkd_tasks``列表中删除自身的任务，那么该任务必须将指针推进到列表中的下一个任务，或者将指针设置为``NULL``如果列表中没有后续任务。

例如，假设任务 T1、T2 和 T3 都 hard-affinitied 到系统中编号最大的 CPU 上。 然后，如果任务 T1 阻塞在 RCU 读端临界区，然后加速宽限期开始，然后任务 T2 阻塞在 RCU 读端临界区，然后正常宽限期开
始，最后任务 3 阻塞在 RCU 读端临界区，那么最后一个叶子 ``rcu_node`` 结构的阻塞任务列表的状态将如下所示：

![blkd_task.svg](https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/blkd_task.svg)

任务 T1 阻塞了两个宽限期，任务 T2 仅阻塞了正常宽限期，任务 T3 没有阻塞任何宽限期。 请注意，这些任务不会在恢复执行后立即从该列表中删除。 它们将保留在列表中，直到它们执行最外层的
``rcu_read_unlock()`` 结束它们的 RCU 读端临界区。

``->wait_blkd_tasks`` 字段指示当前宽限期是否正在等待阻塞的任务。

### 1.3 rcu_segcblist 结构

``rcu_segcblist`` 结构维护一个分段的回调列表

```c
struct rcu_segcblist {
	struct rcu_head *head;
	struct rcu_head **tails[RCU_CBLIST_NSEGS];
	unsigned long gp_seq[RCU_CBLIST_NSEGS];
#ifdef CONFIG_RCU_NOCB_CPU
	atomic_long_t len;
#else
	long len;
#endif
	long seglen[RCU_CBLIST_NSEGS];
	u8 flags;
};
```

**1.3.1 关于 list 结构**：

分段如下：

*  ``RCU_DONE_TAIL``：宽限期已过的回调。 这些回调已准备好被调用。
* ``RCU_WAIT_TAIL``：正在等待当前宽限期的回调。 请注意，不同的 CPU 可能对哪个宽限期是当前有不同的想法，见 ``->gp_seq`` 字段。
* ``RCU_NEXT_READY_TAIL``：等待下一个宽限期开始的回调。
* ``RCU_NEXT_TAIL``：尚未与宽限期关联的回调。  

``->head`` 指针引用第一个回调，或者如果列表不包含回调（这*不* 等于空）则为 ``NULL``。 ``->tails[]`` 数组的每个元素引用列表相应段中最后一个回调的``->next`` 指针，或者如果该段是列
表的``->head`` 指针 并且之前的所有段都是空的。 如果相应的段为空但之前的某个段不为空，则数组元素与其前身相同。 较旧的回调更靠近列表的头部，新的回调添加在尾部。 ``->head`` 指针、
``->tails[]`` 数组和回调之间的关系如下图所示：

![nxtlist.svg](https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/nxtlist.svg)

在此图中，

``->head`` 指针引用列表中的第一个 RCU 回调。 

``->tails[RCU_DONE_TAIL]`` 数组元素引用``->head`` 指针本身，表明没有任何回调准备好调用。

 ``->tails[RCU_WAIT_TAIL]`` 数组元素引用回调 CB 2 的 ``->next`` 指针，这表明 CB 1 和 CB 2 都在等待当前宽限期，给出或接受可能的分歧 哪个宽限期是当前的宽限期。

 ``->tails[RCU_NEXT_READY_TAIL]`` 数组元素引用与``->tails[RCU_WAIT_TAIL]`` 相同的 RCU 回调，这表明没有回调等待下一个 RCU 宽限期。

 ``->tails[RCU_NEXT_TAIL]`` 数组元素引用 CB 4 的``->next`` 指针，表示所有剩余的 RCU 回调尚未被分配 RCU 宽限期。 请注意，
``->tails[RCU_NEXT_TAIL]`` 数组元素始终引用最后一个 RCU 回调的``->next`` 指针，除非回调列表为空，在这种情况下它引用``->head`` 指针 .

``->tails[RCU_NEXT_TAIL]`` 数组元素还有一个重要的特殊情况：当此列表*禁用*时，它可以是 ``NULL``。 当相应的 CPU 处于离线状态或当相应的 CPU 的回调被卸载到 kthread 时，列表将被禁用，这两种情
况都在别处描述。

随着宽限期的推进，CPU 将它们的回调``RCU_NEXT_TAI``推进到``RCU_NEXT_READY_TAIL``到``RCU_WAIT_TAIL``到``RCU_DONE_TAIL``列表段。

1.3.2 gp_seq 

``->gp_seq[]`` 数组记录了与列表段对应的宽限期编号。 这就是允许不同的 CPU 对哪个是当前宽限期有不同的想法，同时仍然避免过早调用它们的回调。 特别是，这允许长时间空闲的 CPU 确定它们的哪些回调
已准备好在重新唤醒后调用。

1.3.3 其他

``->len`` 计数``->head`` 中回调的数量，而``->len_lazy`` 包含已知仅释放内存的那些回调的数量，其调用因此可以安全地推迟。

``->len`` 字段决定是否存在与此 ``rcu_segcblist`` 结构关联的回调，*不是* ``->head`` 指针。 这样做的原因是所有准备好调用的回调（即那些在 ``RCU_DONE_TAIL`` 段中的回调）在回调调用时间（``rcu_do_batch``）被一次性全部提取出来，因此如果 `rcu_segcblist` 中没有未完成的回调，``->head`` 可以设置为 NULL。 如果 callback 调用必须被推迟，（例如，因为一个高优先级进程刚刚在这个 CPU上醒来），那么剩余的回调将被放回 ``RCU_DONE_TAIL`` 段并且 ``->head`` 再次指向段的开始。 简而言之，head 字段可以短暂地为“NULL”，即使 CPU 一直存在回调。 因此，测试 ``->head`` 指针是否为 ``NULL``是不合适的。

相反，``->len`` 和 ``->len_lazy`` 计数仅在调用相应的回调后才进行调整。 这意味着只有当 rcu_segcblist 结构确实没有回调时，->len 计数才为零。 当然，``->len`` 计数的 off-CPU 采样需要小心使用适
当的同步，例如内存屏障。 这种同步可能有点微妙，特别是在 ``rcu_barrier()`` 的情况下。



### 1.4 rcu_data 结构

``rcu_data`` 维护 RCU 子系统的每个 CPU 状态。 除非另有说明，否则只能从相应的 CPU（和tracing）访问此结构中的字段。 此结构是静态检测和 RCU 回调排队的重点。 它还跟踪它与相应的叶``rcu_node``结构的关系，以允许更有效地将静态状态传播到``rcu_node``组合树上。 与 rcu_node 结构一样，它提供了宽限期信息的本地副本，以允许从相应的 CPU 中免费同步访问此信息。 最后，该结构记录了相应 CPU 过去的 dyntick-idle 状态并跟踪统计信息。

``rcu_data`` 结构的字段将在以下部分中单独和成组讨论。

1.4.1 连接到其他数据结构

``rcu_data`` 结构的这一部分声明如下：

```c
struct rcu_data {
  int cpu;
  struct rcu_node *mynode;
  unsigned long grpmask;
  bool beenonline;
}
```

``->cpu`` 字段是相应 CPU 的编号，``->mynode`` 字段引用相应的 ``rcu_node`` 结构。 ``->mynode`` 用于在组合树上传播静止状态。 这两个字段是常量，因此不需要同步。

``->grpmask`` 字段表示``->mynode->qsmask`` 中与此``rcu_data`` 结构对应的位，在传播静止状态时也会使用。 ``->beenonline`` 标志在相应的 CPU 上线时设置，这意味着 debugfs 跟踪不需要转储任何未设置此标志的 ``rcu_data`` 结构。

1.4.2 静态和宽限期跟踪

``rcu_data`` 结构的这一部分声明如下：

```c
struct rcu_data {
  unsigned long gp_seq;
  unsigned long gp_seq_needed;
  bool cpu_no_qs;
  bool core_needs_qs;
  bool gpwrap;
}
```

``->gp_seq`` 字段与``rcu_state`` 和``rcu_node`` 结构中的同名字段对应。 ``->gp_seq_needed`` 字段是 rcu_node 结构中同名字段的对应部分。 它们可能每个都落后于它们的 ``rcu_node`` 对应物，但在
``CONFIG_NO_HZ_IDLE`` 和 ``CONFIG_NO_HZ_FULL`` 内核可以任意落后于 dyntick-idle 模式下的 CPU（但一旦退出 dyntick-idle 模式，这些计数器会赶上）。 如果给定的 ``rcu_data`` 结构的 ``->gp_seq``
的低两位为零，那么这个 ``rcu_data`` 结构认为 RCU 是空闲的。

1.4.3  RCU 回调处理

在没有 CPU 热插拔事件的情况下，RCU 回调由注册它们的同一个 CPU 调用。 这完全是一种缓存位置优化：回调可以并且确实在注册它们的 CPU 之外的 CPU 上被调用。 毕竟，如果注册给定回调的 CPU 在回调可以
被调用之前已经离线，那么真的没有其他选择。

``rcu_data`` 结构的这一部分声明如下：

```c
struct rcu_data {
  struct rcu_segcblist cblist;
  long qlen_last_fqs_check;
  unsigned long n_cbs_invoked;
  unsigned long n_nocbs_invoked;
  unsigned long n_cbs_orphaned;
  unsigned long n_cbs_adopted;
  unsigned long n_force_qs_snap;
  long blimit;
}
```

``->cblist`` 结构是前面描述的分段回调列表。 只要 CPU 注意到另一个 RCU 宽限期已经完成，它就会在其 ``rcu_data`` 结构中推进回调。 CPU 通过注意到其`rcu_data`结构的`->gp_seq`字段的值与其叶`rcu_node`结构的值不同来检测 RCU 宽限期的完成。回想一下，每个 ``rcu_node`` 结构的 ``->gp_seq`` 字段在每个宽限期的开始和结束时更新。

当回调列表变得过长时，``->qlen_last_fqs_check`` 和``->n_force_qs_snap`` 协调来自``call_rcu()`` 和其朋友函数的静态强制。

``->n_cbs_invoked``、``->n_cbs_orphaned`` 和 ``->n_cbs_adopted`` 字段计算调用的回调数，当此 CPU 离线时发送到其他 CPU，并在其他 CPU 离线时从其他 CPU 接收。``->n_nocbs_invoked``在 CPU 的回调被卸载到 kthread 时使用。

最后，``->blimit`` 计数器是在给定时间可以调用的 RCU 回调的最大数量。



1.4.4 Dyntick-Idle 处理

``rcu_data`` 结构的这一部分声明如下：

```c
struct rcu_data {
	int dynticks_snap;
	unsigned long dynticks_fqs;
}
```

``->dynticks_snap`` 字段用于在强制静止状态时拍摄相应 CPU 的 dyntick-idle 状态的快照，因此可以从其他 CPU 访问。 最后，``->dynticks_fqs`` 字段用于统计此CPU 被确定为dyntick-idle 状态的次数，用于跟踪和调试。

rcu_data 结构的这一部分声明如下：

```c
struct rcu_data {
  long dynticks_nesting;
  long dynticks_nmi_nesting;
  atomic_t dynticks;
  bool rcu_need_heavy_qs;
  bool rcu_urgent_qs;
}
```

rcu_data 结构中的这些字段维护相应 CPU 的 per-CPU dyntick-idle 状态。 除非另有说明，否则只能从相应的 CPU（和tracing）访问这些字段。

``->dynticks_nesting`` 字段计算进程执行的嵌套深度，因此在正常情况下该计数器的值为零或一。 NMI、irq 和跟踪器由`->dynticks_nmi_nesting`字段计算。 由于无法屏蔽 NMI，因此必须使用 Andy Lutomirski 提供的算法仔细更改此变量。 idle 的初始转换加 1，嵌套转换加 2，因此嵌套级别 5 由 ``->dynticks_nmi_nesting`` 值 9 表示。 因此，这个计数器可以被认为是计算除了进程级转换之外，这个 CPU 不能进入 dyntick-idle 模式的原因的数量。

然而，事实证明，当在非空闲内核上下文中运行时，Linux 内核完全能够进入永不退出的中断处理程序，反之亦然。 因此，每当 ``->dynticks_nesting`` 字段从零递增时，``->dynticks_nmi_nesting`` 字段被设置为一个大的正数，每当 ``->dynticks_nesting`` 字段减少到 零，``->dynticks_nmi_nesting`` 字段设置为零。 假设错误嵌套中断的数量不足以使计数器溢出，每次相应的 CPU 从进程上下文进入空闲循环时，这种方法都会纠正 ``->dynticks_nmi_nesting`` 字段。

``->dynticks`` 字段计算相应的 CPU 进出 dyntick-idle 模式或用户模式的转换次数，因此当 CPU 处于 dyntick-idle 模式或用户模式时，该计数器的值为偶数，否则为奇数 . 用户模式自适应滴答支持需要计算进/出用户模式的转换（参见 timers/NO_HZ.txt）。

``->rcu_need_heavy_qs`` 字段用于记录 RCU 核心代码真的很想从相应的 CPU 看到一个静止状态，以至于它愿意调用重量级的 dyntick-counter 操作 . 此标志由 RCU 的上下文切换和 ``cond_resched()`` 代码检查，它们提供暂时的空闲逗留作为响应。

最后，``->rcu_urgent_qs`` 字段用于记录 RCU 核心代码真的希望从相应的 CPU 看到静止状态这一事实，其他各种字段表明 RCU 多么想要这种静止状态。 此标志由 RCU 的上下文切换路径
（``rcu_note_context_switch``）和 cond_resched 代码检查。

### 1.5 rcu_head 结构

每个 ``rcu_head`` 结构代表一个 RCU 回调。 这些结构通常嵌入在受 RCU 保护的数据结构中，其算法使用异步宽限期。 相反，当使用阻塞等待 RCU 宽限期的算法时，RCU 用户不需要提供“rcu_head”结构。

``rcu_head`` 结构具有如下字段：

```c
struct rcu_head {
  // ...
  struct rcu_head *next;
  void (*func)(struct rcu_head *head);
  // ...
}
```

``->next`` 字段用于将 ``rcu_data`` 结构中的列表中的 ``rcu_head`` 结构链接在一起。 ``->func`` 字段是一个指向函数的指针，当回调准备好被调用时，这个函数被传递一个指向 ``rcu_head`` 结构的指针。 但是，``kfree_rcu()`` 使用``->func`` 字段来记录``rcu_head`` 结构在封闭的受 RCU 保护的数据结构中的偏移量。

这两个字段都由 RCU 在内部使用。 从 RCU 用户的角度来看，这个结构是一个不透明的“cookie”。

### 1.6 ``task_struct`` 结构中的 RCU 特定字段

``CONFIG_PREEMPT_RCU`` 实现在 ``task_struct`` 结构中使用了一些额外的字段：

```c
#ifdef CONFIG_PREEMPT_RCU
  int rcu_read_lock_nesting;
  union rcu_special rcu_read_unlock_special;
  struct list_head rcu_node_entry;
  struct rcu_node *rcu_blocked_node;
#endif /* #ifdef CONFIG_PREEMPT_RCU */
#ifdef CONFIG_TASKS_RCU
  unsigned long rcu_tasks_nvcsw;
  bool rcu_tasks_holdout;
  struct list_head rcu_tasks_holdout_list;
  int rcu_tasks_idle_cpu;
#endif /* #ifdef CONFIG_TASKS_RCU */
```

``->rcu_read_lock_nesting`` 字段记录了 RCU 读端临界区的嵌套级别，

* ``->rcu_read_unlock_special`` 字段是一个位掩码，记录了需要 ``rcu_read_unlock()`` 做额外操作的特殊条件。
* ``->rcu_node_entry`` 字段用于形成在可抢占 RCU 读端临界区内阻塞的任务列表，
* ``->rcu_blocked_node`` 字段引用 ``rcu_node`` 结构，该任务为其列表的成员，或者如果它没有被阻塞在可抢占的 RCU 读端临界区内，则为 ``NULL``。
* ``->rcu_tasks_nvcsw`` 字段跟踪该任务在当前任务-RCU 宽限期开始时经历的自愿上下文切换次数，

* ``->rcu_tasks_holdout`` 如果当前 task-RCU 宽限期正在等待此任务则设置，
* ``->rcu_tasks_holdout_list`` 是将此任务排入 holdout 列表的列表元素
* ``->rcu_tasks_idle_cpu`` 跟踪此空闲任务正在运行的 CPU，但前提是该任务当前正在运行 ，也就是说，CPU 当前是否处于空闲状态。

### 1.7 访问函数

以下清单显示了``rcu_get_root()``、``rcu_for_each_node_breadth_first`` 和``rcu_for_each_leaf_node()`` 函数和宏：

```c
static struct rcu_node *rcu_get_root(struct rcu_state *rsp)
{
	return &rsp->node[0];
}

#define rcu_for_each_node_breadth_first(rsp, rnp) \
for ((rnp) = &(rsp)->node[0]; \
(rnp) < &(rsp)->node[NUM_RCU_NODES]; (rnp)++)

#define rcu_for_each_leaf_node(rsp, rnp) \
for ((rnp) = (rsp)->level[NUM_RCU_LVLS - 1]; \
(rnp) < &(rsp)->node[NUM_RCU_NODES]; (rnp)++)
```

``rcu_get_root()`` 只是返回指向指定``rcu_state`` 结构的``->node[]`` 数组的第一个元素的指针，这是根``rcu_node`` 结构。

如前所述，``rcu_for_each_node_breadth_first()`` 宏利用``rcu_state`` 结构的``->node[]`` 数组中的``rcu_node`` 结构的布局，执行广度优先 遍历只需按顺序遍历数组即可。 类似地，``rcu_for_each_leaf_node()`` 宏只遍历数组的最后一部分，因此只遍历叶``rcu_node`` 结构。



## 2. rcu 读接口

rcu 读接口主要是 `rcu_read_lock()` 和 `rcu_read_unlock()`。

我们先看 `rcu_read_lock()`

### 2.1 rcu_read_lock()

```c
static __always_inline void rcu_read_lock(void)
{
	__rcu_read_lock();
	__acquire(RCU);
	rcu_lock_acquire(&rcu_lock_map);
	RCU_LOCKDEP_WARN(!rcu_is_watching(),
			 "rcu_read_lock() used illegally while idle");
}

/*
 * Preemptible RCU implementation for rcu_read_lock().
 * Just increment ->rcu_read_lock_nesting, shared state will be updated
 * if we block.
 */
// kernel/rcu/tree_plugin.h
void __rcu_read_lock(void)
{
	rcu_preempt_read_enter();
	if (IS_ENABLED(CONFIG_PROVE_LOCKING))
		WARN_ON_ONCE(rcu_preempt_depth() > RCU_NEST_PMAX);
	if (IS_ENABLED(CONFIG_RCU_STRICT_GRACE_PERIOD) && rcu_state.gp_kthread)
		WRITE_ONCE(current->rcu_read_unlock_special.b.need_qs, true);
	barrier();  /* critical section after entry code. */
}
EXPORT_SYMBOL_GPL(__rcu_read_lock);

// include/linux/rcupdate.h
static inline void __rcu_read_lock(void)
{
	preempt_disable();
}
```

这段代码实现了一个可抢占的RCU（Read-Copy Update）机制，主要是为了在多线程环境下进行读操作时保证数据的一致性。

* `__rcu_read_lock`有两种 case
  * Preemptible RCU
    * 它首先调用 `rcu_preempt_read_enter` 函数来增加当前线程的 **RCU读锁嵌套计数**。然后，它根据配置选项检查一些条件
    * `IS_ENABLED(CONFIG_PROVE_LOCKING)`：如果启用了锁验证，则检查RCU抢占深度。
    * `IS_ENABLED(CONFIG_RCU_STRICT_GRACE_PERIOD)`：如果启用了严格的宽限期，则设置相应标志。
    * 最后，它调用 `barrier`函数来确保内存屏障。
  * Non-preemptible RCU
    * 调用 preempt_disable 关闭抢占

* 调用 `__acquire()`，相关说明： [内核工具 – Sparse 简介 ](https://www.cnblogs.com/hellokitty2/p/12548422.html)
* `rcu_lock_acquire(&rcu_lock_map);`：这是启用 lockdep 的时候会用到。



### 2.2 rcu_read_unlock()

和 `rcu_read_lock()` 类似，只是顺序反了。

```c

static inline void rcu_read_unlock(void)
{
	RCU_LOCKDEP_WARN(!rcu_is_watching(),
			 "rcu_read_unlock() used illegally while idle");
	__release(RCU);
	__rcu_read_unlock();
	rcu_lock_release(&rcu_lock_map); /* Keep acq info for rls diags. */
}

// include/linux/rcupdate.h
static inline void __rcu_read_unlock(void)
{
	preempt_enable();
	if (IS_ENABLED(CONFIG_RCU_STRICT_GRACE_PERIOD))
		rcu_read_unlock_strict();
}

// include/linux/rcupdate.h
// Preemptible RCU implementation for rcu_read_unlock().
void __rcu_read_unlock(void)
{
	struct task_struct *t = current;

	barrier();  // critical section before exit code.
	if (rcu_preempt_read_exit() == 0) {
		barrier();  // critical-section exit before .s check.
		if (unlikely(READ_ONCE(t->rcu_read_unlock_special.s)))
			rcu_read_unlock_special(t);
	}
	if (IS_ENABLED(CONFIG_PROVE_LOCKING)) {
		int rrln = rcu_preempt_depth();

		WARN_ON_ONCE(rrln < 0 || rrln > RCU_NEST_PMAX);
	}
}
EXPORT_SYMBOL_GPL(__rcu_read_unlock);
```

### 2.3 rcu_dereference()

```c
#define __rcu_dereference_check(p, local, c, space) \
({ \
	/* Dependency order vs. p above. */ \
	typeof(*p) *local = (typeof(*p) *__force)READ_ONCE(p); \
	RCU_LOCKDEP_WARN(!(c), "suspicious rcu_dereference_check() usage"); \
	rcu_check_sparse(p, space); \
	((typeof(*p) __force __kernel *)(local)); \
})
#define __rcu_dereference_protected(p, local, c, space) \
({ \
	RCU_LOCKDEP_WARN(!(c), "suspicious rcu_dereference_protected() usage"); \
	rcu_check_sparse(p, space); \
	((typeof(*p) __force __kernel *)(p)); \
})
#define __rcu_dereference_raw(p, local) \
({ \
	/* Dependency order vs. p above. */ \
	typeof(p) local = READ_ONCE(p); \
	((typeof(*p) __force __kernel *)(local)); \
})
```

- `typeof(*p) *local = (typeof(*p) *__force)READ_ONCE(p);`：使用 `READ_ONCE` 读取指针 `p` 的值，并将其强制转换为 `typeof(*p) *` 类型，存储在 `local` 变量中。
- `RCU_LOCKDEP_WARN(!(c), "suspicious rcu_dereference_check() usage");`：lockdep 检查，如果 c  是 null，就报 warning，一般是配合 `rcu_read_lock_bh_held` 判断进程上下文状态。
- `rcu_check_sparse(p, space);`：sparse 检查
- `((typeof(*p) __force __kernel *)(local));`：返回强制转换为 `typeof(*p) __force __kernel *` 类型的 `local`。

`__rcu_dereference_raw` 没有 sparse 和 lockdep 检查

`__rcu_dereference_protected` 没有 `READ_ONCE`: rcu_dereference_protected 仅适用于更新侧的代码。在这种情况下，调用者通常会持有某种锁（例如自旋锁或互斥锁），以确保指针的值不会在读取过程中被其他线程修改。



## 3. rcu 更新接口

RCU的写端调用了synchronize_rcu/call_rcu两种类型的接口，事实上Linux内核提供了三种不同类型的RCU，因此也对应了相应形式的接口。

* `call_rcu()` /`synchronize_rcu()` : 启动Reclaimer，对旧的临界资源进行回收，其中synchronize_rcu表示同步等待回收，call_rcu表示异步回收；
* `rcu_assign_pointer()`: 移除旧的指针指向，指向更新后的临界资源。



### 3.2 rcu_assign_pointer

```c
#define rcu_assign_pointer(p, v)					      \
do {									      \
	uintptr_t _r_a_p__v = (uintptr_t)(v);				      \
	rcu_check_sparse(p, __rcu);					      \
									      \
	if (__builtin_constant_p(v) && (_r_a_p__v) == (uintptr_t)NULL)	      \
		WRITE_ONCE((p), (typeof(p))(_r_a_p__v));		      \
	else								      \
		smp_store_release(&p, RCU_INITIALIZER((typeof(p))_r_a_p__v)); \
} while (0)
```

1. `uintptr_t _r_a_p__v = (uintptr_t)(v);`：将值 v 转换为无符号整数类型 uintptr_t 并赋值给 _r_a_p__v。这样可以确保指针的大小和类型一致。
2. 检查 v 是否是一个编译时常量，并且是否为 NULL。__builtin_constant_p 是一个 GCC 内建函数，用于判断一个值是否是编译时常量。
3. 如果 v 不是编译时常量或不为 NULL，则使用 smp_store_release 函数以释放语义存储 p。RCU_INITIALIZER 用于初始化 RCU 指针。否则使用 WRITE_ONCE。（关于 WRITE_ONCE 和 smp_store_release 的区别，可以参考 [这一篇博客](https://blog.csdn.net/qq_30896803/article/details/142552445)。

总结来说，这段代码确保在多线程环境中安全地更新指针 p，并且根据 v 的类型和值选择不同的更新方式。

### 3.1 synchronize_rcu

`synchronize_rcu` 和 `call_rcu` 实现如下图所示

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231037032-1917858077.png)

`wait_rcu_gp()`: 具体代码在下面

1. **初始化和注册回调**：如果当前元素是第一次出现，初始化 `rs_array`中对应的 `rcu_head` 和 `completion`，并调用回调函数。
2. **等待所有回调被调用**：再次遍历 `crcu_array`。如果当前元素是第一次出现，等待 [`completion`](vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) 完成，并销毁 `rcu_head`。

这里初始化了 completion，运行 `crcu_array[i]`，实际上也就是 `call_rcu_hurry()`（kernel 6.8 是这个）

``call_rcu_hurry()` 其实会异步运行。它运行结束后，会调用 wakeme_after_rcu，告知后面运行 `wait_for_completion()` 的线程任务已完成。

```c
/*
 * Structure allowing asynchronous waiting on RCU.
 */
struct rcu_synchronize {
	struct rcu_head head;
	struct completion completion;
};

void wakeme_after_rcu(struct rcu_head *head)
{
	struct rcu_synchronize *rcu;

	rcu = container_of(head, struct rcu_synchronize, head);
  // 通知另一个线程 completion 事件已完成
	complete(&rcu->completion);
}

void __wait_rcu_gp(bool checktiny, int n, call_rcu_func_t *crcu_array,
		   struct rcu_synchronize *rs_array)
{	//...
	/* Initialize and register callbacks for each crcu_array element. */
	for (i = 0; i < n; i++) {
		// ...
		for (j = 0; j < i; j++)
			if (crcu_array[j] == crcu_array[i])
				break;
    // 遍历所有 crcu_array_func_t，若第一次出现，则对 rcu_synchronize 进行初始化
		if (j == i) {
			init_rcu_head_on_stack(&rs_array[i].head);
			init_completion(&rs_array[i].completion);
      // 调用 call_rcu_hurry(&rs_array[i].head, wakeme_after_rcu)
			(crcu_array[i])(&rs_array[i].head, wakeme_after_rcu);
		}
	}

	/* Wait for all callbacks to be invoked. */
	for (i = 0; i < n; i++) {
    // ...
		for (j = 0; j < i; j++)
			if (crcu_array[j] == crcu_array[i])
				break;
		if (j == i) {
      // 等待 call_rcu_hurry 完成
			wait_for_completion(&rs_array[i].completion);
			destroy_rcu_head_on_stack(&rs_array[i].head);
		}
	}
}
EXPORT_SYMBOL_GPL(__wait_rcu_gp);
```

我们再来看看 `(crcu_array[i])(&rs_array[i].head, wakeme_after_rcu);` 也就是 `call_rcu_hurry` 里面具体在做什么。

从调用链看 `call_rcu_hurry` -> `__cal_rcu_common` -> `__call_rcu_core`

```c
static void
__call_rcu_common(struct rcu_head *head, rcu_callback_t func, bool lazy_in)
{
  // ...
  
  // 设置回调函数和下一个回调指针，并记录辅助栈信息。
	head->func = func;
	head->next = NULL;
	kasan_record_aux_stack_noalloc(head);
  // 保存中断标志并获取当前 CPU 的 RCU 数据
	local_irq_save(flags);
	rdp = this_cpu_ptr(&rcu_data);
	lazy = lazy_in && !rcu_async_should_hurry();

	// 检查并初始化回调列表
	if (unlikely(!rcu_segcblist_is_enabled(&rdp->cblist))) {
		// Very early boot, before rcu_init().  Initialize if needed
		// and then drop through to queue the callback.
		if (rcu_segcblist_empty(&rdp->cblist))
			rcu_segcblist_init(&rdp->cblist);
	}
  
  // 检查回调过载并尝试旁路
	check_cb_ovld(rdp);
	if (rcu_nocb_try_bypass(rdp, head, &was_alldone, flags, lazy))
		return; // Enqueued onto ->nocb_bypass, so just leave.
  
	// 将回调添加到回调列表
	rcu_segcblist_enqueue(&rdp->cblist, head);
  // ...

	// 处理 RCU 核心处理
	if (unlikely(rcu_rdp_is_offloaded(rdp))) {
		__call_rcu_nocb_wake(rdp, was_alldone, flags); /* unlocks */
	} else {
		__call_rcu_core(rdp, head, flags);
		local_irq_restore(flags);
	}
}

```

在 `__call_ruc_common` 中，有如下几步：

1. 设置回调函数和下一个回调指针，并记录辅助栈信息。
2. 检查回调列表是否过载，否则将回调列表加入列表中，即我们前文提到的 `rcu_data` 的 `cb_list`。
3. 进入 `call_rcu_core` 处理

关于 `call_rcu_core` 的代码代码如下图所示（不过这个图的 kernel 版本比较旧了，不过大致的代码逻辑差不多



![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231114207-1997704645.png)

这里我们简要概括下 `__call_rcu_core` 的流程：

* 这个函数首先检查当前是否处于扩展的静默状态（quiescent state），如果是，则调用 `invoke_rcu_core()` 唤醒 softirq 或者是 `rcu_core_kthread` 处理 RCU callbacks

```c
static void invoke_rcu_core(void)
{
	if (!cpu_online(smp_processor_id()))
		return;
	if (use_softirq)
		raise_softirq(RCU_SOFTIRQ);
	else
		invoke_rcu_core_kthread();
}
```

* 将上层 `rcu_node` 的宽限期信息同步到本 CPU 的 rcu_data 中，主要是宽限期等信息。

```c
static void note_gp_changes(struct rcu_data *rdp)
{
  // ...
	needwake = __note_gp_changes(rnp, rdp);
  // ...
	if (needwake)
		rcu_gp_kthread_wake();
}
```

`__note_gp_changes`决定是否 wakeup gp_kthread，需要则 `swake_up_one_online(&rcu_state.gp_wq);` 唤醒。

那么通过`__call_rcu`注册的这些回调函数在哪里调用呢？我们前面提到，答案是在`RCU_SOFTIRQ`软中断中，我们来看看这一部分的执行过程

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231211961-405526075.png)

6.8 kernel 里面 rcu_process_callbacks 换成了 rcu_core

```c
/* Perform RCU core processing work for the current CPU.  */
static __latent_entropy void rcu_core(void)
{
	// ...
	/* Report any deferred quiescent states if preemption enabled. */
	if (IS_ENABLED(CONFIG_PREEMPT_COUNT) && (!(preempt_count() & PREEMPT_MASK))) {
		rcu_preempt_deferred_qs(current);
	} else if (rcu_preempt_need_deferred_qs(current)) {
		set_tsk_need_resched(current);
		set_preempt_need_resched();
	}

	/* Update RCU state based on any recent quiescent states. */
	rcu_check_quiescent_state(rdp);

	/* No grace period and unregistered callbacks? */
	if (!rcu_gp_in_progress() &&
	    rcu_segcblist_is_enabled(&rdp->cblist) && do_batch) {
		rcu_nocb_lock_irqsave(rdp, flags);
		if (!rcu_segcblist_restempty(&rdp->cblist, RCU_NEXT_READY_TAIL))
			rcu_accelerate_cbs_unlocked(rnp, rdp);
		rcu_nocb_unlock_irqrestore(rdp, flags);
	}

	rcu_check_gp_start_stall(rnp, rdp, rcu_jiffies_till_stall_check());

	/* If there are callbacks ready, invoke them. */
	if (do_batch && rcu_segcblist_ready_cbs(&rdp->cblist) &&
	    likely(READ_ONCE(rcu_scheduler_fully_active))) {
		rcu_do_batch(rdp);
		/* Re-invoke RCU core processing if there are callbacks remaining. */
		if (rcu_segcblist_ready_cbs(&rdp->cblist))
			invoke_rcu_core();
	}
	// ...
}

```

主要就做下面这几件事情。

1. **检查静止状态**: `rcu_check_quiescent_state`
   - 基于 rcu_data 的 graceperiod 更新宽限期。
   - 沿着 rcu-tree 树形结构（如 1. Data structures 小节描述的）层层汇报静止状态。rcu_report_qs_rdp ->  rcu_report_qs_rnp -> rcu_report_qs_rsp -> rcu_gp_kthread_wake

```c
static void
rcu_check_quiescent_state(struct rcu_data *rdp)
{
	/* Check for grace-period ends and beginnings. */
	note_gp_changes(rdp);
  //...
	/*
	 * Tell RCU we are done (but rcu_report_qs_rdp() will be the
	 * judge of that).
	 */
	rcu_report_qs_rdp(rdp);
}
```

2. **处理未注册的回调**：
   1. 如果没有处于宽限期且没有未注册的回调，进行相应处理。

3. **检查宽限期开始的停滞**：`rcu_check_gp_start_stall`
   1. 检查是否需要开始新的宽限期。

4. **调用准备好的回调**：

- 如果有回调准备好，调用它们，并在必要时重新调用 RCU 核心处理。

```c
	/* If there are callbacks ready, invoke them. */
	if (do_batch && rcu_segcblist_ready_cbs(&rdp->cblist) &&
	    likely(READ_ONCE(rcu_scheduler_fully_active))) {
		rcu_do_batch(rdp);
		/* Re-invoke RCU core processing if there are callbacks remaining. */
		if (rcu_segcblist_ready_cbs(&rdp->cblist))
			invoke_rcu_core();
	}
```

* 没处理完则调用 Invoke_rcu_core 继续处理



## 4. 宽限期处理

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231238950-269295968.png)

```c
/*
 * Body of kthread that handles grace periods.
 */
static int __noreturn rcu_gp_kthread(void *unused)
{
	rcu_bind_gp_kthread();
	for (;;) {

		/* Handle grace-period start. */
		for (;;) {
			trace_rcu_grace_period(rcu_state.name, rcu_state.gp_seq,
					       TPS("reqwait"));
			WRITE_ONCE(rcu_state.gp_state, RCU_GP_WAIT_GPS);
			swait_event_idle_exclusive(rcu_state.gp_wq,
					 READ_ONCE(rcu_state.gp_flags) &
					 RCU_GP_FLAG_INIT);
			rcu_gp_torture_wait();
			WRITE_ONCE(rcu_state.gp_state, RCU_GP_DONE_GPS);
			/* Locking provides needed memory barrier. */
			if (rcu_gp_init())
				break;
			cond_resched_tasks_rcu_qs();
			WRITE_ONCE(rcu_state.gp_activity, jiffies);
			WARN_ON(signal_pending(current));
			trace_rcu_grace_period(rcu_state.name, rcu_state.gp_seq,
					       TPS("reqwaitsig"));
		}

		/* Handle quiescent-state forcing. */
		rcu_gp_fqs_loop();

		/* Handle grace-period end. */
		WRITE_ONCE(rcu_state.gp_state, RCU_GP_CLEANUP);
		rcu_gp_cleanup();
		WRITE_ONCE(rcu_state.gp_state, RCU_GP_CLEANED);
	}
}
```

* `rcu_gp_fqs_loop()` 中执行 quitescent-state forcing 直到 graceperiod 停止
* `rcu_gp_cleanup()` 执行 grace period 结束之后执行的操作

其实也可以关注下 rcu_state.gp_state 的变化过程，其脉络也很清晰，下面的说明中，我们主要聚焦于上面这两个函数。

### 4.1 rcu_gp_fqs_loop

```c
static noinline_for_stack void rcu_gp_fqs_loop(void)
{
    bool first_gp_fqs = true; // 标记是否是第一次强制进入静止状态
    int gf = 0; // 标记变量，用于记录RCU的状态标志
    unsigned long j; // 用于记录时间间隔
    int ret; // 返回值，用于控制循环逻辑
    struct rcu_node *rnp = rcu_get_root(); // 获取RCU树的根节点

    for (;;) {
      	// ....
        if (!READ_ONCE(rnp->qsmask) && !rcu_preempt_blocked_readers_cgp(rnp))
            break; // 如果根节点的qsmask为0且没有阻塞的读者，退出循环
      
      	// 执行静止状态 forcing(fqs)
        if (!time_after(rcu_state.jiffies_force_qs, jiffies) || (gf & (RCU_GP_FLAG_FQS | RCU_GP_FLAG_OVLD))) {
        // ...
            rcu_gp_fqs(first_gp_fqs); 
            // ...
        }
    }
}
```

`rcu_gp_fqs_loop` 的核心逻辑在于循环执行 `rcu_gp_fqs` ，直到没有阻塞的读者，所有核都进入静止状态。

而 rcu_gp_fqs 做的事情则是执行一轮 `quiescent-state forcing`，会调用到 `force_qs_rnp`，做的事情就是检测所有的叶子 rcu_node，收集每个 CPU 的静止状态。

```c
static void force_qs_rnp(int (*f)(struct rcu_data *rdp))
{
  // 遍历每个叶子节点的 rcu_node
	rcu_for_each_leaf_node(rnp) {
    // ...
		for_each_leaf_node_cpu_mask(rnp, cpu, rnp->qsmask) {
      // 遍历每个核的 rcu_data
			rdp = per_cpu_ptr(&rcu_data, cpu);
			ret = f(rdp);
			if (ret > 0) {
				mask |= rdp->grpmask;
				rcu_disable_urgency_upon_qs(rdp);
			}
			if (ret < 0)
				rsmask |= rdp->grpmask;
		}
```

### 4.2 rcu_gp_cleanup

这段代码主要步骤包括

1. **初始化和锁定**：
   - 更新 RCU 状态的活动时间。
   - 锁定根节点 [`rnp`](vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)，记录宽限期结束时间，并计算宽限期持续时间。
2. **标记宽限期结束**：
   - 调用 [`rcu_poll_gp_seq_end`](vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) 标记宽限期结束，并解锁根节点。
3. **传播新的宽限期序列**：
   - 计算新的宽限期序列 [`new_gp_seq`](vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)。
   - 遍历所有 RCU 节点，更新它们的 [`gp_seq`](vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)，并检查是否需要新的宽限期。
4. **处理回调和清理**：
   - 检查和处理回调，清理过载的 CPU。
5. **检查是否需要新的宽限期**：
   - 如果需要新的宽限期，设置相应的标志位。
6. **通知所有 CPU 宽限期结束（如果严格模式启用）**：
   - 如果启用了严格模式，通知所有 CPU 宽限期结束。

```c
static noinline void rcu_gp_cleanup(void)
{
    // 更新 RCU 状态的活动时间
    WRITE_ONCE(rcu_state.gp_activity, jiffies);
    raw_spin_lock_irq_rcu_node(rnp);
    rcu_state.gp_end = jiffies;
    gp_duration = rcu_state.gp_end - rcu_state.gp_start;
    if (gp_duration > rcu_state.gp_max)
        rcu_state.gp_max = gp_duration;

    // 标记宽限期结束
    rcu_poll_gp_seq_end(&rcu_state.gp_seq_polled_snap);
    raw_spin_unlock_irq_rcu_node(rnp);

    // 传播新的 gp_seq 值到 rcu_node 结构
    new_gp_seq = rcu_state.gp_seq;
    rcu_seq_end(&new_gp_seq);
    rcu_for_each_node_breadth_first(rnp) {
      	// ...
        rdp = this_cpu_ptr(&rcu_data);
        if (rnp == rdp->mynode)
            needgp = __note_gp_changes(rnp, rdp) || needgp;
        needgp = rcu_future_gp_cleanup(rnp) || needgp;
        if (rcu_is_leaf_node(rnp))
            for_each_leaf_node_cpu_mask(rnp, cpu, rnp->cbovldmask) {
                rdp = per_cpu_ptr(&rcu_data, cpu);
                check_cb_ovld_locked(rdp, rnp);
            }
      // ...
    }
    rnp = rcu_get_root();
    raw_spin_lock_irq_rcu_node(rnp);

    // 声明宽限期结束
    trace_rcu_grace_period(rcu_state.name, rcu_state.gp_seq, TPS("end"));
    rcu_seq_end(&rcu_state.gp_seq);
    ASSERT_EXCLUSIVE_WRITER(rcu_state.gp_seq);
    WRITE_ONCE(rcu_state.gp_state, RCU_GP_IDLE);

    // 检查是否需要新的宽限期
    rdp = this_cpu_ptr(&rcu_data);
    if (!needgp && ULONG_CMP_LT(rnp->gp_seq, rnp->gp_seq_needed)) {
        trace_rcu_this_gp(rnp, rdp, rnp->gp_seq_needed, TPS("CleanupMore"));
        needgp = true;
    }

    // 加速回调处理，把一些 callback 的状态设置为可以处理的，而不是等待 graceperiod
    offloaded = rcu_rdp_is_offloaded(rdp);
    if ((offloaded || !rcu_accelerate_cbs(rnp, rdp)) && needgp) {
        WRITE_ONCE(rcu_state.gp_flags, RCU_GP_FLAG_INIT);
        WRITE_ONCE(rcu_state.gp_req_activity, jiffies);
        trace_rcu_grace_period(rcu_state.name, rcu_state.gp_seq, TPS("newreq"));
    }
}
```



## 5. 静止状态处理

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231305329-888937109.png)

update_process_times -> rcu_sched_clock_irq -> rcu_qs

rcu_qs 中会设置每个 rcu_data 的 qs 状态

```c
/*
 * Note a quiescent state for PREEMPTION=n.  Because we do not need to know
 * how many quiescent states passed, just if there was at least one since
 * the start of the grace period, this just sets a flag.  The caller must
 * have disabled preemption.
 */
static void rcu_qs(void)
{
	RCU_LOCKDEP_WARN(preemptible(), "rcu_qs() invoked with preemption enabled!!!");
	if (!__this_cpu_read(rcu_data.cpu_no_qs.s))
		return;
	trace_rcu_grace_period(TPS("rcu_sched"),
			       __this_cpu_read(rcu_data.gp_seq), TPS("cpuqs"));
	__this_cpu_write(rcu_data.cpu_no_qs.b.norm, false);
	if (__this_cpu_read(rcu_data.cpu_no_qs.b.exp))
		rcu_report_exp_rdp(this_cpu_ptr(&rcu_data));
}
```



## 6. 状态转换

![img](https://img2020.cnblogs.com/blog/1771657/202004/1771657-20200424231324436-1739791125.png)



## reference

1. https://www.cnblogs.com/LoyenWang/p/12770878.html
2. https://www.cnblogs.com/hellokitty2/p/16999547.html
3. https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/Data-Structures.html
