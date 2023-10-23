---
title: bpf verifier (2)
date: 2023-10-03 21:22:03
tags:
- bpf
---

# bpf verifier (2)

这一节结合 kernel docs 里 verifier 的介绍和源码一起学习一下 bpf verifier。

eBPF 程序的安全性由两个步骤确定。

第一步进行 DAG 检查以禁止循环和其他 CFG 验证。 特别是，它会检测具有无法访问指令的程序。 这是 `check_cfg()`做的事情。

第二步从第一个insn开始，沿着所有可能的路径遍历。 它模拟每个insn的执行并观察寄存器和堆栈的状态变化。`do_check()` 做的事情。

## 一些判定规则

1. 传递性，比如` R1` 是 `PTR_TO_CTX`，然后执行语句 `R2=R1`，然后我们就知道 `R2` 也是 `PTR_TO_CTX`了。
2. 不允许指针之间的算术运算，比如 `R2=R1+R1` 就不合法。
3. 未初始化的 寄存器不可以用来赋值，比如 `R1` 没有初始化，就不能执行 `R0=R2`。
4. kernel function call 之后，R1-R5 不能读取，R6-R9 是 callee saved, 他们的状态是一直保留的。

```
// 这段代码能过 verifier，但是如果把 R6 换成 R1 就不行了
bpf_mov R6 = 1
bpf_call foo
bpf_mov R0 = R6
bpf_exit
```

5. Load/store 只能对 PTR_TO_CTX, PTR_TO_MAP, PTR_TO_STACK 使用。他们会被 bound/alignment check
6. 一些 customized 的检查，比如 R1 的类型是 PTR_TO_CTX (a pointer to generic `struct bpf_context`) 一个 customized 的回调函数会用来检查 ebpf 程序对于特定 field 的访问。

```
bpf_ld R0 = *(u32 *)(R6 + 8)
```

比如以上程序，会尝试从 R6 + 8 的地址 load 一个 word 大小，放到 R0 里面。如果 R6 是 PTR_TO_CTX，通过  `is_valid_access()` callback 会进行验证

7. verifier 只允许 eBPF program to read data from stack only after it wrote into it. 比如下面这一段，它会读取 PTR_TO_STACK 的开始的 4 个字节，尽管 R10 就是  read-only register and has type PTR_TO_STACK，而且 R10 - 4 is within stack bounds，但是 verifier 会 reject，因为之前没有对这个位置做 store。

```
bpf_ld R0 = *(u32 *)(R10 - 4)
bpf_exit
```

8. Bpf helper function 会通过 bpf_verifier_ops->get_func_proto() ，然后会检查参数是否合规。



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



## 关于 subprogs 的验证

代码里面 `do_check_subprogs` 负责验证 subprog，注释里面提到了顺序问题，另外，关于 subprog，可以看到 subprog 就是一个 bpf object 里面的某一段指令，类似 c 的 global functions。

```c
/* Verify all global functions in a BPF program one by one based on their BTF.
 * All global functions must pass verification. Otherwise the whole program is rejected.
 * Consider:
 * int bar(int);
 * int foo(int f)
 * {
 *    return bar(f);
 * }
 * int bar(int b)
 * {
 *    ...
 * }
 * foo() will be verified first for R1=any_scalar_value. During verification it
 * will be assumed that bar() already verified successfully and call to bar()
 * from foo() will be checked for type match only. Later bar() will be verified
 * independently to check that it's safe for R1=any_scalar_value.
 */
static int do_check_subprogs(struct bpf_verifier_env *env) 
{
	struct bpf_prog_aux *aux = env->prog->aux;
	int i, ret;

	if (!aux->func_info)
		return 0;

	for (i = 1; i < env->subprog_cnt; i++) {
		if (aux->func_info_aux[i].linkage != BTF_FUNC_GLOBAL)
			continue;
		env->insn_idx = env->subprog_info[i].start;
		WARN_ON_ONCE(env->insn_idx == 0);
		ret = do_check_common(env, i);
		if (ret) {
			return ret;
		} else if (env->log.level & BPF_LOG_LEVEL) {
			verbose(env,
				"Func#%d is safe for any args that match its prototype\n",
				i);
		}
	}
	return 0;
}
```

可以看到一个 subprog 是从 env->subprog_info[i].start; 开始的一段指令。

然后 bpf prog 自己则是从 insn_idx = 0 开始的一段指令。

```c 
static int do_check_main(struct bpf_verifier_env *env)
{
	int ret;

	env->insn_idx = 0;
	ret = do_check_common(env, 0);
	if (!ret)
		env->prog->aux->stack_depth = env->subprog_info[0].stack_depth;
	return ret;
}

```

## other tips

1. Subprog 只能返回 int
1. liveness mark，表示寄存器的一些 read 和 write 关系 (bpf verifier log 里面有)

```c
/* Liveness marks, used for registers and spilled-regs (in stack slots).
 * Read marks propagate upwards until they find a write mark; they record that
 * "one of this state's descendants read this reg" (and therefore the reg is
 * relevant for states_equal() checks).
 * Write marks collect downwards and do not propagate; they record that "the
 * straight-line code that reached this state (from its parent) wrote this reg"
 * (and therefore that reads propagated from this state or its descendants
 * should not propagate to its parent).
 * A state with a write mark can receive read marks; it just won't propagate
 * them to its parent, since the write mark is a property, not of the state,
 * but of the link between it and its parent.  See mark_reg_read() and
 * mark_stack_slot_read() in kernel/bpf/verifier.c.
 */
enum bpf_reg_liveness {
	REG_LIVE_NONE = 0, /* reg hasn't been read or written this branch */
	REG_LIVE_READ32 = 0x1, /* reg was read, so we're sensitive to initial value */
	REG_LIVE_READ64 = 0x2, /* likewise, but full 64-bit content matters */
	REG_LIVE_READ = REG_LIVE_READ32 | REG_LIVE_READ64,
	REG_LIVE_WRITTEN = 0x4, /* reg was written first, screening off later reads */
	REG_LIVE_DONE = 0x8, /* liveness won't be updating this register anymore */
};
```



## run kernel tests

bpf verifier 的 test 在 

```shell
root@ubuntu2204:/home/kanxu/Documents/jammy/tools/testing/selftests/bpf# cat ./verifier/tests.h
/* Generated header, do not edit */
#ifdef FILL_ARRAY
#include "and.c"
#include "array_access.c"
// ...
#include "xdp.c"
#include "xdp_direct_packet_access.c"
#endif
```



## reference

1. kernel docs about verifier
   1. https://arthurchiao.art/blog/linux-socket-filtering-aka-bpf-zh/
   2. https://docs.kernel.org/bpf/verifier.html
2. 
