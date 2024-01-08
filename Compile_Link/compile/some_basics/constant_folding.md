# Constant Folding & Constant Propagation

## Constant Folding

### A example between compiler

fload.c

```c
#include <math.h>

long double foo() {
  return logl(3.4567);
}
```

gcc float.c -O3 -S

```shell
zhaojieyi@n37-018-026:~/tmp$ cat float.s
	.file	"float.c"
	.text
	.p2align 4,,15
	.globl	foo
	.type	foo, @function
foo:
.LFB0:
	.cfi_startproc
	fldt	.LC0(%rip)
	ret
	.cfi_endproc
.LFE0:
	.size	foo, .-foo
	.section	.rodata.cst16,"aM",@progbits,16
	.align 16
.LC0:
	.long	1732995966
	.long	2663554842
	.long	16383
	.long	0
	.ident	"GCC: (Debian 8.3.0-6) 8.3.0"
	.section	.note.GNU-stack,"",@progbits
```

clang float.c -O3 -S

```shell
λ C02DH0B1MD6R Self → cat float.s
    .section        __TEXT,__text,regular,pure_instructions
        .build_version macos, 14, 0
        .section        __TEXT,__literal8,8byte_literals
        .p2align        3, 0x0                          ## -- Begin function foo
LCPI0_0:
        .quad   0x400ba77c45cbbc2c              ## double 3.4567800000000002
        .section        __TEXT,__text,regular,pure_instructions
        .globl  _foo
        .p2align        4, 0x90
_foo:                                   ## @foo
        .cfi_startproc
## %bb.0:
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register %rbp
        subq    $16, %rsp
        fldl    LCPI0_0(%rip)
        fstpt   (%rsp)
        callq   _logl
        addq    $16, %rsp
        popq    %rbp
        retq
        .cfi_endproc
                                        ## -- End function
.subsections_via_symbols
```

可以看到gcc做了常量折叠，clang没有。

- 为什么llvm不做常量折叠？
  - 浮点数计算复杂，涉及不同的rounding模式，精度需要符合IEEE标准
  - long double 数据格式复杂，有 x86 的 80bit， PowerPC 的 双double，IEEE 的 128bit 等
  - GCC借助开源库 MPFR 来实现浮点数再编译期间的运算
  - MPFR 的 licence 是 LGPL，LLVM 的 licence 是 BSD，不允许直接使用 MPFR
  - 社区曾试图引入 MPFR 做常量折叠，[被否定](https://lists.llvm.org/pipermail/llvm-dev/2016-April/097936.html)
  - 不愿意依赖外置库、考虑编译运算的结果不一定等于运行环境中的结果