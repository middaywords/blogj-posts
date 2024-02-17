---
title: procfs io - task_io_accounting
date: 2024-01-04 08:48:32
tags:
- linux
---

# procfs io - task io accounting

最开始是因为需要做磁盘 IO 统计，所以看了下相关的实现

## procfs/pid/io - task io accounting

参考内核文档 [The /proc Filesystem — The Linux Kernel documentation](https://docs.kernel.org/filesystems/proc.html)

```sh
root@hrt-workstation-n2shz:/var/home/centos# cat /proc/234225/io
rchar: 35310718
wchar: 1514464
syscr: 23705
syscw: 16502
read_bytes: 0
write_bytes: 327680
cancelled_write_bytes: 131072
```

* 指标相关说明

| metric                | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| Rchar                 | chars read.The number of bytes which this task has caused to be read from storage. This is simply the sum of bytes which this process passed to read() and pread(). It includes things like tty IO and it is unaffected by whether or not actual physical disk IO was required (the read might have been satisfied from pagecache). |
| wchar                 | chars written. The number of bytes which this task has caused, or shall cause to be written to disk. Similar caveats apply here as with rchar. |
| syscr                 | read syscalls. Attempt to count the number of read I/O operations, i.e. syscalls like read() and pread(). |
| syscw                 | write syscalls. Attempt to count the number of write I/O operations, i.e. syscalls like write() and pwrite(). |
| read_bytes            | bytes read. Attempt to count the number of bytes which this process really did cause to be fetched from the storage layer. Done at the [`submit_bio()`](https://docs.kernel.org/core-api/kernel-api.html#c.submit_bio) level, so it is accurate for block-backed filesystems. |
| Write_bytes           | bytes written. Attempt to count the number of bytes which this process caused to be sent to the storage layer. This is done at page-dirtying time. |
| cancelled_write_bytes | 这里最大的不准确之处是截断(truncate)。 如果一个进程向一个文件写入 1MB，然后删除该文件，它实际上不会执行任何写操作。 但它会被视为导致了 1MB 的写入。 换句话说：该进程通过截断 page cache 而导致未发生的字节数。 任务也可能导致“负”IO。 如果此任务截断了一些 dirty pagecache，则另一个任务已占的某些 IO（在其 write_bytes 中）将不会发生。 我们可以从截断任务的 write_bytes 中减去该值，但这样做会丢失信息。 |

* 具体实现

Procfs 相关实现在 `/proc` 中可以找到

```c
// fs/proc/base.c

#ifdef CONFIG_TASK_IO_ACCOUNTING
static int do_io_accounting(struct task_struct *task, struct seq_file *m, int whole)
{
	struct task_io_accounting acct = task->ioac;
    // ...
	if (whole && lock_task_sighand(task, &flags)) {
		struct task_struct *t = task;
        // 计入当前 thread 的 ioac
		task_io_accounting_add(&acct, &task->signal->ioac);
        // 累加每个 thread 的 ioac
		while_each_thread(task, t)
			task_io_accounting_add(&acct, &t->ioac);

		unlock_task_sighand(task, &flags);
	}
    // ...

out_unlock:
	up_read(&task->signal->exec_update_lock);
	return result;
}
```

而关于 `struct task_io_account` 相关实现，都在 `include/linux/task_io_accounting_ops.h` 中。

文件中有这些接口

```c
// include/linux/task_io_accounting_ops.h
#ifdef CONFIG_TASK_IO_ACCOUNTING
static inline void task_io_account_read(size_t bytes)
static inline unsigned long task_io_get_inblock(const struct task_struct *p)
static inline void task_io_account_write(size_t bytes)
static inline unsigned long task_io_get_oublock(const struct task_struct *p)
static inline void task_io_account_cancelled_write(size_t bytes)
static inline void task_io_accounting_init(struct task_io_accounting *ioac)
static inline void task_blk_io_accounting_add(struct task_io_accounting *dst,
						struct task_io_accounting *src)
#endif /* CONFIG_TASK_IO_ACCOUNTING */

#ifdef 
```



## procfs 中还有 diskstats

```
root@hrt-workstation-n2shz:/var/home/centos# cat /proc/diskstats
   7       0 loop0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       1 loop1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       2 loop2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       3 loop3 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       4 loop4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       5 loop5 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       6 loop6 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       7 loop7 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 252       0 vda 362394 2046 29505584 96823 1272845 367709 52724579 950876 0 478856 1067789 0 0 0 0 130852 20090
 252       1 vda1 91 0 728 6 0 0 0 0 0 32 6 0 0 0 0 0 0
 252       2 vda2 651 0 16816 74 2 0 4096 1 0 116 76 0 0 0 0 0 0
 252       3 vda3 361551 2046 29483632 96733 1272843 367709 52720483 950874 0 478784 1047608 0 0 0 0 0 0
 252      16 vdb 124488 755 7518107 29803 401035 119516 20334987 262445 0 255524 302282 0 0 0 0 91364 10033
 252      32 vdc 294 134 11653 16 0 0 0 0 0 76 16 0 0 0 0 0 0
 253       0 dm-0 488412 0 36975221 126252 2051613 0 73429842 1701460 0 544160 1827712 0 0 0 0 0 0
 253       1 dm-1 286 0 13223 36 1 0 4096 0 0 72 36 0 0 0 0 0 0
```

 可以参考 [/proc/diskstats各列含义介绍-CSDN博客](https://blog.csdn.net/u011436427/article/details/103394715) 和 [kernel.org/doc/Documentation/ABI/testing/procfs-diskstats](https://www.kernel.org/doc/Documentation/ABI/testing/procfs-diskstats) 

metrics 说明

## iostat - /sys/block/

```shell
root@hrt-workstation-n2shz:/var/home/centos# iostat
Linux 5.15.0-26-generic (hrt-workstation-n2shz) 	01/03/24 	_x86_64_	(12 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.17    0.00    0.06    0.00    0.02   99.75

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0              4.52        32.82        65.33         0.00   18521014   36865913          0
dm-1              0.00         0.01         0.00         0.00       6611       2048          0
vda               2.91        26.20        46.92         0.00   14784244   26480923          0
vdb               0.93         6.66        18.07         0.00    3761005   10199350          0
vdc               0.00         0.01         0.00         0.00       5826          0          0
```





## Cgroup blkio stats

[Block IO Controller — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v1/blkio-controller.html)

[Control Group v2 — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html#io)



## reference

