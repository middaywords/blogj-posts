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

### TC version

```c
#include <linux/bpf.h>
#include <linux/pkt_cls.h>

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

SEC("tbf")
int tbf_cls(struct __sk_buff *skb)
{
    // Load packet length
    uint32_t pkt_len = skb->len;

    // Load timestamp
    uint64_t now = bpf_ktime_get_ns();

    // Load packet metadata
    u64 key = skb->hash;
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
        return TC_ACT_OK;
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

        return TC_ACT_SHOT;
    }
}

char _license[] SEC("license") = "GPL";
```



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

