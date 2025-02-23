# Runtime Library

## 入口函数和程序初始化

入口函数（Entry Point）

* 操作系统创建进程后，把控制权交给程序的入口，这个入口往往是运行库汇中的某个入口函数
* 入口函数堆运行库和程序运行环境进行初始化：堆、IO、线程、全局变量构造等
* 入口函数完成后，调用main函数，执行程序主体
* main函数完成后，回到入口函数，进行清理工作：全局变量析构、堆销毁、关闭IO等
* 最终进行系统调用结束进程

### glibc的入口函数

glibc为不同的架构都定义了启动的入口函数，简便起见，我们看一下i386的实现：

```c
// glibc/glibc-2.41.9000/source/sysdeps/i386/start.S

ENTRY (_start)
	/* Clearing frame pointer is insufficient, use CFI.  */
	cfi_undefined (eip)
	/* Clear the frame pointer.  The ABI suggests this be done, to mark
	   the outermost frame obviously.  */
	xorl %ebp, %ebp

	/* Extract the arguments as encoded on the stack and set up
	   the arguments for `main': argc, argv.  envp will be determined
	   later in __libc_start_main.  */
	popl %esi		/* Pop the argument count.  */
	movl %esp, %ecx		/* argv starts just at the current stack top.*/

	/* Before pushing the arguments align the stack to a 16-byte
	(SSE needs 16-byte alignment) boundary to avoid penalties from
	misaligned accesses.  Thanks to Edward Seidl <seidl@janed.com>
	for pointing this out.  */
	andl $0xfffffff0, %esp
	pushl %eax		/* Push garbage because we allocate
				   28 more bytes.  */

	/* Provide the highest stack address to the user code (for stacks
	   which grow downwards).  */
	pushl %esp

	pushl %edx		/* Push address of the shared library
				   termination function.  */

#ifdef PIC
	/* Load PIC register.  */
	call 1f
	addl $_GLOBAL_OFFSET_TABLE_, %ebx

	/* This used to be the addresses of .fini and .init.  */
	pushl $0
	pushl $0

	pushl %ecx		/* Push second argument: argv.  */
	pushl %esi		/* Push first argument: argc.  */

# ifdef SHARED
	pushl main@GOT(%ebx)
# else
	/* Avoid relocation in static PIE since _start is called before
	   it is relocated.  This also avoid rely on linker optimization to
	   transform 'movl main@GOT(%ebx), %eax' to 'leal main@GOTOFF(%ebx)'
	   if main is defined locally.  */
	leal __wrap_main@GOTOFF(%ebx), %eax
	pushl %eax
# endif

	/* Call the user's main function, and exit with its value.
	   But let the libc call main.    */
	call __libc_start_main@PLT
#else
	/* This used to be the addresses of .fini and .init.  */
	pushl $0
	pushl $0

	pushl %ecx		/* Push second argument: argv.  */
	pushl %esi		/* Push first argument: argc.  */

	pushl $main

	/* Call the user's main function, and exit with its value.
	   But let the libc call main.    */
	call __libc_start_main
#endif

	hlt			/* Crash if somehow `exit' does return.  */

#ifdef PIC
1:	movl	(%esp), %ebx
	ret
#endif

#if defined PIC && !defined SHARED
__wrap_main:
	jmp	main@PLT
#endif
END (_start)
```

简化之下，其调用逻辑如下：

```c
void _start() {
    %ebp = 0;
    int argc = pop from stack
    char **argv = top of stack
    __libc_start_main(main, argc, argv, __libc_csu_init, __libc_csu_fini, edx, top of stack);
}
```

对于被调用的方法，`__libc_start_main`，其源码逻辑很长，仅保留一些关键调用分析

`__libc_start_main`包含7个入参，main由第一个入参传入，然后是argc、argv，其中argv还包含了环境变量表。接下来的三个指针分别是：

* init：main调用前的初始化
* finit：main结束后的收尾
* rtld_fini：动态加载相关的收尾（rtld runtime loader）

最后的stack_end标明了栈低的位置，即最高的栈地址

然后来看 __libc_start_main 的一个实现：

```cpp
// https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/csu/libc-start.c
/* Note: The init and fini parameters are no longer used.  fini is
   completely unused, init is still called if not NULL, but the
   current startup code always passes NULL.  (In the future, it would
   be possible to use fini to pass a version code if init is NULL, to
   indicate the link-time glibc without introducing a hard
   incompatibility for new programs with older glibc versions.)

   For dynamically linked executables, the dynamic segment is used to
   locate constructors and destructors.  For statically linked
   executables, the relevant symbols are access directly.  */
STATIC int
LIBC_START_MAIN (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
		 int argc, char **argv,
		 __typeof (main) init,
		 void (*fini) (void),
		 void (*rtld_fini) (void), void *stack_end)
{
#ifndef SHARED
  char **ev = &argv[argc + 1];

  __environ = ev;   // __environ 被设置为环境变量数组的起始地址

  /* Store the lowest stack address.  This is done in ld.so if this is
     the code for the DSO.  */
  __libc_stack_end = stack_end;     // __libc_stack_end 存储栈的结束地址

# ifdef HAVE_AUX_VECTOR
    ...
# endif 
    // 共享库相关：可调参数初始化、CPU 特性初始化、静态位置无关代码（PIE）重定位、指令重定位、线程本地存储（TLS）设置、栈保护和指针保护等
#endif /* !SHARED  */

  // 如果动态链接器有析构函数
  if (__glibc_likely (rtld_fini != NULL))
    __cxa_atexit ((void (*) (void *)) rtld_fini, NULL, NULL);

#ifndef SHARED
  // 非共享库情况下的初始化：执行早期初始化、C 库的初始化、注册静态链接程序的析构函数，并进行安全检查（如检查标准文件描述符是否打开）
#endif /* !SHARED */

  /* Call the initializer of the program, if any.  */
#ifdef SHARED
    ...
#else /* !SHARED */
    ...
#endif
  // 调用用户的 main 函数
  __libc_start_call_main (main, argc, argv MAIN_AUXVEC_PARAM);
}
```

```cpp
// https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/sysdeps/generic/libc_start_call_main.h
_Noreturn static __always_inline void
__libc_start_call_main (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
                        int argc, char **argv MAIN_AUXVEC_DECL)
{
  exit (main (argc, argv, __environ MAIN_AUXVEC_PARAM));
}
```

```cpp
// https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/stdlib/exit.c
void
exit (int status)
{
  __run_exit_handlers (status, &__exit_funcs, true, true);
}
```

可以看到即便 `main` 返回了， `exit` 也会被调用。 `exit` 是程序退出的必经之路，因此把 `atexit` 注册的退出相关函数交给 `exit` 来完成时万无一失的。

### MSVC CRT入口函数

略

### 运行库与I/O

程序启动时一个必不可少的动作就是IO初始化，IO初始化函数会在用户空间建立 `stdin` `stdout` `stderr` 及其对应的 FILE 结构，使得程序进入 `main` 之后可以直接使用 `sprintf` 等函数。

## C/C++运行库

C运行库（CTR）逻辑组件：

* 启动和退出
* 标准函数
* IO
* 堆
* 语言实现的特殊功能
* 调试逻辑

C语言标准库：标准输入输出、文件操作、字符操作、字符串操作、数学函数、资源管理、格式转换、时间和日期、断言、类型常数、变长参数、非局部跳转

一个变长参数的简单实现：

```cpp
#define va_list char*
#define va_start(ap, arg) (ap=(va_list)&arg+sizeof(arg))
#define va_arg(ap, T) (*(T*)((ap+=sizeof(T)) - sizeof(T)))
#define va_end(ap) (ap=(va_list)0)

// 计算不定数量整数的和
int sum(int count, ...) {
    va_list args;
    va_start(args, count);

    int result = 0;
    for (int i = 0; i < count; ++i) {
        result += va_arg(args, int);
    }

    va_end(args);
    return result;
}
```

**glibc 启动文件**: 在 Linux 系统中，程序的启动依赖于一系列的启动文件，这些文件负责初始化运行时环境并调用程序的 main 函数。

`crt1.o`：包含程序的入口点 _start，这是程序执行的起点。_start 负责初始化程序的运行时环境，包括设置栈、传递命令行参数和环境变量，并调用 `__libc_start_main`

`crti.o`：提供初始化代码，用于支持 .init 段的执行。在程序启动时，crti.o 中的代码会帮助初始化全局变量和静态对象。

`crtn.o`：提供清理代码，用于支持 .fini 段的执行。在程序退出时，crtn.o 中的代码会帮助清理全局变量和静态对象。

以下是程序启动时的调用顺序：

1. `_start` 程序的入口点，由 crt1.o 提供。它负责初始化运行时环境并调用 __libc_start_main。
2. `__libc_start_main` glibc 提供的启动例程，负责以下初始化工作：
   1. 初始化标准 I/O 流、国际化支持、锁机制等。
   2. 调用 .init 段中的初始化函数。
   3. 调用用户定义的 main 函数。
3. `main` 用户编写的程序主体函数。执行完成后返回一个退出状态码。
`exit`
   1. main 函数返回后，__libc_start_main 调用 exit 函数。
   2. exit 负责调用通过 atexit 或 on_exit 注册的清理函数。
   3. 调用 .fini 段中的清理函数。
   4. 清理动态链接器资源（rtld_fini）。
   5. 关闭所有标准 I/O 流，清理临时文件。

在标准的 Linux 平台上，链接器会按照以下顺序链接这些运行库文件：

`ld crt1.o crti.o [user_objects] [system_libraries] crtn.o`

* `crt1.o` 包含程序的入口函数 `_start`，负责调用 `__libc_start_main` 初始化 `glibc` 并调用 `main` 函数。
* `crti.o` 和 `crtn.o` 提供 `.init` 和 `.fini` 段的辅助代码，确保初始化和清理函数能够正确执行

**Attention**：由于 `.init` 和  `.fini` 的特殊性，一些监控程序性能、调试的工具利用这一特性进行一些初始化和反初始化的工作。此外也可以使用 `__attribute((setcion(".init")))` 将函数放在`.init`段内。但要注意的是普通函数放在`.init`段内可能破坏其结构，因为函数的返回指令会使` _init()` 提前返回，因此不许使用汇编之类，不然编译器对这些新增的函数产生 `ret` 指令。

## 运行库与多线程

### CRT的困扰和改进

早期的C/C++标准（C++03，C89，C99）对多线程只字不提。虽然主流CTR都提供了线程相关的功能：比如MSVC CRT提供了 _beginthread() _endthread()；Linux下，glibc提供了可选的线程库pthread（POSIX Thread）。但这些都不属于标准的运行库，它们实质是平台相关的。

这就导致，早期CRT在设计时未考虑多线程运行环境，存在很多线程不安全的场景：`errno`、`strtok`、`malloc`、`new`、`free`、`delete`、`exception`、`printf`、`signal`。只有一些可重入函数，是线程安全的：`ctype.h` `string.h` `math.h` `stdlib.h` `stdarg.h`。

改进的方式则包括：使用TLS、加锁、改进函数的调用方式

### TLS的实现

TODO @zjy 差资料写个文章 （隐式&显式TLS）

## C++全局构造与析构

### glibc全局构造与析构

> [Global Constructors and Destructors in C++](https://ftp.math.utah.edu/u/ma/hohn/linux/misc/elf/node4.html)

`_start` -> `__libc_start_main` -> `__libc_csu_init` -> `_init` -> `__do_global_ctors_aux`

__do_global_ctors_aux 的代码通常位于 GCC 提供的 crtstuff.c 文件中，它的作用是从 .ctors 段中依次调用所有全局构造函数。

```cpp
static void __attribute__((used))
__do_global_ctors_aux (void)
{
    func_ptr *p;
    for (p = __CTOR_END__ - 1; *p != (func_ptr) -1; p--)
        (*p) ();
}
```

GCC 编译器为每个编译单元生成一个特殊的构造函数（如 `_GLOBAL__I_<name>`），负责初始化该编译单元中的全局对象。编译器将这些特殊函数的指针放入 `.ctors` 段中，链接器会将所有目标文件的 `.ctors` 段合并。

`.ctors` 段是一个函数指针数组，其中存放了所有全局对象的构造函数指针。`__CTOR_END__` 是 `.ctors` 段的末尾指针，`__CTOR_LIST__` 是起始指针。

`__do_global_ctors_aux` 从 `.ctors` 段的末尾向前遍历，依次调用每个构造函数。这种顺序确保了构造函数的调用顺序与它们在 `.ctors` 段中的存储顺序相反。

全局析构函数的调用顺序与它们的注册顺序相反。在 C++ 中，全局对象的析构函数会在程序退出时被调用，这通常通过 `std::exit` 或 `main` 函数返回触发。

`glibc` 使用 `__cxa_atexit` 函数来注册全局析构函数。`__cxa_atexit` 是 `Itanium C++ ABI` 中用于注册析构函数的机制，它比标准的 `atexit` 更灵活，支持动态库的卸载。

```c
// func 是析构函数的指针。
// obj 是析构函数作用的对象。
// dso_handle 是动态共享对象（DSO）的句柄，用于标识析构函数所属的模块。
int __cxa_atexit(void (*func)(void*), void* obj, void* dso_handle);
```

在 glibc 中，atexit 是通过调用 __cxa_atexit 实现的：

```c
// 其中，__dso_handle 是 glibc 提供的特殊符号，用于标识当前模块。
atexit(func) => __cxa_atexit((void (*)(void*))func, NULL, __dso_handle);
```

全局析构函数的调用路径如下：

1. 程序退出：当 `main` 函数返回或调用 `std::exit` 时，程序开始退出流程。
2. `__libc_csu_fini`：exit 函数会调用 `__libc_csu_fini`，这是由编译器生成的函数，用于调用所有全局析构函数。
3. `_libc_csu_fini` 会调用 `_fini`，进而触发 `__do_global_dtors_aux`。
1. `__do_global_dtors_aux` 遍历 `.dtors` 段中的析构函数指针数组，依次调用每个析构函数。

在动态库（如通过 dlopen 加载的库）中，析构函数的注册和调用机制也通过 `__cxa_atexit` 和 `__cxa_finalize` 实现：当动态库被卸载时（如通过 dlclose），`__cxa_finalize` 会被调用，它会遍历 `.fini_array` 段并调用对应的析构函数。

## fread实现

```c
// https://elixir.bootlin.com/glibc/glibc-2.41/source/libio/iofread.c
size_t
_IO_fread (void *buf, size_t size, size_t count, FILE *fp)
{
  size_t bytes_requested = size * count;
  size_t bytes_read;
  // 检查文件流是否有效。如果文件流无效或请求读取的字节数为 0，则直接返回
  CHECK_FILE (fp, 0);   
  if (bytes_requested == 0)
    return 0;
  _IO_acquire_lock (fp);
  // _IO_sgetn 是一个中间函数，它最终调用 _IO_file_xsgetn 来完成实际的读取操作
  bytes_read = _IO_sgetn (fp, (char *) buf, bytes_requested);
  _IO_release_lock (fp);
  return bytes_requested == bytes_read ? count : bytes_read / size;
}
```

```c
size_t
_IO_sgetn (FILE *fp, void *data, size_t n)
{
  /* FIXME handle putback buffer here! */
  return _IO_XSGETN (fp, data, n);
}

#define _IO_XSGETN(FP, DATA, N) JUMP2 (__xsgetn, FP, DATA, N)

// https://elixir.bootlin.com/glibc/glibc-2.41/source/libio/vtables.c#L150
const struct _IO_jump_t __io_vtables[] attribute_relro =
{
    ...
    /* _IO_file_jumps  */
    [IO_FILE_JUMPS] = {
        ...
        JUMP_INIT (xsgetn, _IO_file_xsgetn),
        ...
    },
    ...
}
```

`_IO_file_xsgetn` 检查文件流是否分配了缓冲区。如果没有缓冲区，则调用 `_IO_doallocbuf` 分配一个:

* 如果缓冲区中有足够的数据，则直接从缓冲区复制到目标缓冲区。
* 如果缓冲区数据不足，会尝试通过 `_IO_SYSREAD`（底层系统调用）直接从文件中读取数据。
* 如果需要读取的字节数小于缓冲区大小，则调用 `__underflow` 刷新缓冲区。
