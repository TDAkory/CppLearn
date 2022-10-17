# Memory Bug & Address Sanitizer

## Memory Bug in C++

* Out-of-bounds accesses (OOB, buffer overflow/underflow) 
  * Stack
  * Heap
  * Globals
* Use-after-free (UAF, dangling pointer) 
* Use-after-return (UAR)
* Double free
* Invalid free
* Overapping memcpy parameters
* Uninitialized memory reads (UMR)
* Memory Leaks

## 常用工具

### Static Analysis

coverity, clang-tidy/clang-static-analyzer, cppcheck, etc

### Dynamic Analysis

AddressSanitizer、Valgrind/MemoryCheck、Dr.Memory、gperftools

#### AddressSanitizer方案

核心思想：申请 [0:23] 这块内存时，在其附近额外申请两块内存，作为“危险区” (redzone)，如果访问的地址位于“危险区”内，说明发生了溢出！

1. 申请 [0:23] 这块内存时，在其附近额外申请两块内存，作为“危险区”(redzone)

2. 每一段 application memory 都对应一段 shadow memory，根据 shadow memory 的值区分其对应的 application memory 是否为 redzone

3. 在每一次访存的之前，判断访问的地址是否位于“危险区”内。若是，则说明发生了溢出！

## AddressSanitizer

* Compile-time instrumentation
  * instruments all memory loads/stores, add sanity check if accessed memory is addressable
  * inserts redzones around stack variables and global variables

* Run-time library
  * malloc/free replacement, redzones are created around malloc-ed regions, freed memory is put into quarantine
  * bookkeeping for error messages

* Redzones are created around buffers; freed memory is put into quarantine
* Every 8 bytes of application memory are associated with 1 byte of “shadow” memory
* Shadow of redzones and freed memory is “poisoned”
* On every memory access compiler-injected code checks if shadow is poisoned

* Addressable: 00
* Stack left redzone: F1
* Stack right redzone: F3
* Heap left redzone: FA
* Freed heap redzone: FD