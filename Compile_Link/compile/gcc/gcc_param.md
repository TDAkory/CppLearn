# [GCC Command Options](https://man7.org/linux/man-pages/man1/gcc.1.html)

- [GCC Command Options](#gcc-command-options)
    - [FMA](#fma)
    - [rdynamic](#rdynamic)


### [FMA](https://en.wikipedia.org/wiki/Multiply%E2%80%93accumulate_operation#Fused_multiply%E2%80%93add)

A fused multiply–add is a floating-point multiply–add operation performed in one step, with a single rounding. That is, where an unfused multiply–add would compute the product b × c, round it to N significant bits, add the result to a, and round back to N significant bits, a fused multiply–add would compute the entire expression a + (b × c) to its full precision before rounding the final result down to N significant bits.

* 浮点计算的结果要做round，`fma`指令少了把惩罚结果做round的过程，更精确，但是不符合`IEEE754`标准
* `-ffp-contract`可以打开clang的FMA优化，gcc默认打开
* 浮点数的优化需要嘎开选项-Ofast，而不是`O3`

### [rdynamic](https://stackoverflow.com/questions/36692315/what-exactly-does-rdynamic-do-and-when-exactly-is-it-needed)


`-rdynamic` exports the symbols of an executable, this mainly addresses scenarios as described in Mike Kinghan's answer, but also it helps e.g. Glibc's backtrace_symbols() symbolizing the backtrace.

`-rdynamic` Pass the flag ‘-export-dynamic’ to the ELF linker, on targets that support it. This instructs the linker to add all symbols, not only used ones, to the dynamic symbol table. This option is needed for some uses of dlopen or to allow obtaining backtraces from within a program.
