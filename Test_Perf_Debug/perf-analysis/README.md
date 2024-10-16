# perf-analysis

## Some Basic Thoughs

* 延迟可能是多个视角的：处理延迟、排队延迟、长尾延迟
* 数据结构压测，对比不同数据结构的cache有效利用率
* 减少内存的申请和释放，尽量复用，尽量不要缓存大对象
* 尽量不要使用spinlock，用户态spinlock和内核态spinlock是不同的！ 具体什么不同，有待进一步学习
* 使用更好的内存分配 jemalloc tcmalloc mimalloc， 可以做一个对比分析
* 先profiling，再优化，用数据说话，避免过度优化
* cgroup监控内存和top监控内存的区别
* 熟悉/proc文件系统下，各监控项描述的是什么

## 一些别人做过的事

* facebook hfsort 和 automated hot text and huge page 提高代码的缓存亲和性
* facebook bolt 减少二进制中的padding，提高缓存密度