# [gdb](https://www.sourceware.org/gdb/)

## Some GDB Tricks

### Info display

* `(gdb) show version` 显式gdb版本信息
* `(gdb) show copying` 显式gdb版权信息
* `$ gdb -q` 启动时不提供信息
* `(gdb) set confirm off` 退出时不提供信息
* `set pagination off` `set height 0` 输出较多信息的时候，不会暂停并打印`---Type <return> to continue, or q <return> to quit---`

### Functions

* `info functions` 可以列出可执行文件的所有函数名称
  * `info functions regex` 可以利用正则表达式获取相关的函数
* `step`  `s` 进入带有调试信息的函数 `next` `n` 则是不进入函数，等函数执行完，显式县一个要执行的程序代码
* `set step-mode on` 一般情况下，`s`无法进入不带调试信息的函数，该命令可以使得gdb不跳过没有调试信息的函数，进入其中可以用调试汇编程序的办法来调试
* `finish` 可以让当前函数函数执行完，打印返回值，然后等待接下来的输入命令
  * `return` 可以直接返回，函数不会执行下面的语句
  * `return expression` 可以指定函数的返回值
* `call` `print` 可以直接调用函数执行
* `i frame` 可以打印函数堆栈帧信息
  * `i registers` 可以打印寄存器信息
  * `disassemble func` 可以显式函数的汇编指令
* `set debug entry-values 1` 开启尾调用
* `frame n` `frame addr` 可以选择函数堆栈帧
* `up n` `down n` 可以向上或向下选择函数堆栈帧，其中n是层数

### Breakpoint

* `b Foo::foo` 命名空间断点
  * `b (anonymous namespace)::bar` 匿名空间断点
* `b *0x400522` 在程序地址上打断点
* 当调试没有调试信息的程序时，直接运行start命令是没有效果的。如果不知道main在何处，那么可以在程序入口处打断点。先通过`readelf`或者进入gdb，执行`info files`获得入口地址，然后 `b *addr` `r`
* `b file.c:6` `b a/file.c:6` 在文件行号上断点
* `save breakpoints file-name-to-save` 保存断点信息
  * `source file-name-to-save` 批量设置保存的断点
* `tbreak` `tb` 断点只生效一次
* `break … if cond` `b 10 if i==101` 条件断点
* `ignore bnum count` 接下来count次编号为bnum的断点触发都不会让程序中断，只有第count + 1次断点触发才会让程序中断。

### Watchpoint

> 使用“watch”命令设置观察点，也就是当一个变量值发生变化时，程序会停下来。

* `watch param-name`
  * `watch *(data type*)address`
  * 观察点可以通过软件或硬件的方式实现，取决于具体的系统。但是软件实现的观察点会导致程序运行很慢
  * 如果系统支持硬件观测的话，当设置观测点是会打印如下信息： `Hardware watchpoint num: expr`
  * 如果不想用硬件观测点的话可如下设置： `set can-use-hw-watchpoints`
  * `info watchpoints` 列出当前所设置了的所有观察点
  * watch 所设置的断点也可以用控制断点的命令来控制。如 disable、enable、delete等
* `watch expr thread threadnum` 设置观察点只针对特定线程生效，这种针对特定线程设置观察点方式只对硬件观察点才生效
* `rwatch` 设置读观察点，也就是当发生读取变量行为时，程序就会暂停住,只对硬件观察点才生效
* `awatch` 设置读写观察点，也就是当发生读取变量或改变变量值的行为时，程序就会暂停住,只对硬件观察点才生效

### Catchpoint

> use catchpoints to cause the debugger to stop for certain kinds of program events, such as C++ exceptions or the loading of a shared library.

* `tcatch fork` 只由fork系统调用触发一次
* `catch fork` 为fork调用设置catchpoint,目前只有HP-UX和GNU/Linux支持这个功能
* `catch vfork` 为fork调用设置catchpoint,目前只有HP-UX和GNU/Linux支持这个功能
* `catch exec` 为fork调用设置catchpoint,目前只有HP-UX和GNU/Linux支持这个功能
* `catch syscall [name | number]` 关注的系统调用设置
  * `catch syscall mmap`
* 有些程序不想被gdb调试，它们就会在程序中调用“ptrace”函数，一旦返回失败，就证明程序正在被gdb等类似的程序追踪，所以就直接退出。

### Print

* `x/s $str` 打印ASCII字符串
  * `x/ws $str` 打印宽字符串