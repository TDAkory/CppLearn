# 全局变量初始化顺序

- [全局变量初始化顺序](#全局变量初始化顺序)
  - [在程序启动相关的文件中， `_start` 是定义在 `crt1.o` 中的](#在程序启动相关的文件中-_start-是定义在-crt1o-中的)
  - [`__libc_start_main`](#__libc_start_main)
  - [`__libc_csu_init`](#__libc_csu_init)


main之前，与初始化相关的关键流程：

* _dl_init
  * .preinit_array
  * _start
    * __libc_start_main
      * __libc_csu_init
        * _init
        * .init_array
          * _GLOBAL__sub_I_
            * __static_initialization_and_destruction_0
              * contructors
      * main 

* `-nostdlib` Do not use the standard system startup files or libraries when linking.

## 在程序启动相关的文件中， `_start` 是定义在 `crt1.o` 中的

> https://elixir.bootlin.com/glibc/glibc-2.28/source/sysdeps/x86_64/start.S#L58

```S
ENTRY (_start)
	/* Clearing frame pointer is insufficient, use CFI.  */
	cfi_undefined (rip)
	/* Clear the frame pointer.  The ABI suggests this be done, to mark
	   the outermost frame obviously.  */
	xorl %ebp, %ebp

	/* Extract the arguments as encoded on the stack and set up
	   the arguments for __libc_start_main (int (*main) (int, char **, char **),
		   int argc, char *argv,
		   void (*init) (void), void (*fini) (void),
		   void (*rtld_fini) (void), void *stack_end).
	   The arguments are passed via registers and on the stack:
	main:		%rdi
	argc:		%rsi
	argv:		%rdx
	init:		%rcx
	fini:		%r8
	rtld_fini:	%r9
	stack_end:	stack.	*/

	mov %RDX_LP, %R9_LP	/* Address of the shared library termination
				   function.  */
#ifdef __ILP32__
	mov (%rsp), %esi	/* Simulate popping 4-byte argument count.  */
	add $4, %esp
#else
	popq %rsi		/* Pop the argument count.  */
#endif
	/* argv starts just at the current stack top.  */
	mov %RSP_LP, %RDX_LP
	/* Align the stack to a 16 byte boundary to follow the ABI.  */
	and  $~15, %RSP_LP

	/* Push garbage because we push 8 more bytes.  */
	pushq %rax

	/* Provide the highest stack address to the user code (for stacks
	   which grow downwards).  */
	pushq %rsp

#ifdef PIC
	/* Pass address of our own entry points to .fini and .init.  */
	mov __libc_csu_fini@GOTPCREL(%rip), %R8_LP
	mov __libc_csu_init@GOTPCREL(%rip), %RCX_LP

	mov main@GOTPCREL(%rip), %RDI_LP
#else
	/* Pass address of our own entry points to .fini and .init.  */
	mov $__libc_csu_fini, %R8_LP
	mov $__libc_csu_init, %RCX_LP

	mov $main, %RDI_LP
#endif

	/* Call the user's main function, and exit with its value.
	   But let the libc call main.  Since __libc_start_main in
	   libc.so is called very early, lazy binding isn't relevant
	   here.  Use indirect branch via GOT to avoid extra branch
	   to PLT slot.  In case of static executable, ld in binutils
	   2.26 or above can convert indirect branch into direct
	   branch.  */
	call *__libc_start_main@GOTPCREL(%rip)

	hlt			/* Crash if somehow `exit' does return.	 */
END (_start)
```

## `__libc_start_main`

在调用 main 之前， 会调用全局构造 init， 而这里的 init 就是 __libc_csu_init

> https://elixir.bootlin.com/glibc/glibc-2.28/source/csu/libc-start.c#L129

```S
// 简化之后
STATIC int
__libc_start_main (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
         int argc, char **argv,
         __typeof (main) init,
         void (*fini) (void),
         void (*rtld_fini) (void), void *stack_end)
{
  /* Result of the 'main' function.  */
  int result;

  char **ev = &argv[argc + 1]; 

  __environ = ev;       // 存储环境变量信息

  /* Register the destructor of the program, if any.  */
  if (fini)
    __cxa_atexit ((void (*) (void *)) fini, NULL, NULL);    // 注册全局析构

  if (init)
    (*init) (argc, argv, __environ MAIN_AUXVEC_PARAM);

  result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);

  exit (result);
}
```

## `__libc_csu_init`

> https://elixir.bootlin.com/glibc/glibc-2.28/source/csu/elf-init.c#L67

```S
/* These functions are passed to __libc_start_main by the startup code.
   These get statically linked into each program.  For dynamically linked
   programs, this module will come from libc_nonshared.a and differs from
   the libc.a module in that it does not call the preinit array.  */

void
__libc_csu_init (int argc, char **argv, char **envp)
{
  /* For dynamically linked executables the preinit array is executed by
     the dynamic linker (before initializing any shared object).  */

#ifndef LIBC_NONSHARED
  /* For static executables, preinit happens right before init.  */
  {
    const size_t size = __preinit_array_end - __preinit_array_start;
    size_t i;
    for (i = 0; i < size; i++)
      (*__preinit_array_start [i]) (argc, argv, envp);
  }
#endif

#ifndef NO_INITFINI
  _init ();
#endif

  const size_t size = __init_array_end - __init_array_start;
  for (size_t i = 0; i < size; i++)
      (*__init_array_start [i]) (argc, argv, envp);
}
```