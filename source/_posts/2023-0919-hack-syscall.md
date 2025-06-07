---
title: hack syscall
date: 2023-09-19 12:12:23
tags:
- kernel
---

## hack syscall

weekly homework....

### Task1: add a new syscall to count processes grouped by state

env: x86 intel, VirtualBox Ubuntu 22.04 LTS

>  Add a new system call to loop all the processes and count the process group by the process state. For example, how many D-State process? How many Running-State process?

1. Write a document for your new system call including name, arguments, return value. we have `proc_count_t` defined in `include/uapi/linux/proc_count.h`

```c
struct proc_count_t {
    int task_D; // TASK_UNINTERRUPTIBLE
    int task_I; // TASK_IDLE
    int task_R; // TASK_RUNNING
    int task_T; // TASK_STOPPED
    int task_t; // TASK_TRACED
    int task_S; // TASK_INTERRUPTIBLE(sleeping)
    int task_Z; // EXIT_ZOMBIE
};

asmlinkage long sys_proc_count(struct proc_count_t __user *task_count);
```

* `vi arch/x86/entry/syscalls/syscall_64.tbl`

```
445	common	landlock_add_rule	sys_landlock_add_rule
446	common	landlock_restrict_self	sys_landlock_restrict_self
447	common	memfd_secret		sys_memfd_secret
448	common	process_mrelease	sys_process_mrelease
449     64  proc_count          sys_proc_count
```

* `vi include/linux/syscalls.h`

```c
asmlinkage long sys_proc_count(struct proc_count_t __user *task_count);
```

* `vi kernel/sys.c`

```c
SYSCALL_DEFINE1(proc_count, struct proc_count_t __user *, task_count)
{
        struct proc_count_t count = {0};
        struct task_struct *p;
        printk(KERN_EMERG "proc_count syscall triggered to count procs of state\n");
        unsigned int state = 0;

        // Only root user can access it, 0 is root user id
        if (__kuid_val(current_uid()) != (uid_t) 0) {
                return -EPERM;
        }

        for_each_process(p) {
                state = READ_ONCE(p->__state);
                if (state == TASK_RUNNING) {
                        count.task_R++;
                }
                if (state & __TASK_TRACED != 0 ) {
                        count.task_t++;
                }
                if (state & __TASK_STOPPED != 0) {
                        count.task_T++;
                }
            
                switch (state) {
                case TASK_UNINTERRUPTIBLE:
                        count.task_D++;
                        break;
                case TASK_INTERRUPTIBLE:
                        count.task_S++;
                        break;
                case TASK_IDLE:
                        count.task_I++;
                        break;
                }

                if (READ_ONCE(p->exit_state) & EXIT_ZOMBIE != 0) {
                        count.task_Z++;
                }
        }

        printk(KERN_EMERG "proc_count syscall: R:%d, D:%d, I:%d, T:%d, t:%d\n", count.task_R, count.task_D, count.task_I, count.task_T, count.task_t);

        if (copy_to_user(task_count, &count, sizeof(count)))
                return -EFAULT;

        return 0;
}
```

* Add source code and recompile the kernel.

```shell
> make menuconfig
> make
> make install
> make modules_install 
> reboot
```

3. Boot the system with new kernel.

4. Write a user-space program to test your system call

```c
#include <unistd.h>
#include <stdio.h>
#include "proc_count.h"

#define __NR_proc_count 449

int main(){
        struct proc_count_t count;
        long ret;
        ret = syscall(__NR_proc_count, &count);

        printf("Proc count: ret: %ld\n", ret);
        printf("R: %d, I: %d, D: %d, S: %d, Z: %d\n", count.task_R, count.task_I, count.task_D, count.task_S, count.task_Z);
        return 0;
}
```

Called `0f 05`: fast system call [SYSCALL — Fast System Call (felixcloutier.com)](https://www.felixcloutier.com/x86/syscall.html)  

```
0000000000401775 <main>:
//...
  401797:	bf c1 01 00 00       	mov    $0x1c1,%edi // 0x1c1=449 
  40179c:	b8 00 00 00 00       	mov    $0x0,%eax
  4017a1:	e8 2a e7 04 00       	call   44fed0 <syscall>

000000000044fed0 <syscall>:
// ...
  44fed0:	f3 0f 1e fa          	endbr64
  44fed4:	48 89 f8             	mov    %rdi,%rax
  44fed7:	48 89 f7             	mov    %rsi,%rdi
  44feda:	48 89 d6             	mov    %rdx,%rsi
  44fedd:	48 89 ca             	mov    %rcx,%rdx
  44fee0:	4d 89 c2             	mov    %r8,%r10
  44fee3:	4d 89 c8             	mov    %r9,%r8
  44fee6:	4c 8b 4c 24 08       	mov    0x8(%rsp),%r9
  44feeb:	0f 05                	syscall // int 0x80
  44feed:	48 3d 01 f0 ff ff    	cmp    $0xfffffffffffff001,%rax
  44fef3:	73 01                	jae    44fef6 <syscall+0x26>
  44fef5:	c3                   	ret
  44fef6:	48 c7 c1 b8 ff ff ff 	mov    $0xffffffffffffffb8,%rcx
  44fefd:	f7 d8                	neg    %eax
  44feff:	64 89 01             	mov    %eax,%fs:(%rcx)
  44ff02:	48 83 c8 ff          	or     $0xffffffffffffffff,%rax
  44ff06:	c3                   	ret
  44ff07:	66 0f 1f 84 00 00 00 	nopw   0x0(%rax,%rax,1)
  44ff0e:	00 00
```

5. Only the root user has the permission to call it.

```shell
kanxu@ubuntu2204:~/Documents/test_syscall$ ./a.out
Proc count: ret: -1 #(EPERM = -1)
R: 0, I: 0, D: 0, S: 0, Z: 0
root@ubuntu2204:/home/kanxu/Documents/test_syscall# ./a.out
Proc count: ret: 0
```



### Task2: open question

Can we add a system call with kernel module instead of recompiling the kernel? 

> https://unix.stackexchange.com/a/48208 (in 2012)
>
> It is **not possible** because system call table (called `sys_call_table`) is a **static size array**. And its size is determined at compile time by the number of registered syscalls. This means there is no space for another one.
>
> You can check implementation for example for x86 architecture in `arch/x86/kernel/syscall_64.c` file, where `sys_call_table` is defined. Its size is exactly `__NR_syscall_max+1`. `__NR_syscall_max` is defined in `arch/x86/kernel/asm-offsets_64.c` as `sizeof(syscalls) - 1` (it's the number of last syscall), where `syscall` is a table with all the syscalls.
>
> Doing this from kernel module is not trivial as kernel does not export `sys_call_table` to modules as of version 2.6 (the last kernel version that had this symbol exported was `2.5.41`).
>
> One way to work around this is to change your kernel to export `sys_call_table` symbol to modules. To do this, you have to add following two lines to `kernel/kallsyms.c` (**don't do this on production machines**):
>
> ```
> extern void *sys_call_table;
> EXPORT_SYMBOL(sys_call_table);
> ```

* what i found in my machine: symbol sys_call_table is not exported

```shell
root@ubuntu2204:/home/kanxu/Documents/test_syscall# cat /proc/kallsyms | grep sys_call_table
ffffffff97200300 D sys_call_table
ffffffff97201420 D ia32_sys_call_table
ffffffff97202240 D x32_sys_call_table
```

> `man nm`  explains what `D` is.
>
> "D"/"d" The symbol is in the initialized data section.
>
> "R"/ "r" The symbol is in a read only data section.

* and there is space for a new one(e.g. 335)

```shell
cat /usr/src/linux-headers-5.19.0-50-generic/arch/x86/include/generated/uapi/asm/unistd_64.h | grep '__NR'
#define __NR_io_pgetevents 333
#define __NR_rseq 334
#define __NR_pidfd_send_signal 424
#define __NR_io_uring_setup 425
#define __NR_io_uring_enter 426
#define __NR_io_uring_register 427
```

```c
// arch/x86/um/sys_call_table_64.c
const sys_call_ptr_t sys_call_table[] ____cacheline_aligned = {
#include <asm/syscalls_64.h>
};
int syscall_table_size = sizeof(sys_call_table);
```

items of syscall table is defined in ` ./arch/x86/include/generated/asm/syscalls_64.h`

```c
__SYSCALL(335, sys_ni_syscall) // not implemented syscall
```



* it's possible to add syscall with kernel module

 the other post in lkmpg, syscall section [GitHub - sysprog21/lkmpg: The Linux Kernel Module Programming Guide (updated for 5.0+ kernels)](https://github.com/sysprog21/lkmpg)

we replace the entry for syscall 335 in `sys_call_table`.

```c
#define __NR_mysyscall 335

static asmlinkage long my_syscall(void)
{
    long ret = 123;
    printk("Our syscall is successful");

    return ret;
}

/*
 * Non-implemented system calls get redirected here.
 */
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
```

we have a test program for it:

```c
#include <syscall.h>
#include <unistd.h>
#include <stdio.h>

int main(void)
{
	printf("%ld\n", syscall(335));
	return 0;
}
```

after loading the module to kernel, we can see that the new syscall works

```shell
kanxu@ubuntu2204:~/Documents/test_syscall$ ./mod_test
-1
kanxu@ubuntu2204:~/Documents/test_syscall$ sudo insmod syscall.ko
kanxu@ubuntu2204:~/Documents/test_syscall$ ./mod_test
123
```



### reference 

1. [Adding a New System Call — The Linux Kernel documentation](https://www.kernel.org/doc/html/v4.10/process/adding-syscalls.html) 
2. [lkmpg/examples/syscall.c at master · sysprog21/lkmpg · GitHub]
