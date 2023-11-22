---
title: tc bpf
date: 2023-11-13 17:27:25
tags:
- bpf
---

# tc bpf

TC bpf 在 cilium 里用的很多，但是之前一直了解得不太多。最近在看些相关的东西，于是记一些相关的东西。



## TC framework





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





## reference 

[1] [[译] 深入理解 tc ebpf 的 direct-action (da) 模式（2020）](https://arthurchiao.art/blog/understanding-tc-da-mode-zh/)

[2] [你的第一个TC BPF 程序](https://cloud.tencent.com/developer/article/1626377)

[3] [TC的一些有趣功能](https://yanhang.me/post/2021-tc-ebpf/)
