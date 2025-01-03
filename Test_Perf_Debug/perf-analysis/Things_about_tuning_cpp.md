# Things about tuning C++

* [Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!](https://www.youtube.com/watch?v=nXaxk27zwlk)

* `perf stat` for general stat
* `perf record` & `perf report`
* `perf record -g` & `perf report -g`生成调用关系图
  * 如果堆栈信息不全，要加 `--call-graph dwarf/lbr`
* 一般的编译优化会干掉frame-pointer，导致perf无法获取调用关系，需要添加编译参数`-fno-omit-frame-pointer`
* perf产生的是callee的调用，比较反直觉，可以用如下参数重新打开`perf report -g 'graph,0.5,caller'`
  
```cpp
static void escape(void *p) {
    asm volatile("" : : "g"(p) : "memory");
}

static void clobber() {
    asm volatile("" : : : "memory");
}
```

See[extend asm](../../Compile_Link/compile/gcc/extend_asm.md)

- [使用 Perf 和火焰图分析 CPU 性能](https://senlinzhan.github.io/2018/03/18/perf/?from=from_parent_mindnote)