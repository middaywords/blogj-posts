---
title: OVN control path and OVN datapath(stt tunnel)
date: 2024-11-01 16:45:29
tags:
- linux
- network
- ovn
---

## ovn data path

## ovn control path

## ovn cheatsheet

find openflow configuration from south to north:

* In north bound db, it has mac addr and VM(logical network information)

```
root:# ovn-nbctl show
switch be21f5e0-03de-41e6-8521-d1a85303c10f (sw-10.84.0.0-22)
    port e990cd90-6337-407b-b820-d6bd502b1294
        addresses: ["74:db:d1:3a:b1:97 10.84.2.207"]
    port 478e96b2-5aa6-459a-a1e5-bc58dec7b876
        addresses: ["74:db:d1:7f:cc:d1 10.84.2.19"]
    port 19a5a0ca-d180-42dd-8649-95881c2b0e98
        addresses: ["74:db:d1:0b:99:43 10.84.2.243"]
```

`10.84.2.207` is IP of a VM node on OVN hypervisor.



* In south bound db, it shows info about Hypervisor. It's about physical network information. 

```
root:# ovn-sbctl show | head
Chassis "4479d714-d827-4717-a045-cb7ef457ebcf"
    hostname: z7zqh-node
    Encap stt
        ip: "10.179.206.43"
        options: {csum="true"}
    Port_Binding "d0943c4d-fc57-4d25-afc8-bfeb0b75e30c"
    Port_Binding "15b0fb0a-0605-4244-bb8e-ba3393639e53"
```

the Hypervisior is Chassis "4479d714-d827-4717-a045-cb7ef457ebcf", it will use `stt` protocol to encapsulate packets, it has two ports binded on this Hypervisor. One Port "04217bd3-ce16-449e-8fed-abb2a4815c30" binding to the hypervisor. And it's connected to a VM on this Hypervisor.

```
root@:# ovn-nbctl --no-leader-only show | grep d0943c4d-fc57-4d25-afc8-bfeb0b75e30c -C 3
switch e18980e5-d2da-40de-85cc-0fbbfefadba1 (sw-10.9.0.0-22)
...
	port d0943c4d-fc57-4d25-afc8-bfeb0b75e30c
        addresses: ["74:db:d1:5e:64:d4 10.9.3.189"]
...
```

And the port connected to VM, has MAC addr `d0943c4d-fc57-4d25-afc8-bfeb0b75e30c` and IP addr `10.9.3.189`

* in hypervisor ovs info, we can see some port/addr information

```
# ovs-vsctl show | grep br-int -A 100
		Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port ovn-a45bf0-1
            Interface ovn-a45bf0-1
                type: stt
                options: {csum="true", key=flow, local_ip="10.184.163.36", remote_ip="10.217.73.23"}
        Port ovn-842a08-1
            Interface ovn-842a08-1
                type: stt
                options: {csum="true", key=flow, local_ip="10.184.163.36", remote_ip="10.179.224.71"}
...
        Port br-int
            Interface br-int
                type: internal

```

local ip is local hypervisor IP, while remote ip is remote hypervisor IP.



## A case to understand pod(inside VM) routing

Architecture case:

![image-20241213213953585](../figures/image-20241213213953585.png)

these CIDR will be routed to VM 10.9.2.146.

```
# ovn-nbctl --no-leader-only lr-route-list dev-jr | grep 10.9.2.146 
           10.9.52.144/28                10.9.2.146 dst-ip
            10.9.143.0/28                10.9.2.146 dst-ip
           10.9.193.96/28                10.9.2.146 dst-ip
          10.9.202.192/28                10.9.2.146 dst-ip
           10.9.207.64/28                10.9.2.146 dst-ip
```

check the Logical Port binding, the port `670c5500-4083-4322-84d7-d457b04b605c` is binded to VM IP `10.9.2.146`

```
# ovn-nbctl --no-leader-only show  | grep 10.9.2.146 -C 3
    port 23f2d07b-d052-4d9c-ae1f-44927c2156ff
        addresses: ["74:db:d1:29:c6:a3 10.9.1.31"]
    port 670c5500-4083-4322-84d7-d457b04b605c
        addresses: ["74:db:d1:bf:44:5a 10.9.2.146"]
    port 93db7faa-c8f2-422f-b48e-7621cb41c9fd
        addresses: ["74:db:d1:d4:c7:24 10.9.3.109"]
    port a0ad4811-2ff8-42cd-baab-7793867038ed
```

Found the Hypervisor/Chassis, which has the port binded.

```
# ovn-sbctl --no-leader-only show | grep 670c5500-4083-4322-84d7-d457b04b605c -C 3
    Encap stt
        ip: "10.184.163.36"
        options: {csum="true"}
    Port_Binding "670c5500-4083-4322-84d7-d457b04b605c"

```

Check the openflow rule in Hypervisor, it says all packets going to `10.9.52.144/255`, will be routed to `10.96.133.20` Hypervisor.

```
# ovs-dpctl dump-flows | grep 10.9.52
recirc_id(0),in_port(5),eth(src=74:db:d1:2f:8b:ef,dst=74:db:d2:19:e7:6b),eth_type(0x0800),ipv4(src=10.32.0.0/255.224.0.0,dst=10.9.52.144/255.255.255.240,tos=0/0x3,ttl=63,frag=no), packets:5718, bytes:563542, used:0.713s, actions:set(tunnel(tun_id=0x10f3e000002,src=10.184.162.92,dst=10.96.133.20,ttl=64,tp_dst=7471,flags(df|csum|key))),set(eth(src=74:db:d2:19:e7:6b,dst=74:db:d1:30:9e:dd)),set(ipv4(ttl=62)),3
```

check the logical flow in southbound db, it says all packets going to 10.9.52.144/28 will be routed to logical IP `10.215.91.233`, which is the dst VM.

```
# ovn-sbctl dump-flows dev-jr | grep 10.9.52.144
  table=14(lr_in_ip_routing   ), priority=85   , match=(reg7 == 0 && ip4.dst == 10.9.52.144/28), action=(ip.ttl--; reg8[0..15] = 0; reg0 = 10.215.91.233; reg1 = 10.215.88.1; eth.src = 74:db:d2:19:e7:6b; outport = "lrp-dev-jr-to-sw-10.215.88.0-22"; flags.loopback = 1; next;)
```



The last two openflow rules info do not align with what's shown in ovn-northd info. 

from nbctl shown info, the CIDR `10.9.52.144/28` will be routed to VM `10.9.2.146`, which is on the hypervisor of IP  `10.184.163.36`

```
# ovn-nbctl --no-leader-only lr-route-list dev-jr | grep 10.9.2.146 
           10.9.52.144/28                10.9.2.146 dst-ip
```

However, from the openflow rule, we can see from the hypervisor rule, the packets going to `10.9.52.144` will be routed to HV `10.96.133.20`, while it is expected to be `10.184.163.36` from nb-db info.

from the sb dump flow results, in logical route aspect, the packets going to  `10.9.52.144` will be routed to VM IP `10.215.91.233`, while it is expected to be `10.9.2.146`.

## reference 

* https://zhuanlan.zhihu.com/p/666424141 it helps understanding from practice, really good post(though in Chinese)
* Cheatsheet: https://gist.github.com/odivlad/302d30e333ac2296850827bf4917913a 





