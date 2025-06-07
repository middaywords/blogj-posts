---
title: tcp tuning
date: 2025-01-06 16:21:54
tags:
- tcp
- linux
- network
---



```bash
root@tess-node-6xlzc-tess11:/sysroot/home/tess-admin# ls /proc/sys/net/ipv4/tcp_
tcp_abort_on_overflow               tcp_fastopen                        tcp_moderate_rcvbuf                 tcp_retries2
tcp_adv_win_scale                   tcp_fastopen_blackhole_timeout_sec  tcp_mtu_probe_floor                 tcp_rfc1337
tcp_allowed_congestion_control      tcp_fastopen_key                    tcp_mtu_probing                     tcp_rmem
tcp_app_win                         tcp_fin_timeout                     tcp_no_metrics_save                 tcp_sack
tcp_autocorking                     tcp_frto                            tcp_no_ssthresh_metrics_save        tcp_shrink_window
tcp_available_congestion_control    tcp_fwmark_accept                   tcp_notsent_lowat                   tcp_slow_start_after_idle
tcp_available_ulp                   tcp_invalid_ratelimit               tcp_orphan_retries                  tcp_stdurg
tcp_backlog_ack_defer               tcp_keepalive_intvl                 tcp_pacing_ca_ratio                 tcp_syn_linear_timeouts
tcp_base_mss                        tcp_keepalive_probes                tcp_pacing_ss_ratio                 tcp_syn_retries
tcp_challenge_ack_limit             tcp_keepalive_time                  tcp_pingpong_thresh                 tcp_synack_retries
tcp_child_ehash_entries             tcp_l3mdev_accept                   tcp_plb_cong_thresh                 tcp_syncookies
tcp_comp_sack_delay_ns              tcp_limit_output_bytes              tcp_plb_enabled                     tcp_thin_linear_timeouts
tcp_comp_sack_nr                    tcp_low_latency                     tcp_plb_idle_rehash_rounds          tcp_timestamps
tcp_comp_sack_slack_ns              tcp_max_orphans                     tcp_plb_rehash_rounds               tcp_tso_rtt_log
tcp_congestion_control              tcp_max_reordering                  tcp_plb_suspend_rto_sec             tcp_tso_win_divisor
tcp_dsack                           tcp_max_syn_backlog                 tcp_probe_interval                  tcp_tw_reuse
tcp_early_demux                     tcp_max_tw_buckets                  tcp_probe_threshold                 tcp_window_scaling
tcp_early_retrans                   tcp_mem                             tcp_recovery                        tcp_wmem
tcp_ecn                             tcp_migrate_req                     tcp_reflect_tos                     tcp_workaround_signed_windows
tcp_ecn_fallback                    tcp_min_rtt_wlen                    tcp_reordering                      
tcp_ehash_entries                   tcp_min_snd_mss                     tcp_retrans_collapse                
tcp_fack                            tcp_min_tso_segs                    tcp_retries1    
```



| param                 | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| tcp_abort_on_overflow | If listening service is too slow to accept new connections, reset them. Default state is FALSE. It means that if overflow occurred due to a burst, connection will recover. Enable this option *only* if you are really sure that listening daemon cannot be tuned to accept connections faster. Enabling this option can harm clients of your server. |
| tcp_adv_win_scale     | used to account for the overhead needed by Linux to process packets. The receive window is specified in terms of user payload bytes. Linux needs additional memory beyond that to track other data associated with packets it is processing. can refer the details in https://blog.cloudflare.com/optimizing-tcp-for-high-throughput-and-low-latency/ |
| tcp_app_win           |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |
|                       |                                                              |

