# Constant Folding & Constant Propagation

## Constant Folding

常量折叠（Constant Folding）是指在编译器或解释器中对表达式进行优化的一种技术。它通过计算和简化表达式中的常量部分，将其替换为结果常量，从而减少运行时的计算开销。

常量折叠具有以下几个优点：

* 提高程序的执行效率和性能，避免重复计算。     
* 减小可执行文件的体积，节省存储空间。     
* 简化代码，提高代码的可读性和维护性。     

常量折叠的缺点主要包括以下几点：

* 只适用于常量表达式，无法对变量进行折叠优化。     
* 可能会导致编译时间增加，特别是在处理大量复杂表达式时。     
* 在某些情况下可能会引入精度损失或溢出问题。     

在使用常量折叠时，需要注意以下几点：

* 只对不涉及副作用的纯函数进行常量折叠。如果表达式中存在副作用（如修改全局变量、调用外部方法等），则不能进行常量折叠优化。     
* 避免过度依赖常量折叠，应合理设计代码结构和算法，以提高整体性能。     
* 注意数值溢出和精度损失问题，在进行常量折叠时需谨慎处理。    

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