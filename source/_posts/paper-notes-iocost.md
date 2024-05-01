---
title: paper-notes iocost[ASPLOS22]
date: 2024-02-21 21:44:24
tags:
- linux
- storage
- paper
---

# [ASPLOS22] iocost: Block IO Control for Containers in Datacenters

## abstract

现在 block storage IO 的控制机制对于容器场景还是不够的，IO control 需要能够提供 按比例划分 的资源控制。需要考虑到不同硬件的异构，不同负载的特点。

现代 SSD 设备速度，要求 IO control 以低开销的方式运行，此外 IO control 需要做到 work conservation，考虑到和 memory 子系统的协同性，避免 priority inversion， 致使 isolation failure。

为了解决这些问题，作者提出了 iocost，一种为 容器场景定制的 IO control 解决方案，为了异构设备和各种应用负载提供了可扩展，低开销的控制方案。

IO cost 会执行 offline profiling 来构建 device model，用于估计每个 IO request 的 device occupancy。为了最小化 runtime 的开销，它将 IO control 分离为 fast per-IO issue path 和 slower periodic planning path。作者还提出了一种新的 work-conserving budget donation 的算法，使得 container 能够动态调整还没有使用的 budget。Meta 线上已经部署了 IO cost，并贡献到了 内核，而且也开源了一些 device profiling  tools。iocost 线上运行两年了，这篇文章介绍它的设计和 experience deploying it at scale。

## introduction

现有的 BFQ for block stroage 对于容器环境的隔离来说还不够。

有几个挑战

1. 需要考虑到硬件异构，不同 generation 的 SSD，spinning disks，本地/远程存储，等等可能共存于一个 data center。硬件异构则导致了 latency/throughput  的巨大性能差异。即使同一种 hard drive，同一种类型，都会有差异。一个设备可能一下子性能好，一下子性能又差，不稳定。
2. IO control 需要迎合不同的应用，比如有 latency sensitive 的应用，有些是大吞吐的，还有喜欢 seqential access 或者 random access，有些 pattern 是 burst，有些是 pacing 细水长流。但是找一个 latency 和 throughput 是很困难的。
3. IO isolation 也需要提供一些列特性，如 work conservation。work conservation 表示资源不够用的的时候，能保留下来，等资源空闲了再用，
