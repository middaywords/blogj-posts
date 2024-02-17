---
title: tc bpf
date: 2023-11-13 17:27:25
tags:
- bpf
- tc
---

# linux tc 系列(2) - tc bpf

TC bpf 在 cilium 里用的很多，但是之前一直了解得不太多。最近在看些相关的东西，于是记一些相关的东西。



## TC bpf management

我们可以从 tc bpf 的查询接口管中窥豹。

```c
// net/sched/cls_bpf.c
cls_bpf_dump
    cls_bpf_dump_ebpf_info
    cls_bpf_dump_bpf_info
    	nla_put_u32(skb, TCA_BPF_ID, prog->filter->aux->id)

tc_dump_tfilter
    tcf_chain_dump
        tcf_node_dump
            tp->ops->dump
```



## tc bpf create/load

对于 filter 类型， cls_bpf.c 中有相关定义。

```
cls_bpf_prog_from_ops
    bpf_prog_create - create an unattached filter
        bpf_check_basics_ok
        bpf_prog_alloc
        bpf_prepare_filter
            bpf_check_classic - verify socket filter code
cls_bpf_prog_from_efd - bpf program from an existing fd
	bpf_prog_get_type_dev
	
```

1. 在 sch_handle_ingress 与 sch_handle_egress 两个点（未进入 qdisc），调用 `tcf_classify`

```
__netif_receive_skb_core
    sch_handle_egress
    sch_handle_ingress
        tc_run [net/core/dev.c]
            tcf_classify [net/sched/cls_api.c]
                __tcf_classify
                    cls_bpf_classify
                        ->classify() [net/sched/cls_bpf.c]
```

```c
#ifdef CONFIG_NET_XGRESS
static int tc_run(struct tcx_entry *entry, struct sk_buff *skb)
{
	int ret = TC_ACT_UNSPEC;
#ifdef CONFIG_NET_CLS_ACT
	struct mini_Qdisc *miniq = rcu_dereference_bh(entry->miniq);
	struct tcf_result res;

	if (!miniq)
		return ret;
    // ...
	mini_qdisc_bstats_cpu_update(miniq, skb);
	ret = tcf_classify(skb, miniq->block, miniq->filter_list, &res, false);
    /* Only tcf related quirks below. */
	switch (ret) {
	case TC_ACT_SHOT:
		mini_qdisc_qstats_cpu_drop(miniq);
		break;
	case TC_ACT_OK:
	case TC_ACT_RECLASSIFY:
		skb->tc_index = TC_H_MIN(res.classid);
		break;
	}
#endif /* CONFIG_NET_CLS_ACT */
```

2. 在 qdisc 中，会调用 `tcf_classify` 进行分类。对于不同的 qdisc 类型，会有不同的含义，我们以 htb 为例

```c
// net/sched/sch_htb.c
static struct htb_class *htb_classify(struct sk_buff *skb, struct Qdisc *sch,
				      int *qerr)
{
    // ...
    // 根据 priority 找到 filter
    if (skb->priority == sch->handle)
		return HTB_DIRECT;	/* X:0 (direct flow) selected */
	cl = htb_find(skb->priority, sch);
	if (cl) {
		if (cl->level == 0)
			return cl;
		/* Start with inner filter chain if a non-leaf class is selected */
		tcf = rcu_dereference_bh(cl->filter_list);
	} else {
		tcf = rcu_dereference_bh(q->filter_list);
	}
    
    // 循环迭代 filter
	while (tcf && (result = tcf_classify(skb, NULL, tcf, &res, false)) >= 0) {
        #ifdef CONFIG_NET_CLS_ACT
		switch (result) {
		case TC_ACT_QUEUED:
		case TC_ACT_STOLEN:
        //...
		}
#endif
        // 找到 class
		cl = (void *)res.class;
		if (!cl) {
			if (res.classid == sch->handle)
				return HTB_DIRECT;	/* X:0 (direct flow) */
			cl = htb_find(res.classid, sch);
			if (!cl)
				break;	/* filter selected invalid classid */
		}
		if (!cl->level)
            // 叶子节点 class
			return cl;	/* we hit leaf; return it */

		/* we have got inner class; apply inner filter chain */
		tcf = rcu_dereference_bh(cl->filter_list);
	}
	// 根据结果返回分类
	cl = htb_find(TC_H_MAKE(TC_H_MAJ(sch->handle), q->defcls), sch);
    return cl;
}
```

我们看到 `net/sched/` 下一堆 `cls_*.c` 里则是一些 filter 的实现了。

```c
TC_INDIRECT_SCOPE int cls_bpf_classify(struct sk_buff *skb,
				       const struct tcf_proto *tp,
				       struct tcf_result *res) {
    // ...
    list_for_each_entry_rcu(prog, &head->plist, link) {
		int filter_res;
        // 运行 bpf_prog_run 拿到 filter_res
        if (tc_skip_sw(prog->gen_flags)) {
			filter_res = prog->exts_integrated ? TC_ACT_UNSPEC : 0;
		} else if (at_ingress) {
			/* It is safe to push/pull even if skb_shared() */
			__skb_push(skb, skb->mac_len);
			bpf_compute_data_pointers(skb);
			filter_res = bpf_prog_run(prog->filter, skb);
			__skb_pull(skb, skb->mac_len);
		} else {
			bpf_compute_data_pointers(skb);
			filter_res = bpf_prog_run(prog->filter, skb);
		}
        // ...
        // 根据 filter_rs 设置 结果 (cls_act 的情况)
        if (prog->exts_integrated) { 
			res->class   = 0;
			res->classid = TC_H_MAJ(prog->res.classid) |
				       qdisc_skb_cb(skb)->tc_classid;

			ret = cls_bpf_exec_opcode(filter_res);
			if (ret == TC_ACT_UNSPEC)
				continue;
			break;
		}
        // 非 cls_act
        if (filter_res == 0)
			continue;
		if (filter_res != -1) {
			res->class   = 0;
			res->classid = filter_res;
		} else {
			*res = prog->res;
		}
        // 根据 class 执行 action
        ret = tcf_exts_exec(skb, &prog->exts, res);
		if (ret < 0)
			continue;

		break;
    }
}
```



而 `tcf_exts_exec` 则会调用到 `act_*.c` 里的 `->act()` 方法

```c
//tcf_exts_exec
//-> tcf_action_exec
//-> tc_act

static inline int tc_act(struct sk_buff *skb, const struct tc_action *a,
			   struct tcf_result *res)
{
	return a->ops->act(skb, a, res);
}
```



## reference 

[1] [[译] 深入理解 tc ebpf 的 direct-action (da) 模式（2020）](https://arthurchiao.art/blog/understanding-tc-da-mode-zh/)

[2] [你的第一个TC BPF 程序](https://cloud.tencent.com/developer/article/1626377)

[3] [TC的一些有趣功能](https://yanhang.me/post/2021-tc-ebpf/)
