---
title: ovs 源码笔记
date: 2024-09-09 13:23:25
tags:
- linux
- network
---

# ovs 源码笔记

![img](../figures/v2-d24e15bc8f9855128f22a456d3e1c935_1440w.webp)

## 概览

* dataplane： 以用户态的 ovs-vswitchd 和内核态的 datapath 为主的转发模块，以及与之相关联的数据库模块 ovsdb-server。
* control plane: 由ovs-ofctl模块负责，基于OpenFlow协议与数据面进行交互。
* management plane: 由 OVS 提供的各种工具来负责，这些工具的提供也是为了方便用户对底层各个模块的控制管理，提高用户体验。

主要组件：

* Ovs-switchd: 与内核 datapath 共同组成数据面，用 Openflow 协议与控制器通信，用 OVSDB 与 ovsdb-server 通信
* ovsdb-server: OVS 轻量级数据库服务，配置 OVS，包括接口，VLAN 等
* OpenFlow 控制器（OVN）通过 OpenFlow 协议连接到任何支持 OpenFlow 的交换机，通过向交换机下发 OpenFlow 规则，来控制数据流向
* Kernel datapath 内核模块和 ovs-vswitchd 相互写作，datapath 负责收发包，而ovs-vswitchd通过下发的流表规则指导datapath如何转发包。
* ovs-ofctl是控制面的模块，但本质上它也是一个管理工具，主要是基于OpenFlow协议对OpenFlow交换机进行监控和管理，通过它可以显示一个OpenFlow交换机的当前状态，包括功能、配置和表中的项。
* [ovs-dpctl](https://zhida.zhihu.com/search?q=ovs-dpctl&zhida_source=entity&is_preview=1)用来配置交换机的内核模块datapath，它可以创建，修改和删除datapath，一般单个机器上的datapath有256条（0-255）。一条datapath对应一个虚拟网络设备。该工具还可以统计每条datapath上的设备通过的流量，打印流的信息等。
* ovs-appctl查询和控制运行中的OVS守护进程，包括ovs-switchd，datapath，OpenFlow控制器等，兼具ovs-ofctl、ovs-dpctl的功能，是一个非常强大的命令。ovs-vswitchd等进程启动之后就以一个[守护进程](https://zhida.zhihu.com/search?q=守护进程&zhida_source=entity&is_preview=1)的形式运行，为了能够很好的让用户控制这些进程，就有了这个命令。
* ovs-appctl查询和控制运行中的OVS守护进程，包括ovs-switchd，datapath，OpenFlow控制器等，兼具ovs-ofctl、ovs-dpctl的功能，是一个非常强大的命令。ovs-vswitchd等进程启动之后就以一个[守护进程](https://zhida.zhihu.com/search?q=守护进程&zhida_source=entity&is_preview=1)的形式运行，为了能够很好的让用户控制这些进程，就有了这个命令。

## ovs 源码

### vswitchd 模块

![img](https://picx.zhimg.com/80/v2-eb3ab7d8cb98fbd180360f1f1f6b648f_1440w.webp)

 在ovs中交换机和桥是一个东西, ovs用两个数据结构描述它们: 交换机(ofproto)与桥(bridge)，它们在ovs中是一一对应的。bridge更贴近用户, ofproto跟底层联系更多。ovs用port和ofprot描述端口, 前者对应bridge, 而后者对应ofproto。ovs还有一个结构叫[iface](https://zhida.zhihu.com/search?q=iface&zhida_source=entity&is_preview=1), 一般而言, 一个port包含一个iface, 但存在一种聚合(bond)的情况，此时port和iface就是一对多的关系了。

![img](https://pic4.zhimg.com/80/v2-472b1676cdc03278c2ab9d66b7c7fae5_1440w.webp)





## Reference

https://zhuanlan.zhihu.com/p/637977332
