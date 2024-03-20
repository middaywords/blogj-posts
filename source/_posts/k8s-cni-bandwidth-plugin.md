---
title: k8s-cni-bandwidth-plugin
date: 2024-03-20 21:50:25
tags:
- kubernetes
- cni
- tc
---

# k8s bandwidth CNI plugin: policing v.s. shaping ?

Recently, I've been investigating the design of bandwidth control for kubernetes pod, and I found CNI official bandwidth plugin then.

https://www.cni.dev/plugins/current/meta/bandwidth/

## test setting

I setup a kind cluster, with 1 master node and two worker nodes, then, use network benchmark tools to test. The test results shows that its performance is not good.

Generally, I put client pods on one node `kind-worker2` and server pod on node `kind-worker`

```
/var/home/centos# kubectl get pods -n kube-system -owide | grep netperf
netperf-client                         1/1     Running   1 (5d19h ago)   45d     10.244.110.134   kind-worker2         <none>           <none>
netperf-server                         1/1     Running   1 (5d19h ago)   45d     10.244.162.130   kind-worker          <none>           <none>
```

I have set bandwidth limit for client pod, both ingress and egress:

```yaml
annotations:
	kubernetes.io/ingress-bandwidth: 100M
	kubernetes.io/egress-bandwidth: 100M
```

I setup two tcp connections for test, one is TCP_RR for measuring latency and the other using TCP_STREAM for measuring throughput. The test results are shown below:

| Data flow direction                                   | Throughput(Mbps) for TCP_STREAM flow | Latency(us) |
| ----------------------------------------------------- | ------------------------------------ | ----------- |
| netperf-client -> netperf-server (egress rate limit)  | 95.6                                 | 10760.05    |
| netperf-server -> netperf-client (ingress rate limit) | 95.6                                 | 226120.05   |

## analysis

the results shows that

1. the rate limit works, for both ingress and egress
2. the latency is not good, head-of-blocking and bufferbloat occurs
3. the ingress rate limit case causes latency much higher, even higher than that of egress rate limit case 

 its tc configuration is:

```
root@kind-worker2:/# tc qdisc show dev cali7af7e010ab7
qdisc tbf 1: root refcnt 2 rate 100Mbit burst 512Mb lat 25ms
qdisc ingress ffff: parent ffff:fff1 ----------------

root@kind-worker2:/# tc filter show dev cali7af7e010ab7 ingress
filter parent ffff: protocol all pref 1 u32 chain 0
filter parent ffff: protocol all pref 1 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol all pref 1 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:1 not_in_hw
  match 00000000/00000000 at 0
	action order 1: mirred (Egress Redirect to device bwp88b5a9f3c953) stolen
	index 1 ref 1 bind 1

	action order 2: mirred (Egress Redirect to device bwp88b5a9f3c953) pass
	index 2 ref 1 bind 1
```

This is a topology overview:

The latency is not good is mainly because the deep queue for token bucket rate limiter, [the latency is set 25ms](https://github.com/containernetworking/plugins/blob/main/plugins/meta/bandwidth/ifb_creator.go#L27) by default and we cannot configure it. 

The latency for ingress case is worse maybe due to no backpressure mechanism like TSQ, and causes slower feedback.

Because of two points above, 

1. for egress, I suggest making the queue depth a configurable value. adding a annotation for it. Actually, the whole pod traffic sharing the same queue is not good anyway, we need fine-grained qos. control the queue depth can make it more flexible for users.
2. for ingress, i suggest making it zero queue(policing) to bring quick feedback to sender. 

