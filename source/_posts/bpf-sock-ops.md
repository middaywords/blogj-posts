---
title: bpf sock ops
date: 2023-08-24 10:49:35
tags:
- bpf
---

## bpf_sock_ops

list of sock_ops:

* BPF_SOCK_OPS_TIMEOUT_INIT



related definitions: 

```c
/* User bpf_sock_ops struct to access socket values and specify request ops
 * and their replies.
 * Some of this fields are in network (bigendian) byte order and may need
 * to be converted before use (bpf_ntohl() defined in samples/bpf/bpf_endian.h).
 * New fields can only be added at the end of this structure
 */
struct bpf_sock_ops {
	__u32 op;
	union {
		__u32 args[4];		/* Optionally passed to bpf program */
		__u32 reply;		/* Returned by bpf program	    */
		__u32 replylong[4];	/* Optionally returned by bpf prog  */
	};
	__u32 family;
	__u32 remote_ip4;	/* Stored in network byte order */
	__u32 local_ip4;	/* Stored in network byte order */
	__u32 remote_ip6[4];	/* Stored in network byte order */
	__u32 local_ip6[4];	/* Stored in network byte order */
	__u32 remote_port;	/* Stored in network byte order */
	__u32 local_port;	/* stored in host byte order */
	__u32 is_fullsock;	/* Some TCP fields are only valid if
				 * there is a full socket. If not, the
				 * fields read as zero.
				 */
	__u32 snd_cwnd;
	__u32 srtt_us;		/* Averaged RTT << 3 in usecs */
	__u32 bpf_sock_ops_cb_flags; /* flags defined in uapi/linux/tcp.h */
	__u32 state;
	__u32 rtt_min;
	__u32 snd_ssthresh;
	__u32 rcv_nxt;
	__u32 snd_nxt;
	__u32 snd_una;
	__u32 mss_cache;
	__u32 ecn_flags;
	__u32 rate_delivered;
	__u32 rate_interval_us;
	__u32 packets_out;
	__u32 retrans_out;
	__u32 total_retrans;
	__u32 segs_in;
	__u32 data_segs_in;
	__u32 segs_out;
	__u32 data_segs_out;
	__u32 lost_out;
	__u32 sacked_out;
	__u32 sk_txhash;
	__u64 bytes_received;
	__u64 bytes_acked;
	__bpf_md_ptr(struct bpf_sock *, sk);
	/* [skb_data, skb_data_end) covers the whole TCP header.
	 *
	 * BPF_SOCK_OPS_PARSE_HDR_OPT_CB: The packet received
	 * BPF_SOCK_OPS_HDR_OPT_LEN_CB:   Not useful because the
	 *                                header has not been written.
	 * BPF_SOCK_OPS_WRITE_HDR_OPT_CB: The header and options have
	 *				  been written so far.
	 * BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB:  The SYNACK that concludes
	 *					the 3WHS.
	 * BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB: The ACK that concludes
	 *					the 3WHS.
	 *
	 * bpf_load_hdr_opt() can also be used to read a particular option.
	 */
	__bpf_md_ptr(void *, skb_data);
	__bpf_md_ptr(void *, skb_data_end);
	__u32 skb_len;		/* The total length of a packet.
				 * It includes the header, options,
				 * and payload.
				 */
	__u32 skb_tcp_flags;	/* tcp_flags of the header.  It provides
				 * an easy way to check for tcp_flags
				 * without parsing skb_data.
				 *
				 * In particular, the skb_tcp_flags
				 * will still be available in
				 * BPF_SOCK_OPS_HDR_OPT_LEN even though
				 * the outgoing header has not
				 * been written yet.
				 */
	__u64 skb_hwtstamp;
};

/* Definitions for bpf_sock_ops_cb_flags */
enum {
	BPF_SOCK_OPS_RTO_CB_FLAG	= (1<<0),
	BPF_SOCK_OPS_RETRANS_CB_FLAG	= (1<<1),
	BPF_SOCK_OPS_STATE_CB_FLAG	= (1<<2),
	BPF_SOCK_OPS_RTT_CB_FLAG	= (1<<3),
	/* Call bpf for all received TCP headers.  The bpf prog will be
	 * called under sock_ops->op == BPF_SOCK_OPS_PARSE_HDR_OPT_CB
	 *
	 * Please refer to the comment in BPF_SOCK_OPS_PARSE_HDR_OPT_CB
	 * for the header option related helpers that will be useful
	 * to the bpf programs.
	 *
	 * It could be used at the client/active side (i.e. connect() side)
	 * when the server told it that the server was in syncookie
	 * mode and required the active side to resend the bpf-written
	 * options.  The active side can keep writing the bpf-options until
	 * it received a valid packet from the server side to confirm
	 * the earlier packet (and options) has been received.  The later
	 * example patch is using it like this at the active side when the
	 * server is in syncookie mode.
	 *
	 * The bpf prog will usually turn this off in the common cases.
	 */
	BPF_SOCK_OPS_PARSE_ALL_HDR_OPT_CB_FLAG	= (1<<4),
	/* Call bpf when kernel has received a header option that
	 * the kernel cannot handle.  The bpf prog will be called under
	 * sock_ops->op == BPF_SOCK_OPS_PARSE_HDR_OPT_CB.
	 *
	 * Please refer to the comment in BPF_SOCK_OPS_PARSE_HDR_OPT_CB
	 * for the header option related helpers that will be useful
	 * to the bpf programs.
	 */
	BPF_SOCK_OPS_PARSE_UNKNOWN_HDR_OPT_CB_FLAG = (1<<5),
	/* Call bpf when the kernel is writing header options for the
	 * outgoing packet.  The bpf prog will first be called
	 * to reserve space in a skb under
	 * sock_ops->op == BPF_SOCK_OPS_HDR_OPT_LEN_CB.  Then
	 * the bpf prog will be called to write the header option(s)
	 * under sock_ops->op == BPF_SOCK_OPS_WRITE_HDR_OPT_CB.
	 *
	 * Please refer to the comment in BPF_SOCK_OPS_HDR_OPT_LEN_CB
	 * and BPF_SOCK_OPS_WRITE_HDR_OPT_CB for the header option
	 * related helpers that will be useful to the bpf programs.
	 *
	 * The kernel gets its chance to reserve space and write
	 * options first before the BPF program does.
	 */
	BPF_SOCK_OPS_WRITE_HDR_OPT_CB_FLAG = (1<<6),
/* Mask of all currently supported cb flags */
	BPF_SOCK_OPS_ALL_CB_FLAGS       = 0x7F,
};

/* List of known BPF sock_ops operators.
 * New entries can only be added at the end
 */
enum {
	BPF_SOCK_OPS_VOID,
	BPF_SOCK_OPS_TIMEOUT_INIT,	/* Should return SYN-RTO value to use or
					 * -1 if default value should be used
					 */
	BPF_SOCK_OPS_RWND_INIT,		/* Should return initial advertized
					 * window (in packets) or -1 if default
					 * value should be used
					 */
	BPF_SOCK_OPS_TCP_CONNECT_CB,	/* Calls BPF program right before an
					 * active connection is initialized
					 */
	BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB,	/* Calls BPF program when an
						 * active connection is
						 * established
						 */
	BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB,	/* Calls BPF program when a
						 * passive connection is
						 * established
						 */
	BPF_SOCK_OPS_NEEDS_ECN,		/* If connection's congestion control
					 * needs ECN
					 */
	BPF_SOCK_OPS_BASE_RTT,		/* Get base RTT. The correct value is
					 * based on the path and may be
					 * dependent on the congestion control
					 * algorithm. In general it indicates
					 * a congestion threshold. RTTs above
					 * this indicate congestion
					 */
	BPF_SOCK_OPS_RTO_CB,		/* Called when an RTO has triggered.
					 * Arg1: value of icsk_retransmits
					 * Arg2: value of icsk_rto
					 * Arg3: whether RTO has expired
					 */
	BPF_SOCK_OPS_RETRANS_CB,	/* Called when skb is retransmitted.
					 * Arg1: sequence number of 1st byte
					 * Arg2: # segments
					 * Arg3: return value of
					 *       tcp_transmit_skb (0 => success)
					 */
	BPF_SOCK_OPS_STATE_CB,		/* Called when TCP changes state.
					 * Arg1: old_state
					 * Arg2: new_state
					 */
	BPF_SOCK_OPS_TCP_LISTEN_CB,	/* Called on listen(2), right after
					 * socket transition to LISTEN state.
					 */
	BPF_SOCK_OPS_RTT_CB,		/* Called on every RTT.
					 */
	BPF_SOCK_OPS_PARSE_HDR_OPT_CB,	/* Parse the header option.
					 * It will be called to handle
					 * the packets received at
					 * an already established
					 * connection.
					 *
					 * sock_ops->skb_data:
					 * Referring to the received skb.
					 * It covers the TCP header only.
					 *
					 * bpf_load_hdr_opt() can also
					 * be used to search for a
					 * particular option.
					 */
	BPF_SOCK_OPS_HDR_OPT_LEN_CB,	/* Reserve space for writing the
					 * header option later in
					 * BPF_SOCK_OPS_WRITE_HDR_OPT_CB.
					 * Arg1: bool want_cookie. (in
					 *       writing SYNACK only)
					 *
					 * sock_ops->skb_data:
					 * Not available because no header has
					 * been	written yet.
					 *
					 * sock_ops->skb_tcp_flags:
					 * The tcp_flags of the
					 * outgoing skb. (e.g. SYN, ACK, FIN).
					 *
					 * bpf_reserve_hdr_opt() should
					 * be used to reserve space.
					 */
	BPF_SOCK_OPS_WRITE_HDR_OPT_CB,	/* Write the header options
					 * Arg1: bool want_cookie. (in
					 *       writing SYNACK only)
					 *
					 * sock_ops->skb_data:
					 * Referring to the outgoing skb.
					 * It covers the TCP header
					 * that has already been written
					 * by the kernel and the
					 * earlier bpf-progs.
					 *
					 * sock_ops->skb_tcp_flags:
					 * The tcp_flags of the outgoing
					 * skb. (e.g. SYN, ACK, FIN).
					 *
					 * bpf_store_hdr_opt() should
					 * be used to write the
					 * option.
					 *
					 * bpf_load_hdr_opt() can also
					 * be used to search for a
					 * particular option that
					 * has already been written
					 * by the kernel or the
					 * earlier bpf-progs.
					 */
};
```

