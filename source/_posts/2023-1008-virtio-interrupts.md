---
title: virtio interrupts and interrupt stack
date: 2023-10-08 15:26:35
tags:
- virtio
- kernel
---

# from virtio interrupts -> interrupt stack

[TOC]

事情是这样的，我最近了解了 `cat /proc/interrupts` ，然后看到了 virtio，然后我就看不懂这几个 interrupt 在干什么。然后就一路看中断相关的东西，然后又看到了 interrupt stack，很自然的是，我对栈也不太懂，于是又看了点栈的内容。

```shell
~ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       CPU8       CPU9       CPU10      CPU11
  1:          9          0          0          0          0          0          0          0          0          0          0          0   IO-APIC   1-edge      i8042
  4:          0          0       2051          0          0          0          0          0          0          0          0          0   IO-APIC   4-edge      ttyS0
  6:          0          0          0          3          0          0          0          0          0          0          0          0   IO-APIC   6-edge      floppy
  8:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC   8-edge      rtc0
  9:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC   9-fasteoi   acpi
 10:          0          1          0          0          0          0          0          0          0          0          0          0   IO-APIC  10-fasteoi   virtio4
 11:          0          0          0          0         35          0          0          0          0          0          0          0   IO-APIC  11-fasteoi   uhci_hcd:usb1
 12:          0          0          0          0          0         15          0          0          0          0          0          0   IO-APIC  12-edge      i8042
 14:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC  14-edge      ata_piix
 15:          0          0          0          0          0          0          0          0          0          0          0          0   IO-APIC  15-edge      ata_piix
 24:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 49152-edge      virtio0-config
 25:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 65536-edge      virtio1-config
 26:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 49153-edge      virtio0-control
 27:          0          0          0          0          0          0          0          0          0          0          0     878472   PCI-MSI 65537-edge      virtio1-req.0
 28:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 49154-edge      virtio0-event
 29:          0          0          0          0          0          0          0          0          0          0        256          0   PCI-MSI 49155-edge      virtio0-request
 30:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 81920-edge      virtio2-config
 31:          0          0          0          0          0          0          0          0          0     427428          0          0   PCI-MSI 81921-edge      virtio2-req.0
 32:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 98304-edge      virtio3-config
 33:          0          0          0          0          0          0          0          0        285          0          0          0   PCI-MSI 98305-edge      virtio3-req.0
 34:          0          0          0          0          0          0          0          0          0          0          0          0   PCI-MSI 131072-edge      virtio5-config
 35:          0          0          0   22420794          0          0          0        183          0          0          0          0   PCI-MSI 131073-edge      virtio5-input.0
 36:          0          0          0          0          0          0          0          0        182          0          0   22806773   PCI-MSI 131074-edge      virtio5-output.0
NMI:          0          0          0          0          0          0          0          0          0          0          0          0   Non-maskable interrupts
LOC:   12974758   19154234   19903314   19274621   26221500   18694717   19218334   19207180   21610316   20894393   20209977   18791782   Local timer interrupts
SPU:          0          0          0          0          0          0          0          0          0          0          0          0   Spurious interrupts
PMI:          0          0          0          0          0          0          0          0          0          0          0          0   Performance monitoring interrupts
IWI:          0          0          0          0          0          0          0          0          0          0          0          0   IRQ work interrupts
RTR:          0          0          0          0          0          0          0          0          0          0          0          0   APIC ICR read retries
RES:    4940564    5849255    5452737    3096593    4194589    5083848    3881623    3854364    3738440    3785632    3821414    3761403   Rescheduling interrupts
CAL:    2338191    2077012    1976472    1953296    1912769    1868499    2742395    2496043    2737752    2669259    2432048    2146795   Function call interrupts
TLB:     621007     622901     624885     574235     606242     595633     627439     611063     590904     602618     573677     589296   TLB shootdowns
TRM:          0          0          0          0          0          0          0          0          0          0          0          0   Thermal event interrupts
THR:          0          0          0          0          0          0          0          0          0          0          0          0   Threshold APIC interrupts
DFR:          0          0          0          0          0          0          0          0          0          0          0          0   Deferred Error APIC interrupts
MCE:          0          0          0          0          0          0          0          0          0          0          0          0   Machine check exceptions
MCP:       3957       3957       3957       3957       3957       3957       3957       3957       3957       3957       3957       3957   Machine check polls
ERR:          0
MIS:          0
PIN:          0          0          0          0          0          0          0          0          0          0          0          0   Posted-interrupt notification event
NPI:          0          0          0          0          0          0          0          0          0          0          0          0   Nested posted-interrupt event
PIW:          0          0          0          0          0          0          0          0          0          0          0          0   Posted-interrupt wakeup event
```

然后我参考了这篇文章：[Linux内核学习笔记之中断和中断处理 | 普通人 (hjk.life)](https://hjk.life/posts/linux-kernel-interrupt/)，他说中断号是 `request_irq` 分配的，然后我就去看相关的驱动的实现

```c
// drivers/block/virtio_blk.c
	for (i = 0; i < num_vqs - num_poll_vqs; i++) {
		callbacks[i] = virtblk_done;
		snprintf(vblk->vqs[i].name, VQ_NAME_LEN, "req.%d", i);
		names[i] = vblk->vqs[i].name;
	}

	for (; i < num_vqs; i++) {
		callbacks[i] = NULL;
		snprintf(vblk->vqs[i].name, VQ_NAME_LEN, "req_poll.%d", i);
		names[i] = vblk->vqs[i].name;
	}
```

```c
// drivers/net/virtio_net.c
	/* Parameters for control virtqueue, if any */
	if (vi->has_cvq) {
		callbacks[total_vqs - 1] = NULL;
		names[total_vqs - 1] = "control";
	}

	/* Allocate/initialize parameters for send/receive virtqueues */
	for (i = 0; i < vi->max_queue_pairs; i++) {
		callbacks[rxq2vq(i)] = skb_recv_done;
		callbacks[txq2vq(i)] = skb_xmit_done;
		sprintf(vi->rq[i].name, "input.%d", i);
		sprintf(vi->sq[i].name, "output.%d", i);
		names[rxq2vq(i)] = vi->rq[i].name;
		names[txq2vq(i)] = vi->sq[i].name;
		if (ctx)
			ctx[rxq2vq(i)] = true;
	}
```

然后我们可以根据中断名称，就可以确定是哪个设备了。

但是我在想，假如有两个 virtio-net 设备，怎么分得清哪个中断算哪个设备的呢？大概跑个 iperf 看看流量，不知道有没有直接看配置的方式。（然后 lshw 搞定了）

## Oct.10 cont

有个同事说可以用 lshw 看到很多信息，可以看到 eth0 是 virtio5，也可能看到 PCI， irq num 之类的

```shell
~ sudo lshw -class network
  *-network
       description: Ethernet controller
       product: Virtio network device
       vendor: Red Hat, Inc.
       physical id: 8
       bus info: pci@0000:00:08.0
       version: 00
       width: 64 bits
       clock: 33MHz
       capabilities: msix bus_master cap_list rom
       configuration: driver=virtio-pci latency=0
       resources: irq:11 ioport:c220(size=32) memory:feb95000-feb95fff memory:fe014000-fe017fff memory:feb00000-feb7ffff
     *-virtio5
          description: Ethernet interface
          physical id: 0
          bus info: virtio@5
          logical name: eth0
          serial: 74:db:d1:84:90:10
          capabilities: ethernet physical
          configuration: autonegotiation=off broadcast=yes driver=virtio_net driverversion=1.0.0 ip=10.147.1.7 link=yes multicast=yes
```



## APIC/MSIx/INTx

refer to [3]



## interrupt stack

refer to [2]



# reference 

1. [Linux内核学习笔记之中断和中断处理 | 普通人 (hjk.life)](https://hjk.life/posts/linux-kernel-interrupt/)
2. https://blog.csdn.net/yangkuanqaz85988/article/details/52403726 讲内核栈 中断栈的，讲的很好
3. [中断硬件之Intel架构 (betamao.me)](https://blog.betamao.me/posts/2021/ia-interrupt-hardware/) 
4. [White Paper: Introduction to Intel® Architecture, The Basics](https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/ia-introduction-basics-paper.pdf) 
