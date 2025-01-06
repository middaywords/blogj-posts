---
title: use ftrace/stackcount to find tcp connect/send/recv stack
date: 2023-08-24 13:19:45
tags:
- tcp
- network
---

[TOC]



## trace tcp connect/send/recv

这里用 ftrace 记录下 tcp send/recv 的主要调用路径。一方面是查代码总是看调用路径很麻烦，另一方面是 ftrace 的那些操作流程每次都得查一会儿，干脆写一遍笔记记录下，以后方便查。

machine info:

```shell
root@hn2shz:/sys/kernel/debug/tracing# uname -r
5.15.0-26-generic
root@hn2shz:/sys/kernel/debug/tracing# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04.2 LTS
Release:	22.04
Codename:	jammy
```



### ftrace hands-on

ftrace 有 function 和 function_graph 两种模式，这里面我们用 function graph 来看调用路径

```shell
cd /sys/kernel/debug/tracing
echo 0 > tracing_on # 关闭之前可能仍在tracing的任务
echo > trace # 清空之前的trace内容
echo function_graph > current_tracer # 设置tracer为function graph模式
echo > set_ftrace_filter # 清空输出中的filter
echo tcp_v4_connect > set_graph_function
echo 1 > tracing_on # 开始tracing
echo 0 > tracing_on # 停止tracing
```

另外，其实有个类似的 bcc 工具可以看函数调用栈，还挺方便的 [bcc stackcount](https://github.com/iovisor/bcc/blob/master/tools/stackcount_example.txt)



### client connect (->SYN_SENT)

```c
tcp_v4_connect
    ip_route_output_key_hash
    ip_route_output_ports
    ip_route_output_flow
    tcp_set_state TCP_SYN_SENT..
    inet_hash_connect	// 选择可用端口
    security_sk_classify_flow
    ip_route_output_flow
    sk_setup_caps
    secure_tcp_seq
    secure_tcp_ts_off
    tcp_fastopen_defer_connect
    tcp_connect
        inet_sk_rebuild_header
        tcp_connect_init {
            tcp_v4_md5_lookup;
            tcp_mtup_init;
            ipv4_mtu;
            tcp_sync_mss;
            ipv4_default_advmss;
            tcp_initialize_rcv_mss;
            tcp_select_initial_window;
            tcp_write_queue_purge 
            tcp_chrono_stop;
            tcp_clear_retrans;
        sk_stream_alloc_skb
        	__alloc_skb
        tcp_rbtree_insert
        __tcp_transmit_skb
            skb_clone
            ipv4_default_advmss
            bpf_skops_hdr_opt_len.isra.0
            skb_push
            tcp_options_write
            bpf_skops_write_hdr_opt.isra.0
            tcp_v4_send_check
            ip_queue_xmit
        inet_csk_reset_xmit_timer // 启动重传定时器
```

### Server send back SYN+ACK(LISTEN->SYN_RECV)

后面收包的状态机相关的处理都在 `tcp_v4_do_rcv`-> `tcp_rcv_state_process` 里面。

```c
// L4
tcp_conn_request
    inet_reqsk_alloc	// 分配 request_sock 内核对象
    tcp_parse_options // 解析 tcp options
    tcp_v4_route_req	// 查询路由
    tcp_v4_init_ts_off	// SO_TIMESTAMP 的处理
    tcp_v4_init_seq			// 初始化 SEQ
    tcp_openreq_init_rwin	// 初始化 rwin
    tcp_try_fastopen		// 初始化 rwnd
    inet_csk_reqsk_queue_hash_add	// 加入半连接队列
    tcp_v4_send_synack
        tcp_make_synack	// 构造包
            __alloc_skb
            skb_set_owner_w
            ipv4_default_advmss
            tcp_v4_md5_lookup
            skb_push
            tcp_options_write
        __tcp_v4_send_check 
      ip_build_and_send_pkt
  
```

### client handle SYN+ACK(SYN_SENT->ESTABLISHED)

```c
tcp_rcv_synsent_state_process
	tcp_parse_options
    tcp_ack
    	tcp_sync_mss
    	tcp_clean_rtx_queue
    	tcp_rack_update_reo_wnd
    	tcp_schedule_loss_probe
    	tcp_rearm_rto
    	tcp_newly_delivered
    	tcp_rate_gen
    	tcp_update_pacing_rate
    tcp_sync_mss
    tcp_finish_connect
    	tcp_set_state(sk, TCP_ESTABLISHED)
    	inet_sk_rx_dst_set
    	tcp_init_transfer
    		inet_sk_rebuild_header
    		tcp_init_metrics
    		tcp_init_congestion_control
    		tcp_sndbuf_expand
    		tcp_mstamp_refresh
    tcp_send_ack
    	__alloc_skb
    	__tcp_transmit_skb
    		tcp_established_options
    		skb_push
    		__tcp_select_window
    		tcp_options_write
    		tcp_v4_send_check
    		ip_queue_xmit
```

### Server handle ACK (SYN_RECV->ESTABLISHED)

`tcp_v4_rcv` -> `tcp_check_req`

```c
tcp_check_req
    tcp_parse_options
    tcp_v4_syn_recv_sock
    	tcp_create_openreq_child
    		inet_csk_clone_lock
    		tcp_init_xmit_timers
    		tcp_v6_md5_lookup
    		tcp_bpf_clone
    	inet_sk_rx_dst_set
    	inet_csk_route_child_sock
    		ip_route_output_flow	// 查询路由
    	sk_setup_caps
    	tcp_ca_openreq_child
    		tcp_assign_congestion_control
    		cubictcp_state
		ipv4_mtu
    	tcp_sync_mss
    	ipv4_default_advmss
    	tcp_initialize_rcv_mss
    	__inet_inherit_port
    	inet_ehash_nolisten
    tcp_sync_mss
    tcp_synack_rtt_meas
    inet_csk_complete_hashdance
    
```

### server accept

accept 是从 icsk_accept_queue 里面拿出 sock，不涉及传输层的事情

### tcp send

```c
tcp_sendmsg/tcp_sendmsg_locked
    tcp_rate_check_app_limited
    tcp_send_mss
    tcp_stream_memory_free
    sk_stream_alloc_skb
    skb_entail
    sk_page_frag_refill
    tcp_push
    	__tcp_push_pending_frames
    		tcp_write_xmit
    			tcp_mtu_probe
    			tcp_tso_segs
    			tcp_small_queue_check // TSQ feature
    			__tcp_transmit_skb
----------------
__tcp_transmit_skb
    skb_clone
    tcp_established_options
    skb_push
    __tcp_select_window
    tcp_options_write
    tcp_v4_send_check
    ip_queue_xmit
```

### tcp recv

```c
tcp_rcv_established
    tcp_mstamp_refresh
    tcp_queue_rcv
    tcp_event_data_recv
    tcp_ack
    	tcp_clean_rtx_queue
    		tcp_rack_advance
    		tcp_rate_skb_delivered
    		tcp_ack_tstamp
    		__kfree_skb
    		tcp_chrono_stop
    		tcp_ack_update_rtt // bpf ?
    		cubictcp_acked
    	tcp_rack_update_reo_wnd
    	tcp_schedule_loss_probe
    	tcp_rearm_rto
    	tcp_newly_delivered
    	tcp_rate_gen
    	cubictcp_cong_avoid
    	tcp_update_pacing_rate
    tcp_check_space
    __tcp_ack_snd_check
    	tcp_send_ack
            __alloc_skb
            __tcp_transmit_skb
    tcp_data_ready // 唤醒socket上阻塞的进程
```







