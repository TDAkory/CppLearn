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
* 