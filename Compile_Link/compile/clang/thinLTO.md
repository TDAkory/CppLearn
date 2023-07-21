# [ThinLTO](https://clang.llvm.org/docs/ThinLTO.html)

ThinLTO compilation is a new type of LTO that is both scalable and incremental. LTO (Link Time Optimization) achieves better runtime performance through whole-program analysis and cross-module optimization. However, monolithic LTO implements this by merging all input into a single module, which is not scalable in time or memory, and also prevents fast incremental compiles.

In ThinLTO mode, as with regular LTO, clang emits LLVM bitcode after the compile phase. The ThinLTO bitcode is augmented with a compact summary of the module. During the link step, only the summaries are read and merged into a combined summary index, which includes an index of function locations for later cross-module function importing. Fast and efficient whole-program analysis is then performed on the combined summary index.

However, all transformations, including function importing, occur later when the modules are optimized in fully parallel backends. By default, linkers that support ThinLTO are set up to launch the ThinLTO backends in threads. So the usage model is not affected as the distinction between the fast serial thin link step and the backends is transparent to the user.

- [ThinLTO: Scalable and Incremental LTO](http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html)
  