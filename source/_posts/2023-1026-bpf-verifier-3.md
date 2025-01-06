---
title: bpf verifier (3)
date: 2023-10-26 22:42:51
tags:
- bpf
---

# bpf verifier (3)

## 如何理解 verifier logs

翻了好久源码，终于找到了，比如我们拿 `vfsstat.py` in bcc 作为例子来分析（为什么选这个呢？因为简单）

bpf 程序代码：

```c
static void stats_try_increment(int key) {
    stats.atomic_increment(key);
}

KFUNC_PROBE(vfs_write) {
    stats_try_increment(S_WRITE);
    return 0;
}
```

汇编代码：

```
root@ubuntu2204:/home/kanxu# bpftool prog dump xlated id 1570
int kfunc__vmlinux__vfs_write(unsigned long long * ctx):
; KFUNC_PROBE(vfs_write)        { stats_try_increment(S_WRITE); return 0; }
   0: (b7) r1 = 2
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
   1: (63) *(u32 *)(r10 -4) = r1
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
   2: (18) r1 = map[id:334]
   4: (bf) r2 = r10
;
   5: (07) r2 += -4
; return bpf_map_lookup_elem((void *)map, key);
   6: (07) r1 += 272
   7: (61) r0 = *(u32 *)(r2 +0)
   8: (35) if r0 >= 0x6 goto pc+3
   9: (67) r0 <<= 3
  10: (0f) r0 += r1
  11: (05) goto pc+1
  12: (b7) r0 = 0
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
  13: (15) if r0 == 0x0 goto pc+2
  14: (b7) r1 = 1
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
  15: (db) lock *(u64 *)(r0 +0) += r1
; KFUNC_PROBE(vfs_write)        { stats_try_increment(S_WRITE); return 0; }
  16: (b7) r0 = 0
  17: (95) exit
```

```
func#0 @0
0: R1=ctx(id=0,off=0,imm=0) R10=fp0
; KFUNC_PROBE(vfs_write)        { stats_try_increment(S_WRITE); return 0; }
0: (b7) r1 = 2
1: R1_w=inv2 R10=fp0
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
1: (63) *(u32 *)(r10 -4) = r1
2: R1_w=inv2 R10=fp0 fp-8=mmmm????
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
2: (18) r1 = 0xffff9c7cc9ec3e00
4: R1_w=map_ptr(id=0,off=0,ks=4,vs=8,imm=0) R10=fp0 fp-8=mmmm????
4: (bf) r2 = r10
5: R1_w=map_ptr(id=0,off=0,ks=4,vs=8,imm=0) R2_w=fp0 R10=fp0 fp-8=mmmm????
;
5: (07) r2 += -4
6: R1_w=map_ptr(id=0,off=0,ks=4,vs=8,imm=0) R2_w=fp-4 R10=fp0 fp-8=mmmm????
; return bpf_map_lookup_elem((void *)map, key);
6: (85) call bpf_map_lookup_elem#1
7: R0_w=map_value_or_null(id=1,off=0,ks=4,vs=8,imm=0) R10=fp0 fp-8=mmmm????
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
7: (15) if r0 == 0x0 goto pc+2
 R0_w=map_value(id=0,off=0,ks=4,vs=8,imm=0) R10=fp0 fp-8=mmmm????
8: R0_w=map_value(id=0,off=0,ks=4,vs=8,imm=0) R10=fp0 fp-8=mmmm????
8: (b7) r1 = 1
9: R0_w=map_value(id=0,off=0,ks=4,vs=8,imm=0) R1_w=inv1 R10=fp0 fp-8=mmmm????
; ({ typeof(stats.key) _key = key; typeof(stats.leaf) *_leaf = bpf_map_lookup_elem_(bpf_pseudo_fd(1, -1), &_key); if (_leaf) lock_xadd(_leaf, 1);});
9: (db) lock *(u64 *)(r0 +0) += r1
 R0_w=map_value(id=0,off=0,ks=4,vs=8,imm=0) R1_w=inv1 R10=fp0 fp-8=mmmm????
 R0_w=map_value(id=0,off=0,ks=4,vs=8,imm=0) R1_w=inv1 R10=fp0 fp-8=mmmm????
10: R0=map_value(id=0,off=0,ks=4,vs=8,imm=0) R1=inv1 R10=fp0 fp-8=mmmm????
; KFUNC_PROBE(vfs_write)        { stats_try_increment(S_WRITE); return 0; }
10: (b7) r0 = 0
11: R0_w=inv0 R1=inv1 R10=fp0 fp-8=mmmm????
11: (95) exit
```

相关的 log 打印逻辑在 

```c
static int do_check(struct bpf_verifier_env *env)
{
    // ...
		if (env->log.level & BPF_LOG_LEVEL2 ||
		    (env->log.level & BPF_LOG_LEVEL && do_print_state)) {
			if (env->log.level & BPF_LOG_LEVEL2)
				verbose(env, "%d:", env->insn_idx);
			else
				verbose(env, "\nfrom %d to %d%s:",
					env->prev_insn_idx, env->insn_idx,
					env->cur_state->speculative ?
					" (speculative execution)" : "");
            // 打印 verifer 的所有寄存器状态
			print_verifier_state(env, state->frame[state->curframe]);
			do_print_state = false;
		}

		if (env->log.level & BPF_LOG_LEVEL) {
			const struct bpf_insn_cbs cbs = {
				.cb_call	= disasm_kfunc_name,
				.cb_print	= verbose,
				.private_data	= env,
			};
            // 打印原来的 bpf 程序的代码，根据 insn_idx 找到代码
			verbose_linfo(env, env->insn_idx, "; ");
			verbose(env, "%d: ", env->insn_idx);
            // 打印 bpf 汇编指令
			print_bpf_insn(&cbs, insn, env->allow_ptr_leaks);
		}
}
```



可以看到有两种指令比如，26 序号有两条指令。

```
26: R0_w=inv(id=0) R1_w=map_ptr(id=0,off=0,ks=4,vs=8,imm=0) R2_w=fp-4 R3_w=fp0 R6=inv(id=2) R7=invP0 R10=fp0 fp-8=mmmm???? fp-16_w=mmmmmmmm
26: (07) r3 += -16
```

其中大写寄存器的是 verifier state，其相关的内容在 `kernel/bpf/verifier.c`，

```c
// kernel/bpf/verifier.c
/* string representation of 'enum bpf_reg_type' */
static const char * const reg_type_str[] = {
	[NOT_INIT]		= "?",
	[SCALAR_VALUE]		= "inv",
    //...
};

static char slot_type_char[] = {
	[STACK_INVALID]	= '?',
    // ...
};

static void print_verifier_state(struct bpf_verifier_env *env,
				 const struct bpf_func_state *state);
```

小写的是 指令汇编代码，相关内容在，在 kernel/bpf/disasm.c 里。

```c
// kernel/bpf/disasm.c
void print_bpf_insn(const struct bpf_insn_cbs *cbs,
		    const struct bpf_insn *insn,
		    bool allow_ptr_leaks);
```



## examples

编译命令：

```shell
clang \
-target bpf \
-D __TARGET_ARCH_x86 \
-Wall \
-O2 -g -o bpf_test_3.bpf.o -c bpf_test_3.bpf.c
```

Load:

```
sudo bpftool prog load -d bpf_test_3.bpf.o /sys/fs/bpf/hello3
```

### exp1 bound check

我们从一个 bpf verifier load 失败的例子来看看怎么分析 bpf load 失败（怎么看汇编和看寄存器状态）

这个例子来自 [国内 ebpf 大会的例子](https://view.officeapps.live.com/op/view.aspx?src=https%3A%2F%2Febpftravel.com%2Fuploads%2Fmain%2F3_YRQ_linux_tracing_system_analysis.pptx) ，当时看了印象深刻，于是拿过来研究一番。

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

#define MAX_SIZE 12

struct user_msg_t {
   char message[12];
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, u32);
    __type(value, struct user_msg_t);
} array_map SEC(".maps");

SEC("kprobe/do_unlinkat")
int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
{
	u32 key = 0;
	struct user_msg_t *userd = bpf_map_lookup_elem(&array_map, &key);
	if (userd == NULL) {
		return 0;
	}

	unsigned int pos = bpf_get_smp_processor_id();;

	if (pos < MAX_SIZE){
		userd->message[pos] = 1;
		pos += 1;
	}

	// bpf_printk("debug %d %d %d\n", bpf_get_current_pid_tgid() >> 32,
	// bpf_get_current_pid_tgid(), userd->message[1]);

	if (pos < MAX_SIZE)
		userd->message[pos] = 1;

	return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

failed to load

```
-- BEGIN PROG LOAD LOG --
func#0 @0
R1 type=ctx expected=fp
0: R1=ctx(id=0,off=0,imm=0) R10=fp0
; int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
0: (b7) r1 = 0
1: R1_w=inv0 R10=fp0
; u32 key = 0;
1: (63) *(u32 *)(r10 -4) = r1
last_idx 1 first_idx 0
regs=2 stack=0 before 0: (b7) r1 = 0
2: R1_w=invP0 R10=fp0 fp-8=0000????
2: (bf) r2 = r10
3: R1_w=invP0 R2_w=fp0 R10=fp0 fp-8=0000????
; 
3: (07) r2 += -4
4: R1_w=invP0 R2_w=fp-4 R10=fp0 fp-8=0000????
; struct user_msg_t *userd = bpf_map_lookup_elem(&array_map, &key);
4: (18) r1 = 0xffff9382ed4bc000
6: R1_w=map_ptr(id=0,off=0,ks=4,vs=12,imm=0) R2_w=fp-4 R10=fp0 fp-8=0000????
6: (85) call bpf_map_lookup_elem#1
7: R0_w=map_value_or_null(id=1,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
7: (bf) r6 = r0
8: R0_w=map_value_or_null(id=1,off=0,ks=4,vs=12,imm=0) R6_w=map_value_or_null(id=1,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; if (userd == NULL) {
8: (15) if r6 == 0x0 goto pc+15
 R0_w=map_value(id=0,off=0,ks=4,vs=12,imm=0) R6_w=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
9: R0_w=map_value(id=0,off=0,ks=4,vs=12,imm=0) R6_w=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; unsigned int pos = bpf_get_smp_processor_id();;
9: (85) call bpf_get_smp_processor_id#8
10: R0=inv(id=0) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
10: (bf) r2 = r0
11: R0=inv(id=2) R2_w=inv(id=2) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
11: (67) r2 <<= 32
12: R0=inv(id=2) R2_w=inv(id=0,smax_value=9223372032559808512,umax_value=18446744069414584320,var_off=(0x0; 0xffffffff00000000),s32_min_value=0,s32_max_value=0,u32_max_value=0) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
12: (77) r2 >>= 32
13: R0=inv(id=2) R2_w=inv(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; if (pos < MAX_SIZE){
13: (25) if r2 > 0xb goto pc+10
---
 R0=inv(id=2) R2_w=inv(id=0,umax_value=11,var_off=(0x0; 0xf)) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
---
14: R0=inv(id=2) R2_w=inv(id=0,umax_value=11,var_off=(0x0; 0xf)) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; userd->message[pos] = 1;
14: (bf) r3 = r6
15: R0=inv(id=2) R2_w=inv(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,imm=0) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
15: (0f) r3 += r2
last_idx 15 first_idx 10
regs=4 stack=0 before 14: (bf) r3 = r6
regs=4 stack=0 before 13: (25) if r2 > 0xb goto pc+10
regs=4 stack=0 before 12: (77) r2 >>= 32
regs=4 stack=0 before 11: (67) r2 <<= 32
regs=4 stack=0 before 10: (bf) r2 = r0
 R0_rw=invP(id=0) R6_rw=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
parent didn't have regs=1 stack=0 marks
last_idx 9 first_idx 0
regs=1 stack=0 before 9: (85) call bpf_get_smp_processor_id#8
16: R0=inv(id=2) R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
16: (b7) r1 = 1
17: R0=inv(id=2) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; userd->message[pos] = 1;
17: (73) *(u8 *)(r3 +0) = r1
 R0=inv(id=2) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
18: R0=inv(id=2) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; if (pos < MAX_SIZE)
18: (15) if r2 == 0xb goto pc+5
 R0=inv(id=2) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????1
19: R0=inv(id=2) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; pos += 1;
19: (07) r0 += 1
20: R0_w=inv(id=0) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
; userd->message[pos] = 1;
20: (67) r0 <<= 32
21: R0_w=inv(id=0,smax_value=9223372032559808512,umax_value=18446744069414584320,var_off=(0x0; 0xffffffff00000000),s32_min_value=0,s32_max_value=0,u32_max_value=0) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
21: (77) r0 >>= 32
22: R0_w=inv(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
22: (0f) r6 += r0
last_idx 22 first_idx 10
regs=1 stack=0 before 21: (77) r0 >>= 32
regs=1 stack=0 before 20: (67) r0 <<= 32
regs=1 stack=0 before 19: (07) r0 += 1
regs=1 stack=0 before 18: (15) if r2 == 0xb goto pc+5
regs=1 stack=0 before 17: (73) *(u8 *)(r3 +0) = r1
regs=1 stack=0 before 16: (b7) r1 = 1
regs=1 stack=0 before 15: (0f) r3 += r2
regs=1 stack=0 before 14: (bf) r3 = r6
regs=1 stack=0 before 13: (25) if r2 > 0xb goto pc+10
regs=1 stack=0 before 12: (77) r2 >>= 32
regs=1 stack=0 before 11: (67) r2 <<= 32
regs=1 stack=0 before 10: (bf) r2 = r0
 R0_rw=invP(id=0) R6_rw=map_value(id=0,off=0,ks=4,vs=12,imm=0) R10=fp0 fp-8=mmmm????
parent already had regs=1 stack=0 marks
23: R0_w=invP(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R10=fp0 fp-8=mmmm????
; userd->message[pos] = 1;
23: (73) *(u8 *)(r6 +0) = r1
---
 R0_w=invP(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R1_w=inv1 R2_w=invP(id=0,umax_value=11,var_off=(0x0; 0xf)) R3_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=11,var_off=(0x0; 0xf),s32_max_value=15,u32_max_value=15) R6_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R10=fp0 fp-8=mmmm????
R6 unbounded memory access, make sure to bounds check any such access
---
verification time 243 usec
stack depth 4
processed 23 insns (limit 1000000) max_states_per_insn 0 total_states 1 peak_states 1 mark_read 1
-- END PROG LOAD LOG --
```

assembly code:

```
~/proj/learning-ebpf/scratch (main ✗) llvm-objdump-14 -S bpf_test.bpf.o

bpf_test.bpf.o: file format elf64-bpf

Disassembly of section kprobe/do_unlinkat:

0000000000000000 <do_unlinkat>:
; int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
       0:       b7 01 00 00 00 00 00 00 r1 = 0
;       u32 key = 0;
       1:       63 1a fc ff 00 00 00 00 *(u32 *)(r10 - 4) = r1
       2:       bf a2 00 00 00 00 00 00 r2 = r10
       3:       07 02 00 00 fc ff ff ff r2 += -4
;       struct user_msg_t *userd = bpf_map_lookup_elem(&array_map, &key);
       4:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
       6:       85 00 00 00 01 00 00 00 call 1
       7:       bf 06 00 00 00 00 00 00 r6 = r0
;       if (userd == NULL) {
       8:       15 06 0f 00 00 00 00 00 if r6 == 0 goto +15 <LBB0_4>
;       unsigned int pos = bpf_get_smp_processor_id();;
       9:       85 00 00 00 08 00 00 00 call 8
      10:       bf 02 00 00 00 00 00 00 r2 = r0
      11:       67 02 00 00 20 00 00 00 r2 <<= 32
      12:       77 02 00 00 20 00 00 00 r2 >>= 32
;       if (pos < MAX_SIZE){
      13:       25 02 0a 00 0b 00 00 00 if r2 > 11 goto +10 <LBB0_4>
;               userd->message[pos] = 1;
      14:       bf 63 00 00 00 00 00 00 r3 = r6
      15:       0f 23 00 00 00 00 00 00 r3 += r2
      16:       b7 01 00 00 01 00 00 00 r1 = 1
      17:       73 13 00 00 00 00 00 00 *(u8 *)(r3 + 0) = r1
;       if (pos < MAX_SIZE)
      18:       15 02 05 00 0b 00 00 00 if r2 == 11 goto +5 <LBB0_4>
;               pos += 1;
      19:       07 00 00 00 01 00 00 00 r0 += 1
;               userd->message[pos] = 1;
      20:       67 00 00 00 20 00 00 00 r0 <<= 32
      21:       77 00 00 00 20 00 00 00 r0 >>= 32
      22:       0f 06 00 00 00 00 00 00 r6 += r0
      23:       73 16 00 00 00 00 00 00 *(u8 *)(r6 + 0) = r1

00000000000000c0 <LBB0_4>:
; int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
      24:       b7 00 00 00 00 00 00 00 r0 = 0
      25:       95 00 00 00 00 00 00 00 exit
```

从 log 来看其实比较直观，就是说 R6 的边界检查失败了。R6 的状态为 `R6_w=map_value(id=0,off=0,ks=4,vs=12,umax_value=4294967295,var_off=(0x0; 0xffffffff)) ` ，它的最大值为 4294967295，比 map 的 max_size 大，

* 这里我们研究一下第一段汇编代码

```
; int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
       0:	b7 01 00 00 00 00 00 00	r1 = 0
; 	u32 key = 0;
       1:	63 1a fc ff 00 00 00 00	*(u32 *)(r10 - 4) = r1
       2:	bf a2 00 00 00 00 00 00	r2 = r10
       3:	07 02 00 00 fc ff ff ff	r2 += -4

; int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
0: (b7) r1 = 0
1: R1_w=inv0 R10=fp0
; u32 key = 0;
1: (63) *(u32 *)(r10 -4) = r1
last_idx 1 first_idx 0
regs=2 stack=0 before 0: (b7) r1 = 0
2: R1_w=invP0 R10=fp0 fp-8=0000????
2: (bf) r2 = r10
3: R1_w=invP0 R2_w=fp0 R10=fp0 fp-8=0000????
;
3: (07) r2 += -4
4: R1_w=invP0 R2_w=fp-4 R10=fp0 fp-8=0000????
```

1. `r1=0` 之后显示，r1 寄存器的状态为 `R1_w=inv0`表示 R1刚被写入，是一个 scalar类型的值，可以看到首先赋值给 r1=1。
2. `*(u32 *)(r10 -4) = r1` 需要将 r1 的值 0 赋值到给一个栈上的地址。
3. `r2 = r10`  和 `r2 += -4`将 r10（栈指针）赋值给 r2，将 r2 指向该栈上的值

* 接着第二句 `;       struct user_msg_t *userd = bpf_map_lookup_elem(&array_map, &key);` 

```
       4:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
       6:       85 00 00 00 01 00 00 00 call 1
       7:       bf 06 00 00 00 00 00 00 r6 = r0
```

1. `r1 = 0 ll` 可以看到，在这一句中，verifier log 中对应的语句是 ` r1 = 0xffff9382ed4bc000`，这个是 array_map 地址。从 verifer log 中可以看到这个语句之后，寄存器的类型是 map_ptr。因为上面的汇编代码是 llvm-objdump 的结果，所以可能还没有 load 进内核的时候，其实是不知道 map 的地址的。
2. 0x85 是 helper function call，call 1 是在调用 bpf_map_lookup_elem，这个  id 对应了 `include/uapi/linux/bpf.h` 里的 __BPF_FUNC_MAPPER 宏。
3. 最后是 r6 存一下 helper function 的返回值

* 接着3,4句

```

;       if (userd == NULL) {
       8:       15 06 0f 00 00 00 00 00 if r6 == 0 goto +15 <LBB0_4>
;       unsigned int pos = bpf_get_smp_processor_id();;
       9:       85 00 00 00 08 00 00 00 call 8
      10:       bf 02 00 00 00 00 00 00 r2 = r0
      11:       67 02 00 00 20 00 00 00 r2 <<= 32
      12:       77 02 00 00 20 00 00 00 r2 >>= 32
```

1. 这里调用了 8 号辅助函数 `bpf_get_smp_processor_id`，在之后用 r2 来缓存了 r0，不过为什么要做这样一个 移位的操作，不是很懂，总之是编译器的行为

* 再接着，第一次判断 MAX_SIZE 的关系，对应代码为

```c
	if (pos < MAX_SIZE){
		userd->message[pos] = 1;
		pos += 1;
	}
```

```
;       if (pos < MAX_SIZE){
      13:       25 02 0a 00 0b 00 00 00 if r2 > 11 goto +10 <LBB0_4>
;               userd->message[pos] = 1;
      14:       bf 63 00 00 00 00 00 00 r3 = r6
      15:       0f 23 00 00 00 00 00 00 r3 += r2
      16:       b7 01 00 00 01 00 00 00 r1 = 1
      17:       73 13 00 00 00 00 00 00 *(u8 *)(r3 + 0) = r1
```

1. 这段代码对移位后的 r0 判断了大小，然后对 map 地址 执行了写操作。
2. 这里没有 pos+=1 的操作
3.  r3 的范围来自 r2，r2 的状态为 ` R2_w=inv(id=0,umax_value=11,var_off=(0x0; 0xf))` 

* 最后，判断 pos 和 MAX_SIZE 的关系，对应代码为

```c
	if (pos < MAX_SIZE)
		userd->message[pos] = 1;
```

```
;       if (pos < MAX_SIZE)
      18:       15 02 05 00 0b 00 00 00 if r2 == 11 goto +5 <LBB0_4>
;               pos += 1;
      19:       07 00 00 00 01 00 00 00 r0 += 1
;               userd->message[pos] = 1;
      20:       67 00 00 00 20 00 00 00 r0 <<= 32
      21:       77 00 00 00 20 00 00 00 r0 >>= 32
      22:       0f 06 00 00 00 00 00 00 r6 += r0
      23:       73 16 00 00 00 00 00 00 *(u8 *)(r6 + 0) = r1
```

1. 这一段首先判断了 pos 是不是等于 11，因为 pos+=1 这个逻辑是在上一个 if 中的，经过编译器编译，放到这个逻辑里面，并且加了个判断
2. 这里最终访问 map 的寄存器来自于 r0，为什么不用 r2 呢，不知道编译器怎么想的



### exp2: pkt access

由某篇的博客变形而来：https://hechao.li/2019/06/20/An-Invalid-bpf_context-Access-Bug/，他的 kernel 似乎是 4.x 的

```c
SEC("sockops")
int func2(struct bpf_sock_ops* skops) {
  __u64 byte0 = skops->remote_ip6[0];
  __u64 byte1 = skops->remote_ip6[1];
  // __u64 byte2 = skops->remote_ip6[2];
  // __u64 byte3 = skops->remote_ip6[3];

  __u64 addr_hi = (byte1 << 32) | byte0;
  // __u64 addr_lo = (byte3 << 32) | byte2;

  bpf_trace_printk("remote_ip: %llu\n", addr_hi);

  return 0;
}
```

```
 -- BEGIN PROG LOAD LOG --
func#0 @0
0: R1=ctx(id=0,off=0,imm=0) R10=fp0
; bpf_trace_printk("remote_ip: %llu\n", addr_hi);
0: (61) r2 = *(u32 *)(r1 +32)
1: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R10=fp0
1: (18) r1 = 0xffff9381bb0ff910
3: R1_w=map_value(id=0,off=0,ks=4,vs=17,imm=0) R2_w=inv(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R10=fp0
3: (85) call bpf_trace_printk#6
 R1_w=map_value(id=0,off=0,ks=4,vs=17,imm=0) R2_w=inv(id=0,umax_value=4294967295,var_off=(0x0; 0xffffffff)) R10=fp0
invalid access to map value, value_size=17 off=0 size=0
R1 min value is outside of the allowed memory range
verification time 312 usec
stack depth 0
processed 3 insns (limit 1000000) max_states_per_insn 0 total_states 0 peak_states 0 mark_read 0
-- END PROG LOAD LOG --
```

```shell
~/proj/learning-ebpf/scratch (main ✗) llvm-objdump-14 -S bpf_test_2.bpf.o

bpf_test_2.bpf.o:       file format elf64-bpf

Disassembly of section sockops:

0000000000000000 <func2>:
;   bpf_trace_printk("remote_ip: %llu\n", addr_hi);
       0:       61 12 20 00 00 00 00 00 r2 = *(u32 *)(r1 + 32)
       1:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
       3:       85 00 00 00 06 00 00 00 call 6
;   return 0;
       4:       b7 00 00 00 00 00 00 00 r0 = 0
       5:       95 00 00 00 00 00 00 00 exit
```

编译器直接给优化到直接算出 r2 了，但 还是挂了

### exp3: kprobe pt_regs

从 https://github.com/iovisor/bcc/issues/751 而来，我改写成了 libbpf 版的

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

static int dummy1(struct pt_regs *ctx, void *dest, size_t len) {          
	switch(ctx->ip) {                                                             
		case 0x55e29a4ce2a4ULL: *((uint64_t *)dest) = (uint64_t)ctx->si; return 0;    
		case 0x55e29a4ce7abULL: *((uint64_t *)dest) = (uint64_t)ctx->bp; return 0;    
	}                                                                             
	return -1;                                                                    
}                  

SEC("kprobe/__x64_sys_getpgid")
int do_start(struct pt_regs *ctx)                                               
{                                                                               
        u64 arg1 = 0;         

        int ret2 = dummy1(ctx, &arg1, sizeof(arg1));                                        
        if (arg1 != 0) {                                                        
            bpf_trace_printk("Hi 2, %d\n", ret2);  
        }                                                                       

        return 0;                                                               
}              
```

```
-- BEGIN PROG LOAD LOG --
func#0 @0
R1 type=ctx expected=fp
0: R1=ctx(id=0,off=0,imm=0) R10=fp0
; switch(ctx->ip) {                                                             
0: (79) r2 = *(u64 *)(r1 +128)
1: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0) R10=fp0
1: (18) r3 = 0x55e29a4ce7ab
3: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0) R3_w=inv94431739701163 R10=fp0
; switch(ctx->ip) {                                                             
3: (1d) if r2 == r3 goto pc+5
 R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0) R3_w=inv94431739701163 R10=fp0
4: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0) R3_w=inv94431739701163 R10=fp0
4: (18) r3 = 0x55e29a4ce2a4
6: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0) R3_w=inv94431739699876 R10=fp0
6: (5d) if r2 != r3 goto pc+10
 R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0,umin_value=94431739699876,umax_value=94431739699876,var_off=(0x55e200000000; 0xffffffff)) R3_w=inv94431739699876 R10=fp0
7: R1=ctx(id=0,off=0,imm=0) R2_w=inv(id=0,umin_value=94431739699876,umax_value=94431739699876,var_off=(0x55e200000000; 0xffffffff)) R3_w=inv94431739699876 R10=fp0
7: (b7) r2 = 104
8: R1=ctx(id=0,off=0,imm=0) R2_w=inv104 R3_w=inv94431739699876 R10=fp0
8: (05) goto pc+1
10: R1=ctx(id=0,off=0,imm=0) R2=inv104 R3=inv94431739699876 R10=fp0
10: (0f) r1 += r2
last_idx 10 first_idx 10
 R1_r=ctx(id=0,off=0,imm=0) R2_rw=invP104 R3_w=inv94431739699876 R10=fp0
parent didn't have regs=4 stack=0 marks
last_idx 8 first_idx 0
regs=4 stack=0 before 8: (05) goto pc+1
regs=4 stack=0 before 7: (b7) r2 = 104
11: R1_w=ctx(id=0,off=104,imm=0) R2=invP104 R3=inv94431739699876 R10=fp0
; 
11: (79) r1 = *(u64 *)(r1 +0)
dereference of modified ctx ptr R1 off=104 disallowed
verification time 102 usec
stack depth 0
processed 9 insns (limit 1000000) max_states_per_insn 0 total_states 1 peak_states 1 mark_read 1
-- END PROG LOAD LOG --
```

```
~/proj/learning-ebpf/scratch (main ✗) llvm-objdump-14 -S bpf_test_3.bpf.o

bpf_test_3.bpf.o:       file format elf64-bpf

Disassembly of section kprobe/__x64_sys_getpgid:

0000000000000000 <do_start>:
;       switch(ctx->ip) {                                                             
       0:       79 12 80 00 00 00 00 00 r2 = *(u64 *)(r1 + 128)
       1:       18 03 00 00 ab e7 4c 9a 00 00 00 00 e2 55 00 00 r3 = 94431739701163 ll
       3:       1d 32 05 00 00 00 00 00 if r2 == r3 goto +5 <LBB0_3>
       4:       18 03 00 00 a4 e2 4c 9a 00 00 00 00 e2 55 00 00 r3 = 94431739699876 ll
;       switch(ctx->ip) {                                                             
       6:       5d 32 0a 00 00 00 00 00 if r2 != r3 goto +10 <LBB0_6>
       7:       b7 02 00 00 68 00 00 00 r2 = 104
       8:       05 00 01 00 00 00 00 00 goto +1 <LBB0_4>

0000000000000048 <LBB0_3>:
       9:       b7 02 00 00 20 00 00 00 r2 = 32

0000000000000050 <LBB0_4>:
      10:       0f 21 00 00 00 00 00 00 r1 += r2
      11:       79 11 00 00 00 00 00 00 r1 = *(u64 *)(r1 + 0)
;         if (arg1 != 0) {                                                        
      12:       15 01 04 00 00 00 00 00 if r1 == 0 goto +4 <LBB0_6>
;             bpf_trace_printk("Hi 2, %d\n", ret2);  
      13:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
      15:       b7 02 00 00 00 00 00 00 r2 = 0
      16:       85 00 00 00 06 00 00 00 call 6

0000000000000088 <LBB0_6>:
;         return 0;                                                               
      17:       b7 00 00 00 00 00 00 00 r0 = 0
      18:       95 00 00 00 00 00 00 00 exit
~/proj/learning-ebpf/scratch (main ✗) 
```

### exp4: clang O0 vs. O2

O2 可以 load，O0 不可以

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

static int dummy2(int a) {
	int a1 = a+1;
	int a2 = a1+1;
	int a3 = a2+1;
	int a4 = a3+1;

	return a1+a2+a3+a4;
}

SEC("kprobe/__x64_sys_getpgid")
int do_start(struct pt_regs *ctx)                                               
{                                                                               
        u64 arg1 = 0;         

		int ret1 = dummy2(arg1);                                 
		bpf_trace_printk("Hi 1, %d\n", ret1);           

        return 0;                                                               
}              
```

```
-- END BTF LOAD LOG --
libbpf: Error loading .BTF into kernel: -22. BTF is optional, ignoring.
libbpf: map 'bpf_test.data': created successfully, fd=3
libbpf: map '.rodata.str1.1': created successfully, fd=4
libbpf: prog 'do_start': added 21 insns from sub-prog 'dummy2'
libbpf: prog 'do_start': insn #5 relocated, imm 10 points to subprog 'dummy2' (now at 16 offset)
libbpf: prog 'do_start': BPF program load failed: Invalid argument
libbpf: prog 'do_start': -- BEGIN PROG LOAD LOG --
func#0 @0
func#1 @16
unknown opcode 8d
verification time 14 usec
stack depth 0+0
processed 0 insns (limit 1000000) max_states_per_insn 0 total_states 0 peak_states 0 mark_read 0
-- END PROG LOAD LOG --
```

不知道为什么 O0 编译出了，unknown op code，有 callx 这种指令

```
~/proj/learning-ebpf/scratch (main ✗) llvm-objdump-14 -S bpf_test_3.bpf.o

bpf_test_3.bpf.o:       file format elf64-bpf

Disassembly of section .text:

0000000000000000 <dummy2>:
; static int dummy2(int a) {
       0:       63 1a fc ff 00 00 00 00 *(u32 *)(r10 - 4) = r1
;       int a1 = a+1;
       1:       61 a1 fc ff 00 00 00 00 r1 = *(u32 *)(r10 - 4)
       2:       07 01 00 00 01 00 00 00 r1 += 1
       3:       63 1a f8 ff 00 00 00 00 *(u32 *)(r10 - 8) = r1
;       int a2 = a1+1;
       4:       61 a1 f8 ff 00 00 00 00 r1 = *(u32 *)(r10 - 8)
       5:       07 01 00 00 01 00 00 00 r1 += 1
       6:       63 1a f4 ff 00 00 00 00 *(u32 *)(r10 - 12) = r1
;       int a3 = a2+1;
       7:       61 a1 f4 ff 00 00 00 00 r1 = *(u32 *)(r10 - 12)
       8:       07 01 00 00 01 00 00 00 r1 += 1
       9:       63 1a f0 ff 00 00 00 00 *(u32 *)(r10 - 16) = r1
;       int a4 = a3+1;
      10:       61 a1 f0 ff 00 00 00 00 r1 = *(u32 *)(r10 - 16)
      11:       07 01 00 00 01 00 00 00 r1 += 1
      12:       63 1a ec ff 00 00 00 00 *(u32 *)(r10 - 20) = r1
;       return a1+a2+a3+a4;
      13:       61 a0 f8 ff 00 00 00 00 r0 = *(u32 *)(r10 - 8)
      14:       61 a1 f4 ff 00 00 00 00 r1 = *(u32 *)(r10 - 12)
      15:       0f 10 00 00 00 00 00 00 r0 += r1
      16:       61 a1 f0 ff 00 00 00 00 r1 = *(u32 *)(r10 - 16)
      17:       0f 10 00 00 00 00 00 00 r0 += r1
      18:       61 a1 ec ff 00 00 00 00 r1 = *(u32 *)(r10 - 20)
      19:       0f 10 00 00 00 00 00 00 r0 += r1
      20:       95 00 00 00 00 00 00 00 exit

Disassembly of section kprobe/__x64_sys_getpgid:

0000000000000000 <do_start>:
; {                                                                               
       0:       7b 1a f8 ff 00 00 00 00 *(u64 *)(r10 - 8) = r1
       1:       b7 01 00 00 00 00 00 00 r1 = 0
;         u64 arg1 = 0;         
       2:       7b 1a e0 ff 00 00 00 00 *(u64 *)(r10 - 32) = r1
       3:       7b 1a f0 ff 00 00 00 00 *(u64 *)(r10 - 16) = r1
;               int ret1 = dummy2(arg1);                                 
       4:       79 a1 f0 ff 00 00 00 00 r1 = *(u64 *)(r10 - 16)
       5:       85 10 00 00 ff ff ff ff call -1
       6:       63 0a ec ff 00 00 00 00 *(u32 *)(r10 - 20) = r0
;               bpf_trace_printk("Hi 1, %d\n", ret1); 
       7:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
       9:       79 13 00 00 00 00 00 00 r3 = *(u64 *)(r1 + 0)
      10:       61 a2 ec ff 00 00 00 00 r2 = *(u32 *)(r10 - 20)
      11:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
      13:       8d 00 00 00 03 00 00 00 callx r3
;         return 0;                                                               
      14:       79 a0 e0 ff 00 00 00 00 r0 = *(u64 *)(r10 - 32)
      15:       95 00 00 00 00 00 00 00 exit
~/proj/learning-ebpf/scratch (main ✗) 
```

另外，从这里也可以看出来，r1 只是参数，并不一定是 PTR_TO_CTX。
