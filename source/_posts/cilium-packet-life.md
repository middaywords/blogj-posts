---
title: cilium packet life
date: 2023-08-11 15:25:54
tags:
- bpf
- network
- cilium
---

## cilium packet life

[TOC]



Most content of this post is borrowed from https://arthurchiao.art/blog/cilium-life-of-a-packet-pod-to-service/#introduction,  and I refer to cilium 1.14 and update some notes. Thanks for [ARTHURCHIAO](https://arthurchiao.github.io/)'s  great guide.

cilium version: v1.14



![network topology inside cilium-powered k8s env](https://arthurchiao.art/assets/img/cilium-life-of-a-packet/network-topology.png)

As can be seen from the network topology figure above, network devices now seems to be “disconnected from each other”, there are no bridging devices or normal forwarding/routing rules that connect them. If capturing the traffic with `tcpdump`, you may see that the traffic disappears somewhere, then suddenly spring out in another place. **It confuses people how the traffic is transferred between the devices**, and to the best we could guess is that it is done by BPF.

Here we investigate the traffic flow from pod 1 on node 1 to a service-IP

### overview

Request packet journel: 

![taffic path of Pod-to-ServiceIP](https://arthurchiao.art/assets/img/cilium-life-of-a-packet/pod-to-service-path.png)

Reply packet journey: 

![reply packet journey](https://arthurchiao.art/assets/img/cilium-life-of-a-packet/round-trip-path.png)

### node 1 lxc device: pod 1 egress BPF procesing

```c
__section("from-container")
cil_from_container
  - bpf_clear_meta
  - reset_queue_mapping
  - validate_ethertype
  - switch(eth_proto_type) // take IPV4 for example here
    - ep_tail_call(ctx, CILIUM_CALL_IPV4_FROM_LXC)
    	- __tail_handle_ipv4
    		- __per_packet_lb_svc_xlate_4
    			- lb4_lookup_service
    			- lb4_local
    			- ep_tail_call(ctx, CILIUM_CALL_IPV4_CT_EGRESS);
						- CILIUM_CALL_IPV4_FROM_LXC_CONT
              - handle_ipv4_from_lxc
              	- ipv4_l3
              - tail_call_static(ctx, &CUSTOM_CALLS_MAP, CUSTOM_CALLS_IDX_IPV4_EGRESS);
```

* validate_ethertype(): check whether the skb is supported L2 packet, get l3 proto type
* tail call handler depending on the eth proto info
  * IPV4: goto bpf prog id=CILIUM_CALL_IPV4_FROM_LXC
  * __tail_handle_ipv4
    * **Service load balancing**: select a proper Pod from backend list, we assume POD4 on NODE2 is selected.
    * Create or update **connection tracking** (CT or conntrack) record.
    * **Perform DNAT**, replace ServiceIP with `POD4_IP` for the `dst_ip` field in IP header.
    * Perform egress **network policy checking**.
    * **Perform encapsulation if in tunnel mode, or pass the packet to kernel stack if in direct routing mode.**, we will see the latter one.



### node 1 bond0 device: egress BPF processing

```c
__section("to-netdev")
to_netdev
	- validate_ethertype
  - policy_clear_mark
  - switch (proto) // IPV4 for example
    - handle_to_netdev_ipv4
    	- src_id = resolve_srcid_ipv4
    		- lookup_ip4_remote_endpoint
    			- ipcache_lookup4
    	- ipv4_host_policy_egress(src_id) // bpf/lib/host_filewall.h
    		- ct_lookup4
    		- lookup_ip4_remote_endpoint
    		- policy_can_egress4 // Perform policy lookup
    		- ct_create4 // Only create CT entry for accepted connections
    		- send_policy_verdict_notify
    
```

* ipv4_host_policy_egress: We need to pass the srcid from ipcache to host firewall.
  *  ct_lookup4： Lookup connection in conntrack map
  * lookup_ip4_remote_endpoint： Retrieve destination identity.
  * policy_can_egress4: Perform policy lookup
  * ct_create4: Only create CT entry for accepted connections 
  * send_policy_verdict_notify:  Emit verdict if drop or if allow for CT_NEW or CT_REOPENED.



**BPF programs on native devices are mainly used for North-South traffic** processing, namely, the external (in & out k8s cluster) traffic [3]. This includes,

- Traffic of LoadBalancer Services
- Traffic of ClusterIP Services with externalIPs
- Traffic of NodePort Services

In the next, kernel will lookup routing table and ARP table, and encap the L2 header for the packet.

**Determine `src_mac` and `dst_mac`**

```
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.255.255.1    0.0.0.0         UG    0      0        0 bond0
10.1.1.0        10.1.1.1        255.255.255.0   UG    0      0        0 cilium_host
10.1.1.1        0.0.0.0         255.255.255.255 UH    0      0        0 cilium_host

$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
10.255.255.1             ether   00:00:5e:00:01:0c   C                     bond0
```

This packet will hit the default routing rule on the kernel routing table, so it determines,

- **`bond0`’s MAC to be `src_mac`**: MAC addresses are only meaningful inside a L2 network, the host and Pod are in different L2 networks (Cilium manages a distinct CIDR/network), so the host will set its own MAC address as `src_mac` when forwarding this packet.
- **The MAC of host’s gateway (`10.255.255.1`) to be `dst_mac`**: next hop is host’s gateway.

Then, the packet will be sent to the underlying data center network via bond0 (and physical NICS).

### data center routing

Data center network **routes packets based on `dst_ip`**.

As **NODE2 has already announced that `PodCIDR2` via BGP, and `POD4_IP` falls into `PodCIDR2`, so routers will pass this packet to NODE2**.

> Network Virtualization: cross-host networking.
>
> From network layer’s perspective, there are two kinds of cross-host networking schemes:
>
> 1. L2/LL2 (Large L2): run a **software switch or software bridge** inside each node, typical: OpenStack Neutron+OVS.
> 2. L3: run a **software router** inside each node (actually kernel itself is the router), each node is a layer 3 node, typical: Cilium+BGP[2].
>
> One big difference when trouble shooting:
>
> 1. In L2/LL2 network, **src_mac stays unchanged during the entire path**, so receiver sees the same src_mac as sender does; L2 forwarding only changes dst_mac;
> 2. In L3 network, both src_mac and dst_mac will be changed.
>
> Understanding this is important when you capture packets.



### node 2 bond: ingress BPF processing

```c
// kernel source tree, 4.19

ixgbe_poll
 |-ixgbe_clean_rx_irq
    |-if support XDP offload
    |    skb = ixgbe_run_xdp()
    |-skb = ixgbe_construct_skb()
    |-ixgbe_rx_skb
       |-napi_gro_receive
          |-napi_skb_finish(dev_gro_receive(skb))
             |-netif_receive_skb_internal
                |-if generic XDP
                |  |-if do_xdp_generic() != XDP_PASS
                |       return NET_RX_DROP
                |-__netif_receive_skb(skb)
                   |-__netif_receive_skb_one_core
                      |-__netif_receive_skb_core(&pt_prev)
                         |-for tap in taps:
                         |   deliver_skb
                         |-sch_handle_ingress                     // net/core/dev.c
                            |-tcf_classify                        // net/sched/cls_api.c
                               |-for tp in tps:
                                   tp->classify
                                       |-cls_bpf_classify         // net/sched/cls_bpf.c
```

Main steps:

1. NIC received a packet.
2. Execute XDP programs if there are XDP programs and NIC supports XDP offload (not our case here).
3. Create `skb`.
4. Do GRO, assemble fragemented packets.
5. Generic XDP processing: if NIC doesn’t support XDP offload, then XDP programs will be delayed to execute here from step 2.
6. Tap processing (not our case here).
7. TC ingress processing, execute TC programs, and tc BPF program is one kind.

Check the BPF program loaded at tc ingress hook:

```shell
$ tc filter show dev bond0 ingress
filter protocol all pref 1 bpf
filter protocol all pref 1 bpf handle 0x1 bpf_netdev_bond0.o:[from-netdev] direct-action not_in_hw tag 75f5
```

This piece of BPF will process **the traffic coming into bond0 via physical NICs**.

```c
__section("from-netdev")
cil_from_netdev
  - if allow_vlan
    - CTX_ACT_OK or send_drop_notify_error
  - flags = ctx_get_xfer(ctx, XFER_FLAGS)
  - ctx_skip_nodeport_clear
  - if (flags & XFER_PKT_NO_SVC)
    - ctx_snat_done_set
  - if (flags & XFER_PKT_SNAT_DONE)
    - ctx_snat_done_set
  if (flags & XFER_PKT_ENCAP)
    - edt_set_aggregate
    - encap_and_redirect_with_nodeid_opt
  - decapsulate_overlay
  - handle_netdev(from_host=false)
    - validate_ethertype
    - do_netdev
    	- identity = resolve_srcid_ipv4() 
    	- ctx_store_meta(CB_SRC_IDENTITY, identity)
    	- if (from_host)
    		- ep_tail_call(ctx, CILIUM_CALL_IPV4_FROM_HOST)
    		- ep_tail_call(ctx, CILIUM_CALL_IPV4_FROM_NETDEV)

CILIUM_CALL_IPV4_FROM_NETDEV
- tail_handle_ipv4
	- handle_ipv4
		- lookup_ip4_endpoint
		- maybe_add_l2_hdr
		- ipv4_local_delivery
		
```

1. Call `handle_netdev()` to process **the packets that will enter Cilium-managed network from host**, specific things including,
   1. `do_netdev` Extract identity of this packet (Cilium relies on identity for policy enforcement), and save it to packet’s metadata.
      - `resolve_srcid_ipv4()` In direct routing mode, **lookup identity from ipcache** (ipcache syncs itself with cilium’s kvstore). 
      - In tunnel mode, identity is encapsulated in Geneve header, so no need to lookup ipcache.
   2. Tail call to `tail_handle_ipv4_from_netdev()`.
2. `tail_handle_ipv4_from_netdev()` calls `tail_handle_ipv4()`, the latter further calls `handle_ipv4()`. `handle_ipv4()` performs:
   1. **Determine the endpoint (POD4) that `dst_ip` relates to**.
   2. Call `ipv4_local_delivery()`, this method will **tail call to endpoint’s BPF for Pod egress processing**.  Performs IPv4 L2/L3 handling and delivers the packet to the destination pod, on the same node, either via the stack or via a redirect call.

### pod4 ingress BPF handling

Just as previous sections, let’s look at the egress hook of POD4’s `lxc00dd` (this corresponding to POD4’s ingress),

```
(NODE2) $ tc filter show dev lxc00dd egress
```

**Not any loaded BPF programs, why**？

It’s because in Cilium’s design (**performance optimization**), the Pod ingress program is not triggered to execute (normal way), but directly via tail call (short-cut) from `bond0`’s BPF: (`l3_local_delivery`in `bpf/lib/l3.h`)

```
    tail_call_dynamic(ctx, &POLICY_CALL_MAP, ep->lxc_id);
```

So the ingress BPF needs not to be loaded to `lxc00dd`,

The tail-call calls to `to-container` BPF. Call stack:

```c
__section("to-container")
handle_to_container                                            //    bpf/bpf_lxc.c
  - magic=inherit_identity_from_host(skb, &identity)           // -> bpf/lib/identity.h
  - if (magic == MARK_MAGIC_PROXY_EGRESS_EPID)
  		- tail_call_dynamic(ctx, &POLICY_EGRESSCALL_MAP, identity);
  - if (identity == HOST_ID)
  	- tail_call_static(ctx, &POLICY_CALL_MAP, HOST_EP_ID);
  - __lookup_ip4_endpoint
  - ep_tail_call(ctx, CILIUM_CALL_IPV4_CT_INGRESS);
  	- tail_ipv4_policy
      |-ipv4_policy                                            //    bpf/bpf_lxc.c
          |-policy_can_access_ingress                          //    bpf/lib/policy.h
              |-__policy_can_access                            //    bpf/lib/policy.h
                  |-if p = map_lookup_elem(l3l4_key); p     // L3+L4 policy
                  |    return TC_ACK_OK
                  |-if p = map_lookup_elem(l4only_key); p   // L4-Only policy
                  |    return TC_ACK_OK
                  |-if p = map_lookup_elem(l3only_key); p   // L3-Only policy
                  |    return TC_ACK_OK
                  |-if p = map_lookup_elem(allowall_key); p // Allow-all policy
                  |    return TC_ACK_OK
                  |-return DROP_POLICY;                     // DROP
```

Things done by this piece of BPF:

1. Extract src identity of this packet, actually this info is already in packet’s metadata.
2. Call `tail_ipv4_to_endpoint()`, which will further call `ipv4_policy()`, the latter performs POD4’s **ingress network policy checking**.

If the packet is not denied by network policy, it will be forwarded to `lxc00dd`’s peer end, namely, POD4’s virtual NIC `eth0`.



### arrive pod4's eth0

On arriving `eth0` of POD4, it could be processed by upper layers.

At last, there is one important thing that needs to be noted: **do not make any performance assumptions by comparing the number of hops between Cilium/eBPF and OpenStack/OVS topologies** as shown in this post, as “hop” in Cilium/eBPF is a different concept in this post, mainly used for illustrating the processing steps, and it is not comparable with a traditional “hop”. For example, Step 6 to Step 7 is just a function call, it costs almost nothing in terms of forwarding benchmarks.



## TODO 

investigate receive path

![reply packet journey](https://arthurchiao.art/assets/img/cilium-life-of-a-packet/round-trip-path.png)



## reference 

Reference:

1. https://arthurchiao.art/blog/cilium-life-of-a-packet-pod-to-service/
2. [Using BIRD to run BGP — Cilium 1.8.3 documentation](https://docs.cilium.io/en/v1.8/gettingstarted/bird/)



