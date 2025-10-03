---
title: bandwidth estimation
date: 2023-10-23 23:37:44
tags:
---

# bandwidth estimation

Some terms:

| Term                   | Definition                                                   |
| ---------------------- | ------------------------------------------------------------ |
| Capacity               | The maximum rate at which packets can be transmitted by a link |
| Narrow link            | The link with the smallest capacity along a path             |
| Available bandwidth    | A link’s unused capacity                                     |
| Tight link             | The link with minimum available bandwidth along a path       |
| Bulk Transfer Capacity | operates at  the transport layer, gauging the amount of capacity available to an    application that wishes to send data. |

Model of available bandwidth (ABW)

* it is unused capacity.

* ought to look at the average unused bandwidth over some time interval T

* $$
  A_i(t, T) = \frac{1}{T}\int_{t}^{T+t}(C_i-\lambda_i(t))\ dt
  $$

  * A是在link i，t时刻的available bandwidth
  * lambda是traffic 大小

* 一条路径上的available bandwidth大小是所有链路上最小的那一条

Packet Pair 类方法

Packet pair是在router会采用fair queuing前提下使用的，但现在的Internet里面的路由器都不会采取fair queuing，所以不能在实际中使用。

Cprobe

它并不假设fair queuing，它不是发送 a pair of packets，而是发送一串ICMP包，通过ICMP ECHO的第一个和最后一个包的到达间隔来计算available bandwidth。 但是Dovrolis证明了这些只是测量了一个称为Asymptotic Dispersion Rate (ADR)的值，和available bandwidth相关但不相同。

之后的测量方法主要有两类：

* **The probe gap model (PGM)** ：通过接收端两次连续的probe的信息来计算，两个包发送间隔记为`delta_in`，接受的时间间隔记做`data_out`。假设只有一个bottleneck，在第一个包和第二个包到达该bottleneck link的时候，队列不为空，所以  `(delta_out-delta_in)` 就是传输其他cross traffic的时间，所以其他traffic的速率为`(delta_out-delta_in)/delta_in`，C是bottleneck link的capacity ，那么

  $$
  A = C \times(1- \frac{delta\_out - delta\_in}{delta\_in})
  $$

  例如IGI、Delphi都用的这种

* **The probe rate model (PRM)** 是基于self-induced congestion，如果发送速率低于available bandwidth，那么接收端的速率会和发送速率**match**，否则，queue会build up，probe会被delay，probe's rate会比发送速率低，因此可以通过找速率**match**的turning point来确定available bandwidth。

  Pathload，PathChirp，PTR，TOPP都是这种模型。

* 为了消除cross traffic的影响，两种模型都需要用 a train of probe packets to generate a single measurement.

* 两种模型都需要一些假设：

  * FIFO queuing at all routers along the path;
  * cross traffic follows a fluid model (i.e., non-probe packets have an infinitely small packet size);
  * cross taffic的速度在测量时不会突变 average rates of cross traffic change slowly and is constant for the duration of a single measurement.
  * Probe gap model (PGM)还需要多一个假设：the probe gap model assumes a single bottleneck which is both the narrow and tight link for that path.

* 还有许多工作不是直接估计available bandwidth。 

  * 比如Pathchar [10], bprobe [6], pchar [5], tailgating [15], nettimer [16], clink [8], pathrate [7] 是用来估计capacity
  * Treno [17], and cap [3] estimate the TCP fair rate along a path.

Spruce

* 假设：a single bottleneck is both narrow and tight link（尽管假设有时候不太符合，也还是准确的，他们的实验结果）
* Design：基于 probe gap model 做的改进
  * 输入: `delta_in`, `delta_out`, `capacity`
  * 为了在之前说的模型上改进，他们讲inter-gap time设置为服从指数分布（期望`tao`远大于`delta_in`），所以是一个柏松采样过程。这样设计有两个好处，一是对于single bottleneck，non-fluid cross traffic，使用柏松采样过程可以知道平均的cross traffic rate（？）。其二，柏松采样过程可以保证Spruce是non-intrusive，Spruce控制的时间间隔比较大，发送速率不会太快，不会对其他流造成太大影响，保证测量准确性。
* 他们也进行了实验对比，真值通过(Multi-Router Traffic Grapher (MRTG))来得到的，对比Pathload(PRM模型)和IGI(PRM和PGM结合)，他们有更高的精确度

SLoPS

* 通过改变sender的发送速率，在receiver上观测oneway delay的变化，若是one way delay一直上涨，说明当前发送速率超过available bandwidth，一直平稳说明小于available bandwidth

Reference

* **A Measurement Study of Available Bandwidth Estimation Tools** [IMC03]
* RFC 5136 https://tools.ietf.org/html/rfc5136#section-1
