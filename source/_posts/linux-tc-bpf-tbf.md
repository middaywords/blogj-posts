---
title: linux tc (5) bpf-tbf
date: 2024-02-07 09:11:27
tags:
---

# linux tc 系列(5) - 实现 tbf

## background & intro

idea 源自 LPC 的 talk，当时说实现起来复杂，因为 map 不提供 concurrency 保证。

1. 1. http://vger.kernel.org/lpc_bpf2018_talks/LPC2018-TokenBucket_v4.pdf 
   2. https://blog.cloudflare.com/cloudflare-architecture-and-how-bpf-eats-the-world 

根据 token bucket filter 的算法 https://en.wikipedia.org/wiki/Token_bucket，我们知道 token bucket 计算有两步:

1. fill token and cap by burst

$$
token_{n+1} = max\{token_{n} + rate*time\_gap, BURST\}
$$

2. substract token by data bytes sent

$$
token_{n+1} = token_{n+1} - packet\_size
$$



而这两部需要读写分离，也需要 concurrency promise，之前在 talk 时，并没有 bpf 相关的同步源语，而之后 kernel 版本中引入了 `bpf_spin_lock()`，我们就可以借助其实现 token bucket 了。

## implementation



### xdp version

```c
#include <linux/bpf.h>
#include <linux/pkt_cls.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>

#define RATE_LIMIT 1000000 // 1Gbps in bps
#define BURST_SIZE 125000   // 125KB, adjust as needed
#define LATENCY_NS 50000    // 50ms in nanoseconds

struct tbf_state {
    uint64_t last_time;
    uint32_t tokens;
};

BPF_HASH(tbf_map, u64, struct tbf_state, 65536);
BPF_PERCPU_ARRAY(dropped_packets, u64, 1);
BPF_PERCPU_ARRAY(dropped_bytes, u64, 1);
BPF_SPIN_LOCK(lock);

SEC("xdp")
int tbf_xdp(struct xdp_md *ctx)
{
    // Load packet data pointer and end
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;

    // Load packet length
    uint32_t pkt_len = data_end - data;

    // Load timestamp
    uint64_t now = bpf_ktime_get_ns();

    // Load packet metadata
    u64 key = bpf_get_prandom_u32();
    struct tbf_state *state;
    struct tbf_state new_state;

    // Acquire lock
    bpf_spin_lock(&lock);

    state = tbf_map.lookup(&key);
    if (!state) {
        new_state.last_time = now;
        new_state.tokens = BURST_SIZE;
        tbf_map.update(&key, &new_state);
        state = &new_state;
    }

    // Calculate elapsed time since last packet
    uint64_t elapsed_time = now - state->last_time;

    // Calculate tokens added since last packet
    uint32_t tokens_added = elapsed_time * RATE_LIMIT / 1000000000;

    // Add tokens to the bucket
    state->tokens += tokens_added;

    // Clamp tokens to burst size
    if (state->tokens > BURST_SIZE)
        state->tokens = BURST_SIZE;

    // Check if there are enough tokens to transmit the packet
    if (pkt_len <= state->tokens)
    {
        // Transmit packet
        state->tokens -= pkt_len;
        state->last_time = now;  // Update last time
        bpf_spin_unlock(&lock); // Release lock
        return XDP_PASS;
    }
    else
    {
        // Drop packet
        bpf_spin_unlock(&lock); // Release lock

        // Increment dropped packets counter
        unsigned long long *dropped_pkts = dropped_packets.lookup(0);
        if (dropped_pkts) {
            (*dropped_pkts)++;
        }

        // Increment dropped bytes counter
        unsigned long long *dropped_bytes_ptr = dropped_bytes.lookup(0);
        if (dropped_bytes_ptr) {
            (*dropped_bytes_ptr) += pkt_len;
        }

        return XDP_DROP;
    }
}

char _license[] SEC("license") = "GPL";

```

Performance: retransmission 非常多，这是为什么呢

```
~/proj/xdp-tutorial/experiment02-policing (master ✗) iperf3 -c 10.11.1.2 -R
Connecting to host 10.11.1.2, port 5201
Reverse mode, remote host 10.11.1.2 is sending
[  5] local 10.11.1.1 port 42838 connected to 10.11.1.2 port 5201
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  1.74 MBytes  14.6 Mbits/sec
[  5]   1.00-2.00   sec   848 KBytes  6.95 Mbits/sevvvvcbdhchrglvrncchfdlgiuukehbdldjc
[  5]   2.00-3.00   sec  1.02 MBytes  8.60 Mbits/sec
[  5]   3.00-4.00   sec   836 KBytes  6.85 Mbits/sec
[  5]   4.00-5.00   sec  1.04 MBytes  8.74 Mbits/sec
[  5]   5.00-6.00   sec   884 KBytes  7.24 Mbits/sec
[  5]   6.00-7.00   sec   840 KBytes  6.88 Mbits/sec
[  5]   7.00-8.00   sec  1.02 MBytes  8.57 Mbits/sec
[  5]   8.00-9.00   sec   870 KBytes  7.12 Mbits/sec
[  5]   9.00-10.00  sec   878 KBytes  7.19 Mbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  10.1 MBytes  8.50 Mbits/sec  1239             sender
[  5]   0.00-10.00  sec  9.87 MBytes  8.28 Mbits/sec                  receiver

iperf Done.
```



bcc netqtop 能看到包大概在 

```
-----------------------------------------------------------------------------
Tue Apr  2 00:51:10 2024
TX
 QueueID    avg_size   [0, 64)    [64, 512)  [512, 2K)  [2K, 16K)  [16K, 64K)
 0          70.87      1          1.31K      0          0          0
 Total      70.87      1          1.31K      0          0          0

RX
 QueueID    avg_size   [0, 64)    [64, 512)  [512, 2K)  [2K, 16K)  [16K, 64K)
 0          6.26K      20         1          321        1.24K      39
 Total      6.26K      20         1          321        1.24K      39
-----------------------------------------------------------------------------
```



stack 基本上都是

```
        tcp_retransmit_skb+1
        tcp_write_timer_handler+355
        tcp_write_timer+158
        call_timer_fn+44
        run_timer_softirq+992
        __do_softirq+218
        irq_exit_rcu+121
        sysvec_apic_timer_interrupt+124
        asm_sysvec_apic_timer_interrupt+18
        native_safe_halt+11
        arch_cpu_idle+18
        default_idle_call+53
        do_idle+521
        cpu_startup_entry+32
        start_secondary+298
        secondary_startup_64_no_verify+194


        tcp_retransmit_skb+1
        tcp_xmit_retransmit_queue+25
        tcp_xmit_recovery.part.0+23
        tcp_ack+1277
        tcp_rcv_established+392
        tcp_v4_do_rcv+320
        tcp_v4_rcv+3449
        ip_protocol_deliver_rcu+53
        ip_local_deliver_finish+72
        ip_local_deliver+245
        ip_rcv_finish+182
        ip_rcv+199
        __netif_receive_skb_one_core+136
        __netif_receive_skb+21
        process_backlog+169
        __napi_poll+48
        net_rx_action+284
        __do_softirq+218
        irq_exit_rcu+121
        sysvec_apic_timer_interrupt+124
        asm_sysvec_apic_timer_interrupt+18
        native_safe_halt+11
        arch_cpu_idle+18
        default_idle_call+53
        do_idle+521
        cpu_startup_entry+32
        start_secondary+298
        secondary_startup_64_no_verify+194
```

`tcp_xmit_recovery.part.0+23` 看起来像快重传。



限速 800M

好吧，其实 TC 也是重传很高的...

```
$ iperf3 -c 10.11.1.2 -R -t 60
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  6.36 GBytes   911 Mbits/sec  27296             sender
-----------------------------------------------------------

[  5]  31.00-32.00  sec  95.4 MBytes   800 Mbits/sec    0    370 KBytes
[  5]  32.00-33.00  sec  95.4 MBytes   800 Mbits/sec    0    370 KBytes
[  5]  33.00-34.00  sec  95.4 MBytes   800 Mbits/sec    0    370 KBytes
[  5]  34.00-35.00  sec  95.4 MBytes   800 Mbits/sec    0    370 KBytes
[  5]  35.00-36.00  sec  95.4 MBytes   800 Mbits/sec    0    389 KBytes
```

```
$ iperf3 -c 10.11.1.2 -R -t 60 -b 800M
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  5.59 GBytes   800 Mbits/sec    0             sender
[  5]   0.00-60.00  sec  5.59 GBytes   800 Mbits/sec                  receiver
```

按我们的 XDP 来说，也还是会有 很多 retrans，我感觉会不会是并发出了问题

```
$ iperf3 -c 10.11.1.2 -R -t 60 -b 800M
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  5.43 GBytes   778 Mbits/sec  26494             sender
[  5]   0.00-60.00  sec  5.43 GBytes   777 Mbits/sec                  receiver

...
[  5]  17.00-18.00  sec  95.4 MBytes   800 Mbits/sec    0    354 KBytes
[  5]  18.00-19.00  sec  95.4 MBytes   800 Mbits/sec    0    354 KBytes
[  5]  19.00-20.00  sec  95.4 MBytes   800 Mbits/sec    0    354 KBytes
[  5]  20.00-21.00  sec  95.4 MBytes   800 Mbits/sec    0    354 KBytes
[  5]  21.00-22.00  sec  85.5 MBytes   717 Mbits/sec   75   7.07 KBytes
[  5]  22.00-23.00  sec  92.4 MBytes   775 Mbits/sec  962   1.41 KBytes
[  5]  23.00-24.00  sec  91.4 MBytes   766 Mbits/sec  379   5.66 KBytes
```

而且很神奇的是，XDP 是从第20s开始的，前 20s 并不丢，难道是 concurrency没做好？而 TC filter 一直都是很稳定的

```
$ iperf3 -c 10.11.1.2 -R -t 60
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  5.42 GBytes   776 Mbits/sec  36283             sender
[  5]   0.00-60.00  sec  5.42 GBytes   775 Mbits/sec                  receiver
```

