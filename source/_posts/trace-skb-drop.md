---
title: trace skb drop
date: 2023-09-05 16:30:22
tags:
- linux
- network
---

## trace skb drop

我们知道，有很多方式可以 trace，我们想看下有哪些 drop 姿势

### trace method

1. ftrace

```shell
$ echo > trace # 清空 trace
$ echo function > current_tracer # 修改 tracer 类型
$ echo 1 > options/func_stack_trace # 启用打印调用栈选项
$ echo kfree_skbmem > set_ftrace_filter # 追踪 skb 释放函数
```

2. Bpftrace

```shell
$ bpftrace -e 'kprobe:kfree_skbmem { @[kstack] = count(); }'
```



### drop reason

在 5.17 kernel 之后有了 `kfree_skb_reason` 的支持才比较好，说明了 drop 的原因，这种一般代表不正常的 drop。还有正常接到包的 drop，这种一般直接调用 `__kfree_skb`。 `__kfree_skb`  底层调用 kfree_skbmem，还有 `__consume_stateless_skb`，这种是给 udp recvmsg 服务的，底层也是调用的 `kfree_skbmem`。总共关系呢，就是这样：

```
kfree_skb_reason dev_kfree_skb_irq dev_kfree_skb_...
     |
  kfree_skb kfree_skb_partial ...
     | 
 __kfree_skb      __consume_stateless_skb
      |             |
        kfree_skbmem
```

* application 调用 tcp 的 recvmsg 的时候会释放

```
---
 => kfree_skbmem
 => __kfree_skb
 => tcp_recvmsg_locked
 => tcp_recvmsg
 => inet_recvmsg
 => sock_recvmsg
 => sock_read_iter
 => new_sync_read
 => vfs_read
 => ksys_read
 => __x64_sys_read
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
```

* 调用 udp 的 recvmsg 的时候也释放

```
=> kfree_skbmem
 => __consume_stateless_skb
 => skb_consume_udp
 => udp_recvmsg
 => inet_recvmsg
 => sock_recvmsg
 => __sys_recvfrom
 => __x64_sys_recvfrom
 => do_syscall_64
```

* softirq  里收包的时候直接 clean_rtx_queue，清理发送包(不是 skb_clone 的那个)

```
=> kfree_skbmem
 => __kfree_skb
 => tcp_clean_rtx_queue
 => tcp_ack
 => tcp_rcv_established
 => tcp_v4_do_rcv
 => tcp_v4_rcv
 => ip_protocol_deliver_rcu
 => ip_local_deliver_finish
 => ip_local_deliver
 => ip_rcv_finish
 => ip_rcv
 => __netif_receive_skb_one_core
 => __netif_receive_skb
 => process_backlog
 => __napi_poll
 => net_rx_action
 => __do_softirq
 => run_ksoftirqd
 => smpboot_thread_fn
 => kthread
 => ret_from_fork
```

* softirq 里面收包清理 skb 的

``` => kfree_skbmem
 => __kfree_skb
 => tcp_data_queue
 => tcp_rcv_established
 => tcp_v4_do_rcv
 => tcp_v4_rcv
 => ip_protocol_deliver_rcu
 => ip_local_deliver_finish
 => ip_local_deliver
 => ip_rcv_finish
 => ip_rcv
 => __netif_receive_skb_one_core
 => __netif_receive_skb
 => process_backlog
 => __napi_poll
 => net_rx_action
 => __do_softirq
 => do_softirq
```

* 清理发送包(skb_clone的那个)

```
 => kfree_skbmem
 => napi_consume_skb
 => free_old_xmit_skbs
 => virtnet_poll_tx
 => __napi_poll
 => net_rx_action
 => __do_softirq
 => irq_exit_rcu
 => common_interrupt
 => asm_common_interrupt
 => native_safe_halt
 => arch_cpu_idle
 => default_idle_call
 => do_idle
 => cpu_startup_entry
 => start_secondary
 => secondary_startup_64_no_verify
```

