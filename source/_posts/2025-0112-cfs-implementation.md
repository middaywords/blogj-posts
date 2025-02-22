---
title: about scheduler interface
date: 2025-01-12 18:44:04
tags:
- sched
- linux
- kernel
---

这位博主将 [调度系列](https://hackmd.io/@RinHizakura?tags=%5B%22Scheduler%22%5D) 中将 CFS 的实现解析得非常详尽，

在这篇文章里，我们则关注 sched core 和 sched cfs 之间的接口交互。

(这篇文章后面没细致写了，感觉这样写脉络不是很清晰。还是从一个 task_group 被 fork 以及被kill 开始说

## 0. basic concepts

调度类需要实现的所有接口定义在 struct sched_class 里。下面对其中最重要的一些调度类接口做简单的介绍:

```c
struct sched_class {

#ifdef CONFIG_UCLAMP_TASK
	int uclamp_enabled;
#endif
	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*yield_task)   (struct rq *rq);
	bool (*yield_to_task)(struct rq *rq, struct task_struct *p);
  
	void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);
  
	struct task_struct *(*pick_next_task)(struct rq *rq);

  void (*put_prev_task)(struct rq *rq, struct task_struct *p);
	void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

#ifdef CONFIG_SMP
	int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
	int  (*select_task_rq)(struct task_struct *p, int task_cpu, int flags);

	struct task_struct * (*pick_task)(struct rq *rq);

	void (*migrate_task_rq)(struct task_struct *p, int new_cpu);

	void (*task_woken)(struct rq *this_rq, struct task_struct *task);

	void (*set_cpus_allowed)(struct task_struct *p,
				 const struct cpumask *newmask,
				 u32 flags);

	void (*rq_online)(struct rq *rq);
	void (*rq_offline)(struct rq *rq);

	struct rq *(*find_lock_rq)(struct task_struct *p, struct rq *rq);
#endif

	void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
	void (*task_fork)(struct task_struct *p);
	void (*task_dead)(struct task_struct *p);

	/*
	 * The switched_from() call is allowed to drop rq->lock, therefore we
	 * cannot assume the switched_from/switched_to pair is serialized by
	 * rq->lock. They are however serialized by p->pi_lock.
	 */
	void (*switched_from)(struct rq *this_rq, struct task_struct *task);
	void (*switched_to)  (struct rq *this_rq, struct task_struct *task);
	void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
			      int oldprio);

	unsigned int (*get_rr_interval)(struct rq *rq,
					struct task_struct *task);

	void (*update_curr)(struct rq *rq);

#define TASK_SET_GROUP		0
#define TASK_MOVE_GROUP		1

#ifdef CONFIG_FAIR_GROUP_SCHED
	void (*task_change_group)(struct task_struct *p, int type);
#endif
};
```

* `enqueue_task`: Called when a task enters a runnable state. It puts the scheduling entity (task) into the red-black tree and increments the nr_running variable. 将待运行的任务插入到per-cpu rq。向就绪队列中添加一个任务，当某个任务进入可运行状态时，调用这个函数。典型的场景就是内核里的唤醒函数，将被唤醒的任务插入rq然后设置任务运行态为 TASK_RUNNING。对 CFS 调度器来说，则是将任务插入红黑树，给 nr_running 增加计数。
* `dequeue_task`: When a task is no longer runnable, this function is called to keep the corresponding scheduling entity out of the red-black tree. It decrements the nr_running variable. 将非运行态任务移除出per-cpu rq。将一个任务从就绪队列中删除，典型的场景就是任务调度引起阻塞的内核函数，把任务运行态设置成 TASK_INTERRUPTIBLE 或 TASK_UNINTERRUPTIBLE，然后调用 schedule 函数，最终触发dequeue_task的操作。对 CFS 调度器来说，则是将不在处于运行态的任务从红黑树中移除，给 nr_running 减少计数。

- yield_task：This function is basically just a dequeue followed by an enqueue, unless the compat_yield sysctl is turned on; in that case, it places the scheduling entity at the right-most end of the red-black tree. 处于运行态的任务主动让出 CPU。典型的场景就是处于运行态的应用调用sched_yield系统调用，直接让出 CPU。此时系统调用 sched_yield 系统调用先调用 yield_task 申请让出 CPU，然后调用 schedule 去做上下文切换。对 CFS 调度器来说，如果 nr_running 是 1，则直接返回，最终 schedule 函数也不产生上下文切换。否则，任务被标记为skip 状态。调度器在红黑树上选择待运行任务时肯定会跳过该任务。之后，因为 schedule 函数被调用，pick_next_task 最终会被调用。其代码会从红黑树中最左侧选择一个任务，然后把要放弃运行的任务放回红黑树，然后调用上下文切换函数做任务上下文切换。
- yield_to_task：让处于运行态的任务主动放弃CPU，并执行指定的任务。
- check_preempt_curr：用于在待运行任务插入rq后，检查是否应该抢占正在CPU上运行的当前任务。Wakeup Preemption 的实现逻辑主要在这里。对 CFS 调度器而言，主要是在是否能满足调度时延和是否能保证足够任务运行时间之间来取舍。CFS 调度器也提供了预定义的 Threshold 允许做 Wakeup Preemption 的调用
- pick_next_task：This function chooses the most appropriate task eligible to run next. 选择下一个最适合调度运行的任务，将其从rq移除。并且如果前一个任务还保持在运行态，即没有从rq移除，则将当前的任务重新放回到rq。内核 schedule 函数利用它来完成调度时任务的选择。对CFS调度器而言，大多数情况下，下一个调度任务是从红黑树的最左侧节点选择并移除。如果前一个任务是其它调度类，则调用该调度类的 put_prev_task 方法将前一个任务做正确的安置处理。但如果前一个任务如果也属于CFS调度类的话，为了效率，跳过调度类标准方法 put_prev_task，但核心逻辑仍旧是 put_prev_task_fair 的主要部分。
- put_prev_task：将前一个正在CPU上运行的任务从CPU上拿下的处理。如果任务还在运行态则将任务放回rq，否则，根据调度类要求做简单处理。此函数通常是 pick_next_task 的密切关联操作，是 schedule 实现的关键部分。如果前一个任务属于CFS调度类，则使用CFS调度类的具体实现 put_prev_task_fair。此时，如果任务还是 TASK_RUNNING 状态，则被重新插入到红黑树的最右侧。如果这个任务不是 TASK_RUNNING 状态，则已经从红黑树移除过了，只需要修改CFS 当前任务指针 cfs_rq->curr 即可。
- select_task_rq：为给定的任务选择一个最优的CPU就绪队列rq，返回rq所属的CPU号。典型的使用场景是唤醒，fork/exec 进程时，给进程选择一个rq，这也给调度器一个CPU负载均衡的机会。对CFS调度器而言，主要是根据传入的参数要求找到符合亲和性要求的最空闲的CPU所属的rq。
- task_dead：进程结束时调用。
- switched_from：用于切换调度类。
- switched_to：切换到下一个进程来运行。
- prio_changed：改变进程优先级。
- `task_tick`: This function is mostly called from time tick functions; it might lead to process switch. This drives the running preemption.



## 1. schedule() 调度核心逻辑

调度子系统的核心逻辑在 `__schedule()` 中，进入该函数有三种主要的方式

1. 显式阻塞
2. 返回 userspace 的时候，检查 TIF_NEED_RESCHED 标志后会调用。
3. 任务唤醒

并且调用该函数的时候，要保证禁用抢占。

```c
/*
 * __schedule() is the main scheduler function.
 *
 * The main means of driving the scheduler and thus entering this function are:
 *
 *   1. Explicit blocking: mutex, semaphore, waitqueue, etc.
 *
 *   2. TIF_NEED_RESCHED flag is checked on interrupt and userspace return
 *      paths. For example, see arch/x86/entry_64.S.
 *
 *      To drive preemption between tasks, the scheduler sets the flag in timer
 *      interrupt handler sched_tick().
 *
 *   3. Wakeups don't really cause entry into schedule(). They add a
 *      task to the run-queue and that's it.
 *
 *      Now, if the new task added to the run-queue preempts the current
 *      task, then the wakeup sets TIF_NEED_RESCHED and schedule() gets
 *      called on the nearest possible occasion:
 *
 *       - If the kernel is preemptible (CONFIG_PREEMPTION=y):
 *
 *         - in syscall or exception context, at the next outmost
 *           preempt_enable(). (this might be as soon as the wake_up()'s
 *           spin_unlock()!)
 *
 *         - in IRQ context, return from interrupt-handler to
 *           preemptible context
 *
 *       - If the kernel is not preemptible (CONFIG_PREEMPTION is not set)
 *         then at the next:
 *
 *          - cond_resched() call
 *          - explicit schedule() call
 *          - return from syscall or exception to user-space
 *          - return from interrupt-handler to user-space
 *
 * WARNING: must be called with preemption disabled!
 */
static void __sched notrace __schedule(unsigned int sched_mode)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	unsigned long prev_state;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;
  
  // 当前 cpu 的编号
	cpu = smp_processor_id();
  // 当前 cpu 运行队列
	rq = cpu_rq(cpu);
  // 当前正在运行的任务
	prev = rq->curr;

	schedule_debug(prev, !!sched_mode);
  
  // 如果启用了高分辨率定时器功能，则清除高分辨率定时器。
	if (sched_feat(HRTICK) || sched_feat(HRTICK_DL))
		hrtick_clear(rq);
  
  // 禁用本地中断 & 记录上下文切换信息。
	local_irq_disable();
	rcu_note_context_switch(!!sched_mode);
  
  // 锁定运行队列，并在自旋锁后执行内存屏障，确保操作顺序。
	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
  // 更新运行队列的时钟标志和时钟。
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);
	rq->clock_update_flags = RQCF_UPDATED;

  // 初始化切换计数指针，指向非自愿上下文切换计数。
	switch_count = &prev->nivcsw;
  
  // 读取前一个任务的状态。
	prev_state = READ_ONCE(prev->__state);
  // 如果前一个任务不可抢占且有挂起的信号，则将其状态设置为运行中。
  // 否则，检查任务是否对负载有贡献，并在必要时增加不可中断任务计数。
  // 然后将任务从运行队列中移除，并在任务等待I/O时增加I/O等待计数。
	if (!(sched_mode & SM_MASK_PREEMPT) && prev_state) {
		if (signal_pending_state(prev_state, prev)) {
			WRITE_ONCE(prev->__state, TASK_RUNNING);
		} else {
			prev->sched_contributes_to_load =
				(prev_state & TASK_UNINTERRUPTIBLE) &&
				!(prev_state & TASK_NOLOAD) &&
				!(prev_state & TASK_FROZEN);

			if (prev->sched_contributes_to_load)
				rq->nr_uninterruptible++;

			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}
		}
		switch_count = &prev->nvcsw;
	}

  // 选择下一个要运行的任务。
	next = pick_next_task(rq, prev, &rf);
  // 清除需要重新调度的标志。
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();
  // 如果启用了调度调试，则重置最后一次需要重新调度的时间戳。
#ifdef CONFIG_SCHED_DEBUG
	rq->last_seen_need_resched_ns = 0;
#endif

  // 如果前一个任务和下一个任务不同，则执行上下文切换，更新运行队列的当前任务指针，
  // 增加切换计数，并处理各种记账任务。最后，调用context_switch()函数进行实际的上下文切换。
  // 如果前一个任务和下一个任务相同，则解锁运行队列并执行必要的回调。
	if (likely(prev != next)) {
		rq->nr_switches++;
		RCU_INIT_POINTER(rq->curr, next);
		++*switch_count;

		migrate_disable_switch(rq, prev);
		psi_account_irqtime(rq, prev, next);
		psi_sched_switch(prev, next, !task_on_rq_queued(prev));

		trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next, prev_state);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq_unpin_lock(rq, &rf);
		__balance_callbacks(rq);
		raw_spin_rq_unlock_irq(rq);
	}
}
```



## 2.  enqueue_task - 进程加入运行队列

Called when a task enters a runnable state. It puts the scheduling entity (task) into the red-black tree and increments the nr_running variable.

```c
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
	if (!(flags & ENQUEUE_NOCLOCK))
		update_rq_clock(rq);

	if (!(flags & ENQUEUE_RESTORE)) {
		sched_info_enqueue(rq, p);
		psi_enqueue(p, (flags & ENQUEUE_WAKEUP) && !(flags & ENQUEUE_MIGRATED));
	}

	uclamp_rq_inc(rq, p);
	p->sched_class->enqueue_task(rq, p, flags);

	if (sched_core_enabled(rq))
		sched_core_enqueue(rq, p);
}

void activate_task(struct rq *rq, struct task_struct *p, int flags)
{
	if (task_on_rq_migrating(p))
		flags |= ENQUEUE_MIGRATED;
	if (flags & ENQUEUE_MIGRATED)
		sched_mm_cid_migrate_to(rq, p);

	enqueue_task(rq, p, flags);

	WRITE_ONCE(p->on_rq, TASK_ON_RQ_QUEUED);
	ASSERT_EXCLUSIVE_WRITER(p->on_rq);
}
```



1. `wake_up_new_task`:  **将新创建的进程加入就绪队列，等待调度**

```c
void wake_up_new_task(struct task_struct *p)
{
  // a. 为进程选择一个合适的 CPU
  // b. 为进程指定运行队列
  __set_task_cpu(p, select_task_rq(p, task_cpu(p), WF_FORK));
  
  // c. 将进程添加到运行队列
  rq = __task_rq_lock(p, &rf);
  activate_task(rq, p, ENQUEUE_NOCLOCK);
}
```

a. 调用 `sched_class->select_task_rq()` 方法来选择选择合适的 CPU

* 对于 fair scheduler，我们使用 `select_task_rq_fair`。先考虑使用上一次进程使用的 CPU 逻辑核，或者要唤醒它的物理核的另一个逻辑核。这两个核的缓存大概还是热数据，调度上去执行较快。
* 再考虑调度到相邻共享缓存且 idle 的 CPU。会有两种 path

```c
static int select_task_rq_fair(...) {
  // ...
	if (unlikely(sd)) {
		/* Slow path */
		new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);
	} else if (wake_flags & WF_TTWU) { /* XXX always ? */
		/* Fast path */
		new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);
	}
```

* 在 slow path，会去找负载最小的组。再 fastpath，会考虑共享缓存且 idle 的 CPU，优先选择任务上次运行的 CPU。

b. 指定运行队列，绑定到某个 `struct rq`

c. enqueue task: 调用 `activate_task()` -> `p->sched_class->enqueue_task(...)`



2. 不同核之间发生 migration

move_queued_task: 把已经运行的进程迁移到指定的 CPU 可运行队列中，比如在 set affinity 的时候会调用这个

```c
static struct rq *move_queued_task(struct rq *rq, struct rq_flags *rf,
				   struct task_struct *p, int new_cpu)
{
// ...
  rq_lock(rq, rf);
	WARN_ON_ONCE(task_cpu(p) != new_cpu);
	activate_task(rq, p, 0);
	wakeup_preempt(rq, p, 0);
// ...
}
```

3. Wake up a thread

```c
/**
 * try_to_wake_up - wake up a thread
 ...
 */
int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags) */
```

当一个进程在等待一个 I/O 的时候，会主动放弃 CPU。但是当 I/O 到来的时候，进程往往会被唤醒。这个时候是一个时机。当被唤醒的进程优先级高于 CPU 上的当前进程，就会触发抢占。try_to_wake_up() 调用 ttwu_queue 将这个唤醒的任务添加到队列当中。ttwu_queue 再调用 ttwu_do_activate 激活这个任务。

在 ttwu_do_activate 中，首先调用 activate_task 加入 sched_class 的 cfs_rq 中，然后 ttwu_do_activate 调用 ttwu_do_wakeup，将任务标记为可以运行的。到这里，你会发现，抢占问题只做完了一半。就是标识当前运行中的进程应该被抢占了，但是真正的抢占动作并没有发生。在 **3 taks_tick** 中我们会解释这一点。



## 3. task_tick

常见的现象就是**一个进程执行时间太长了，是时候切换到另一个进程了**。那怎么衡量一个进程的运行时间呢？

在计算机里面有一个时钟，会过一段时间触发一次时钟中断，通知操作系统，时间又过去一个时钟周期，这是个很好的方式，可以查看是否是需要抢占的时间点。时钟中断处理函数会调用 scheduler_tick()，它的代码如下：

```c
void scheduler_tick(void)
{
  int cpu = smp_processor_id();
  struct rq *rq = cpu_rq(cpu);
  struct task_struct *curr = rq->curr;
// ......
  curr->sched_class->task_tick(rq, curr, 0);
  cpu_load_update_active(rq);
  calc_global_load_tick(rq);
// ......
}
```

如果当前运行的进程是普通进程，调度类为 fair_sched_class，调用的处理时钟的函数为 task_tick_fair。我们来看一下它的实现。

```c
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
  struct cfs_rq *cfs_rq;
  struct sched_entity *se = &curr->se;
  for_each_sched_entity(se) {
    cfs_rq = cfs_rq_of(se);
    entity_tick(cfs_rq, se, queued);
  }
......
}
```

根据当前进程的 task_struct，找到对应的调度实体 sched_entity 和 cfs_rq 队列，调用 entity_tick。

```c
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
  update_curr(cfs_rq);
  update_load_avg(curr, UPDATE_TG);
  update_cfs_shares(curr);
.....
  if (cfs_rq->nr_running > 1)
    check_preempt_tick(cfs_rq, curr);
}
```



在 entity_tick 里面，我们又见到了熟悉的 update_curr。它会更新当前进程的 vruntime，然后调用 check_preempt_tick。顾名思义就是，检查是否是时候被抢占了。

```c
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
  unsigned long ideal_runtime, delta_exec;
  struct sched_entity *se;
  s64 delta;
  ideal_runtime = sched_slice(cfs_rq, curr);
  delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
  if (delta_exec > ideal_runtime) {
    resched_curr(rq_of(cfs_rq));
    return;
  }
......
  se = __pick_first_entity(cfs_rq);
  delta = curr->vruntime - se->vruntime;
  if (delta < 0)
    return;
  if (delta > ideal_runtime)
    resched_curr(rq_of(cfs_rq));
}
```

check_preempt_tick 先是调用 sched_slice 函数计算出的 ideal_runtime。ideal_runtime 是一个调度周期中，该进程运行的实际时间。

sum_exec_runtime 指进程总共执行的实际时间，prev_sum_exec_runtime 指上次该进程被调度时已经占用的实际时间。每次在调度一个新的进程时都会把它的 se->prev_sum_exec_runtime = se->sum_exec_runtime，所以 sum_exec_runtime-prev_sum_exec_runtime 就是这次调度占用实际时间。如果这个时间大于 ideal_runtime，则应该被抢占了。

除了这个条件之外，还会通过 __pick_first_entity 取出红黑树中最小的进程。如果当前进程的 vruntime 大于红黑树中最小的进程的 vruntime，且差值大于 ideal_runtime，也应该被抢占了。

**当发现当前进程应该被抢占，不能直接把它踢下来，而是把它标记为应该被抢占。为什么呢？因为进程调度第一定律呀，一定要等待正在运行的进程调用 __schedule 才行啊，所以这里只能先标记一下**。

标记一个进程应该被抢占，都是调用 resched_curr，它会调用 set_tsk_need_resched，标记进程应该被抢占，但是此时此刻，并不真的抢占，而是打上一个标签 TIF_NEED_RESCHED。

```c
static inline void set_tsk_need_resched(struct task_struct *tsk)
{
  set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);
}
```

另外一个可能抢占的场景是**当一个进程被唤醒的时候**。即我们再 **2 enqueue_task** 中提到的 try_to_wake_up 相关的实现。

有位博主用了思维图来描述，总结得很好，借用一下：cr: https://www.cnblogs.com/mysky007/p/12342505.html

![img](https://img2018.cnblogs.com/i-beta/1414775/202002/1414775-20200221182836700-577726174.png)

> 调度, 切换运行进程, 有两种方式
>   \- 进程调用 sleep 或等待 I/O, 主动让出 CPU
>   \- 进程运行一段时间, 被动让出 CPU
> 主动让出 CPU 的方式, 调用 schedule(), schedule() 调用 __schedule()
>   \- __schedule() 取出 rq; 取出当前运行进程的 task_struct
>   \- 调用 pick_next_task 取下一个进程
>     \- 依次调用调度类(优化: 大部分都是普通进程), 因此大多数情况调用 fair_sched_class.pick_next_task[_fair]
>     \- pick_next_task_fair 先取出 cfs_rq 队列, 取出当前运行进程调度实体, 更新 vruntime
>     \- pick_next_entity 取最左节点, 并得到 task_struct, 若与当前进程不一样, 则更新红黑树 cfs_rq
>   \- 进程上下文切换: 切换进程内存空间, 切换寄存器和 CPU 上下文(运行 context_switch)
>     \- context_switch() -> switch_to() -> __switch_to_asm(切换[内核]栈顶指针) -> __switch_to()
>     \- switch_to() 取出 Per CPU 的 tss(任务状态段) 结构体
>     \- > x86 提供以硬件方式切换进程的模式, 为每个进程在内存中维护一个 tss, tss 有所有寄存器, 同时 TR(任务寄存器)指向某个 tss, 更改 TR 会触发换出 tss(旧进程)和换入 tss(新进程), 但切换进程没必要换所有寄存器
>     \- 因此 Linux 中每个 CPU 关联一个 tss, 同时 TR 不变, Linux 中参与进程切换主要是栈顶寄存器
>     \- task_struct 的 thread 结构体保留切换时需要修改的寄存器, 切换时将新进程 thread 写入 CPU tss 中
>     \- 具体各类指针保存位置和时刻
>       \- 用户栈, 切换进程内存空间时切换
>       \- 用户栈顶指针, 内核栈 pt_regs 中弹出
>       \- 用户指令指针, 从内核栈 pt_regs 中弹出
>       \- 内核栈, 由切换的 task_struct 中的 stack 指针指向
>       \- 内核栈顶指针, switch_to_asm 函数切换(保存在 thread 中)
>       \- 内核指令指针寄存器: 进程调度最终都会调用 __schedule, 因此在让出(从当前进程)和取得(从其他进程) CPU 时, 该指针都指向同一个代码位置.

preemption 更多相关的在 **6. wakeup_preempt** 里面有提到



## 4. dequeue_task

与 enqueue_task 类似，它表示 task 被移出调度队列。

主调度器的入口函数是`schedule`，在内核中，当需要将CPU分配给与当前进程不同的另一个进程时，就会调用`schedule`函数来选择下一个可执行进程。`schedule`函数最终调用的是`__schedule`函数，所以这里将`__schedule`函数拆分来讲解。

```c
// kernel/sched/core.c
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
	prev = rq->curr;

	switch_count = &prev->nivcsw;
	if (!preempt && prev->state) {
		if (unlikely(signal_pending_state(prev->state, prev))) {
			prev->state = TASK_RUNNING;	/* 1 */
		} else {
			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);	/* 2 */
			prev->on_rq = 0;
	}

	next = pick_next_task(rq, prev, &rf);	/* 3 */
	clear_tsk_need_resched(prev);	/* 4 */
	clear_preempt_need_resched();

	if (likely(prev != next)) {
		rq = context_switch(rq, prev, next, &rf);	/* 5 */
	}
}
```

* 如果当前进程处于可中断的睡眠状态，同时现在接收到了信号，那么将再次被提升为可运行进程。
* 否则就调用`deactivate_task`函数将当前进程变成不活跃状态，这个函数最终会调用调度器类的`dequeue_task`完成删除运行队列的进程的工作。将进程的on_rq标志位置为0，表示不在就绪队列上了。
* 调用`pick_next_task`函数选择下一个执行的进程。
* 清除当前进程的`TIF_NEED_RESCHED`标志位，意味着不需要重调度。
* 如果下一个被调度执行进程不是当前进程，调用`context_switch`函数进行进程上下文切换。



## 5. yield_task

在进程想要自愿放弃对处理器的控制权时，可以使用 sched_yield 系统调用。这导致内核调用yield_task。

```c
SYSCALL_DEFINE0(sched_yield)
{
	do_sched_yield();
	return 0;
}

static void do_sched_yield(void)
{
	struct rq_flags rf;
	struct rq *rq;

	rq = this_rq_lock_irq(&rf);

	schedstat_inc(rq->yld_count);
	current->sched_class->yield_task(rq);

	preempt_disable();
	rq_unlock_irq(rq, &rf);
	sched_preempt_enable_no_resched();

	schedule();
}
```

* `rq = this_rq_lock_irq(&rf);`: 拿到当前 cpu 所在核的 rq
* `current->sched_class->yield_task(rq);`:  对 rq 执行 yield_task
* `schedule()`: 执行下一次调度 

会调用到 sched_class 的 yield_task 接口。

```c
static void yield_task_fair(struct rq *rq)
{
	struct task_struct *curr = rq->curr;
	struct cfs_rq *cfs_rq = task_cfs_rq(curr);
	struct sched_entity *se = &curr->se;

	/*
	 * Are we the only task in the tree?
	 */
	if (unlikely(rq->nr_running == 1))
		return;

	clear_buddies(cfs_rq, se);

	update_rq_clock(rq);
	/*
	 * Update run-time statistics of the 'current'.
	 */
	update_curr(cfs_rq);
	/*
	 * Tell update_rq_clock() that we've just updated,
	 * so we don't do microscopic update in schedule()
	 * and double the fastpath cost.
	 */
	rq_clock_skip_update(rq);

	se->deadline += calc_delta_fair(se->slice, se);
}

static void __clear_buddies_next(struct sched_entity *se)
{
	for_each_sched_entity(se) {
		struct cfs_rq *cfs_rq = cfs_rq_of(se);
		if (cfs_rq->next != se)
			break;

		cfs_rq->next = NULL;
	}
}

static void clear_buddies(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	if (cfs_rq->next == se)
		__clear_buddies_next(se);
}
```

* `if (unlikely(rq->nr_running == 1))` : 如果 rq 里面只有一个进程，直接返回。
* `clear_buddies(cfs_rq, se);`: 如果下一个准备调度的 sched_entity 是当前 task_struct，则设成空。（下一次不要调度当前 task
* `update_rq_clock(rq);` & `update_curr(cfs_rq);`: 更新当前 CFS 数据，时钟，运行时间等。

简而言之，`yield_task` 就是，如果当前进程对应的 se 是 cfs_rq 下一个要调度的对象，就拿走，并调用 `schedule()` 立即启动下一轮调度。



## 6. wakeup_preempt

This function checks if a task that entered the runnable state should preempt the currently running task.

之前这个函数叫 `check_preempt_curr`，后面改名了。

大多被调用的情况是， `activate_task` 之后，即 `enqueue_task` 被调用之后，则会调用到 `wakeup_preempt`。

```c
void wakeup_preempt(struct rq *rq, struct task_struct *p, int flags)
{
	if (p->sched_class == rq->curr->sched_class)
		rq->curr->sched_class->wakeup_preempt(rq, p, flags);
	else if (sched_class_above(p->sched_class, rq->curr->sched_class))
		resched_curr(rq);

	/*
	 * A queue event has occurred, and we're going to schedule.  In
	 * this case, we can save a useless back to back clock update.
	 */
	if (task_on_rq_queued(rq->curr) && test_tsk_need_resched(rq->curr))
		rq_clock_skip_update(rq);
}
```

看到的被调用的有几种情况

1. `wake_up_new_task`: 当新 task_struct 被调用后，os 会先 `activate_task`，加入 cfs_rq 之后再调用 `wakeup_preempt`。
2. `try_to_wake_up` ->  `ttwu_runnable`: `try_to_wake_up` 的时候，发现 task_struct 已经在某个 rq 中。
3. `try_to_wake_up` -> `ttwu_do_activate`: `try_to_wake_up` 的时候，走常规流程，先 `enqueue_task`，然后 `wakeup_preempt`。
4. `__migrate_swap_task`/`move_queued_task`: 核间 load balance 的时候，涉及到 activate task，完成后会调用 `wakeup_preempt`

在之前我们提到 preemption 其实在之前，只是标记了一下。

>  一个进程应该被抢占，都是调用 resched_curr，它会调用 set_tsk_need_resched，标记进程应该被抢占，但是此时此刻，并不真的抢占，而是打上一个标签 TIF_NEED_RESCHED。

在这里 wakeup_preempt 的实现里面，最终的判定结果是需要抢占的话，也是执行 `resched_curr`:

```c
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void check_preempt_wakeup_fair(struct rq *rq, struct task_struct *p, int wake_flags)
{
  struct task_struct *curr = rq->curr;
	struct sched_entity *se = &curr->se, *pse = &p->se;
	struct cfs_rq *cfs_rq = task_cfs_rq(curr);
	int cse_is_idle, pse_is_idle;

	if (unlikely(se == pse))
		return;

	//...

	/* Idle tasks are by definition preempted by non-idle tasks. */
	if (unlikely(task_has_idle_policy(curr)) &&
	    likely(!task_has_idle_policy(p)))
		goto preempt;


	cse_is_idle = se_is_idle(se);
	pse_is_idle = se_is_idle(pse);

	/*
	 * Preempt an idle group in favor of a non-idle group (and don't preempt
	 * in the inverse case).
	 */
	if (cse_is_idle && !pse_is_idle)
		goto preempt;
	if (cse_is_idle != pse_is_idle)
		return;

	cfs_rq = cfs_rq_of(se);
	update_curr(cfs_rq);

	/*
	 * XXX pick_eevdf(cfs_rq) != se ?
	 */
	if (pick_eevdf(cfs_rq) == pse)
		goto preempt;

	return;

preempt:
	resched_curr(rq);
}
```

这里我们看到

* 当 curr 运行的是 idle process 的时候，会 goto preempt 执行抢占（标记 TIF_NEED_RESCHED）
* 另外当 eevdf 相关的逻辑执行的时候，也会标记 TIF_NEED_RESCHED

那什么时候执行抢占呢？这些 [博客](https://hackmd.io/@sysprog/linux-preempt#%E4%BD%95%E6%99%82%E5%9F%B7%E8%A1%8C%E6%90%B6%E4%BD%94%EF%BC%9F) 和 [博客](https://blog.csdn.net/weixin_45030965/article/details/128646726) 讲的很好，简单说

* 用户态抢占
  * system call 结束，准备返回 user mode。 `do_syscall_64()` -> `syscall_exit_to_user_mode()` -> `exit_to_user_mode_prepare()` -> `exit_to_user_mode_loop()`
  * 中断结束，准备返回 user mode
    * `irqentry_exit() ` -> `irqentry_exit_cond_resched()`
    * `prepare_exit_to_usermode()`->`prepare_exit_to_usermode()`->` exit_to_usermode_loop()`
* 内核态抢占
  * 系统调用 `preempt_enable` 时
  * 从中断返回内核态



## 7 put_prev_task & set_next_task & pick_task()

Called right before p is going to be taken off the CPU. 把 task 拿下 cpu。

有几种情形会用到他们 

1. schedule() 进行 context switch 时
2. tasks migrate between between sched group/classes

这里我们看一下 schedule conext swtich 时的逻辑

在 `__schedule()` 中，挑选下一个 task 的时候调用到了 `next = pick_next_task(rq, prev, &rf);` ，这其中主要做了两件事情。一个是把之前的 task 通过 `put_prev_task()` 从调度类里 **拿下** CPU，再挑选通过 `pick_next_task()`挑选合适的 task，然后把下一个挑选到的 task 通过 `set_next_task` **放上** CPU 。

```c
static struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	struct task_struct *next, *p, *max = NULL;
	const struct cpumask *smt_mask;
	bool fi_before = false;
	bool core_clock_updated = (rq == rq->core);
	unsigned long cookie;
	int i, cpu, occ = 0;
	struct rq *rq_i;
	bool need_sync;
  
  // 如果未启用 CONFIG_SCHED_CORE core aware scheduling。
  // 则调用__pick_next_task。
	if (!sched_core_enabled(rq))
		return __pick_next_task(rq, prev, rf);
  
  // 后面都是 core scheduling 相关的内容，比较复杂，很多细节还不是很理解。
	cpu = cpu_of(rq);
  
  // ...
	return next;
}
```

我们接着看下 `__pick_next_task`，比较清晰易懂

```c

/*
 * Pick up the highest-prio task:
 */
static inline struct task_struct *
__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;

	// 如果 CFS sched_class 是唯一一个有 running task 的，直接调用 cfs 相关的 pick_next_task 函数
	if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
		   rq->nr_running == rq->cfs.h_nr_running)) {

		p = pick_next_task_fair(rq, prev, rf);
		if (unlikely(p == RETRY_TASK))
			goto restart;
    
    // 如果没有选出来，那么就进入 idle
		if (!p) {
			put_prev_task(rq, prev);
			p = pick_next_task_idle(rq);
		}
    // ...
		return p;
	}

restart:
  // 在核间做 balance 后，再做 put_prev_task
	put_prev_task_balance(rq, prev, rf);
  
  // 遍历每个 sched_class 执行 pick_next_task
	for_each_class(class) {
		p = class->pick_next_task(rq);
		if (p)
			return p;
	}

	BUG(); /* The idle class should always have a runnable task. */
}
```

* Fast path
  * 如果 CFS sched_class 是唯一一个有 running task 的，直接调用 cfs 相关的 pick_next_task 接口实现方法
  * 如果没有选出来，那么就进入 idle
* 正常流程
  * 先调用 load balance 接口实现方法再执行 put_prev_task
  * 然后遍历每个 sched_class 执行 pick_next_task

从代码来看，非 core scheduling 的时候，不需要调用 set_next_task。



## 8. select_task_rq

TBD

## 9. migrate_task_rq & balance

```c
/*
 * sched_balance_newidle is called by schedule() if this_cpu is about to become
 * idle. Attempts to pull tasks from other CPUs.
 *
 * Returns:
 *   < 0 - we released the lock and there are !fair tasks present
 *     0 - failed, no new tasks
 *   > 0 - success, new (fair) tasks present
 */
static int sched_balance_newidle(struct rq *this_rq, struct rq_flags *rf) {
  // ...
}
```

当前进程即将变空的时候，尝试从其他 cpu rq 里拉取 task。

migrate_task_rq 表示当 task 从一个 CPU 迁移到了另一个 CPU 之后，发现准备迁移了，调用 migrate_task_rq 方法。







reference: 

* [schedule 函数的调用过程](https://www.cnblogs.com/mysky007/p/12342599.html)
* [抢占式调度](https://www.cnblogs.com/mysky007/p/12342505.html)
* [17 | 调度（下）：抢占式调度是如何发生的？](https://freegeektime.com/100024701/93711/)
* [CSE 306 Operating Systems Linux Process Scheduling](https://www3.cs.stonybrook.edu/~youngkwon/cse306/Lecture16_Linux_Process_Scheduling.pdf)

