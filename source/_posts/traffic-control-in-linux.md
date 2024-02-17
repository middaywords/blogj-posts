---
title: traffic control in linux
date: 2023-12-02 19:51:52
tags:
- network
- linux
- kernel
---

# traffic control in linux

这一节介绍 linux  TC 机制，主要参考 

1. TCP IP arch chapter 15, IP QoS
2. 以及 Linux  advanced routing & traffic control

简单说，QoS 做的事情是决定 packet

* 接受的包：以何种顺序(order)，以何种速度 (bandwidth rate) 被接收
* 发送的包：如何安排在队列，如何限制速度。

## basic components

* queuing discipline

  每个网络设备都有一个 queuing discipline，控制网络包发送之前，如何排队以及出队。

* classes

  对于 class based 的 queuing discipline，我们可以将网络包格局 filter(IP, TCP/IP port. etc) 来分类到各个的 classes。而 每个 calss 又会根据 priority 来让 packet 出队。

* filters

  filter 将 packet 划分到不同的 classes，基于其参数 (IP, TCP/IP port. etc) 。

* policing

  在网络包入队之后，包可能会被放行，被丢弃，打上标记

## pfifo_fast qdisc

Pfifo 有三个 band，0/1/2，当 0 有数据的时候，先发送 0，不会发 1 和 2。

pfifo 根据 packet 的 TOS 来决定其放到那个 band。TOS 共 4 个 bits，

| Binary (TOS bits) | Decimal | Meanings               |
| ----------------- | ------- | ---------------------- |
| 1000              | 8       | Minimize delay         |
| 0100              | 4       | Maximize throughput    |
| 0010              | 2       | Maximize realiability  |
| 0001              | 1       | Minimize monetary cost |
| 0000              | 0       | Normal service         |

RFC 1349 introduced an additional "lowcost" field. The four available ToS bits now becomes:

|        7        |  6   |  5   |    4     |     3      |      2      |         1          |       0        |
| :-------------: | :--: | :--: | :------: | :--------: | :---------: | :----------------: | :------------: |
| (IP Precedence) |      |      | lowdelay | throughput | reliability | lowcost (RFC 1349) | (Must be zero) |

不过根据 wiki 的说法，后面整个字段的定义又更改了， RFC 2474 规定 前 6 个 bit 用于 Differentiated Services Code Point (DSCP)，后面两个 bit 用于 ECN。

|  7   |  6   |  5   |  4   |  3   |  2   |  1   |  0   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| DSCP |      |      |      |      |      | ECN  |      |

## qdisc data structure

Qdisc 表示，一个流控规则，它绑定到一个 net device。

enqueue 有个处理函数，dequeue 有个处理函数。

每个 qdisc 有个 handle，是个 32 bit 的 id 标识。

## filters

Filters 需要给所有 incoming packets 分配一个 qdisc，基于 IP/port 等



## How 
