# Runtime Library

## 入口函数和程序初始化

入口函数（Entry Point）

* 操作系统创建进程后，把控制权交给程序的入口，这个入口往往是运行库汇中的某个入口函数
* 入口函数堆运行库和程序运行环境进行初始化：堆、IO、线程、全局变量构造等
* 入口函数完成后，调用main函数，执行程序主体
* main函数完成后，回到入口函数，进行清理工作：全局变量析构、堆销毁、关闭IO等
* 最终进行系统调用结束进程

## glibc的入口函数

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

最后stack_end标明了栈低，即最高的栈地址

```c
// https://github.com/lattera/glibc/blob/master/csu/libc-start.c
# define LIBC_START_MAIN __libc_start_main

LIBC_START_MAIN (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
		 int argc, char **argv,
		 __typeof (main) init,
		 void (*fini) (void),
		 void (*rtld_fini) (void), void *stack_end)
{
  /* Result of the 'main' function.  */
  int result;

  __libc_multiple_libcs = &_dl_starting_up && !_dl_starting_up;

#ifndef SHARED
  // some complex logic for shared lib
#endif /* !SHARED  */

  /* Register the destructor of the dynamic linker if there is any.  */
  if (__glibc_likely (rtld_fini != NULL))
    __cxa_atexit ((void (*) (void *)) rtld_fini, NULL, NULL);

#ifndef SHARED
  /* Call the initializer of the libc.  This is only needed here if we
     are compiling for the static library in which case we haven't
     run the constructors in `_dl_start_user'.  */
  __libc_init_first (argc, argv, __environ);

  /* Register the destructor of the program, if any.  */
  if (fini)
    __cxa_atexit ((void (*) (void *)) fini, NULL, NULL);

  /* Some security at this point.  Prevent starting a SUID binary where
     the standard file descriptors are not opened.  We have to do this
     only for statically linked applications since otherwise the dynamic
     loader did the work already.  */
  if (__builtin_expect (__libc_enable_secure, 0))
    __libc_check_standard_fds ();
#endif

  /* Call the initializer of the program, if any.  */
#ifdef SHARED
  if (__builtin_expect (GLRO(dl_debug_mask) & DL_DEBUG_IMPCALLS, 0))
    GLRO(dl_debug_printf) ("\ninitialize program: %s\n\n", argv[0]);
#endif
  if (init)
    (*init) (argc, argv, __environ MAIN_AUXVEC_PARAM);

#ifdef SHARED
  /* Auditing checkpoint: we have a new object.  */
  if (__glibc_unlikely (GLRO(dl_naudit) > 0))
    {
      struct audit_ifaces *afct = GLRO(dl_audit);
      struct link_map *head = GL(dl_ns)[LM_ID_BASE]._ns_loaded;
      for (unsigned int cnt = 0; cnt < GLRO(dl_naudit); ++cnt)
	{
	  if (afct->preinit != NULL)
	    afct->preinit (&head->l_audit[cnt].cookie);

	  afct = afct->next;
	}
    }
#endif

#ifdef SHARED
  if (__glibc_unlikely (GLRO(dl_debug_mask) & DL_DEBUG_IMPCALLS))
    GLRO(dl_debug_printf) ("\ntransferring control: %s\n\n", argv[0]);
#endif

#ifndef SHARED
  _dl_debug_initialize (0, LM_ID_BASE);
#endif
#ifdef HAVE_CLEANUP_JMP_BUF
  /* Memory for the cancellation buffer.  */
  struct pthread_unwind_buf unwind_buf;

  int not_first_call;
  not_first_call = setjmp ((struct __jmp_buf_tag *) unwind_buf.cancel_jmp_buf);
  if (__glibc_likely (! not_first_call))
    {
      struct pthread *self = THREAD_SELF;

      /* Store old info.  */
      unwind_buf.priv.data.prev = THREAD_GETMEM (self, cleanup_jmp_buf);
      unwind_buf.priv.data.cleanup = THREAD_GETMEM (self, cleanup);

      /* Store the new cleanup handler info.  */
      THREAD_SETMEM (self, cleanup_jmp_buf, &unwind_buf);

      /* Run the program.  */
      result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);
    }
  else
    {
      /* Remove the thread-local data.  */
# ifdef SHARED
      PTHFCT_CALL (ptr__nptl_deallocate_tsd, ());
# else
      extern void __nptl_deallocate_tsd (void) __attribute ((weak));
      __nptl_deallocate_tsd ();
# endif

      /* One less thread.  Decrement the counter.  If it is zero we
	 terminate the entire process.  */
      result = 0;
# ifdef SHARED
      unsigned int *ptr = __libc_pthread_functions.ptr_nthreads;
#  ifdef PTR_DEMANGLE
      PTR_DEMANGLE (ptr);
#  endif
# else
      extern unsigned int __nptl_nthreads __attribute ((weak));
      unsigned int *const ptr = &__nptl_nthreads;
# endif

      if (! atomic_decrement_and_test (ptr))
	/* Not much left to do but to exit the thread, not the process.  */
	__exit_thread ();
    }
#else
  /* Nothing fancy, just call the function.  */
  result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);
#endif

  exit (result);
}
