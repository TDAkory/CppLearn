# 关于`fork`的一些事

## `fork`、`exec`和`pthread`

[**`fork`**](https://man7.org/linux/man-pages/man2/fork.2.html)通过复制创建一个新的进程。新产生的是子进程，调用返回值为0；发起fork调用的是父进程，调用返回值是子进程的PID。父子进程运行在独立的内存空间上，在fork这个点上，两者拥有相同的内存内容。在fork调用之后，两个进程都继续执行fork函数之后的指令。

子进程是父进程的完全拷贝，包括：进程上下文、进程堆栈、内存信息、打开的文件描述符、信号控制设置、进程优先级、进程组号、当前工作目录、根目录、资源限制、控制终端等。但不包括如下信息：

* 子进程拥有自己的进程号 PID，并且这个PID是全新的，不属于任何进程组
* 子进程的父进程号PPID是父进程的PID
* 子进程不继承父进程的内存锁 mlock mlockall
* 进程资源占用 getrusage 和 cpu计时 times 被重置为零
* 子进程上挂起的信号被初始化为空 sigpending
* 子进程不继承信号调整 semop
...... (详见 man page)

[**`exec`**](https://man7.org/linux/man-pages/man3/exec.3.html) 其实是一组函数，用来替换当前的进程映像。exec函数的函数执行成功后不会返回，因为调用进程的实体，包括代码段，数据段和堆栈等都已经被新的内容取代，只有进程ID等一些表面上的信息仍保持原样。调用失败时，会设置errno并返回-1，然后从原程序的调用点接着往下执行。Exec()本身并没有创建新进程，只是将原进程分配了新的数据段和堆栈段。

[**`pthreads`**](https://man7.org/linux/man-pages/man7/pthreads.7.html)是针对操作POSIX线程定义的一组函数。一个进程可以包含多个线程，这些线程都执行相同的程序（坑你是程序内不同的逻辑部分）。这些线程共享全局内存（数据和堆）、独享栈以及其他一些信息。

* `pthread_t` 是pthread的线程ID
* `pthread_create()`用于创建新的线程
* `pthread_equal()`用于比较两个线程id是否相等
* `pthread_self()` 用于获取当前线程的id
* `pthread_exit()` 线程调用该函数主动退出线程
* `pthread_join()` 用于线程同步，以阻塞的方式等待指定线程结束

## fork safety & signal safety

除了上面的基本理解外，打开`man fork`，还有更进一步的说明：

```vim
Note the following further points:

*  The  child process is created with a single thread—the one that called fork().  The entire virtual address space of the parent is replicated in the child, including the states of mutexes, condition variables, and other pthreads objects; the use of pthread_atfork(3) may be helpful for dealing with problems that this can cause.

*  After a fork() in a multithreaded program, the child can safely call only async-signal-safe functions (see signal-safety(7)) until such time as it calls execve(2).

*  The child inherits copies of the parent's set of open file descriptors.  Each file descriptor in the child refers to the same open file description (see open(2)) as the corresponding file  descriptor in the parent.  This means that the two file descriptors share open file status flags, file offset, and signal-driven I/O attributes (see the description of F_SETOWN and F_SETSIG in fcntl(2)).

*  The child inherits copies of the parent's set of open message queue descriptors (see mq_overview(7)).  Each file descriptor in the child refers to the same  open  message  queue  description as the corresponding file descriptor in the parent.  This means that the two file descriptors share the same flags (mq_flags).

*  The child inherits copies of the parent's set of open directory streams (see opendir(3)).  POSIX.1 says that the corresponding directory streams in the parent and child may share the directory stream positioning; on Linux/glibc they do not.
```

从中我们可以得知

* fork只复制当前线程到子进程，其他的线程在子进程中是不存在的
* 子进程会得到父进程在虚拟地址空间上的拷贝，包括互斥锁、条件变量，这有可能引发死锁
* 利用`pthread_atfork`可以处理部分问题
* fork之后，在调用execve之前，子进程可以调用异步信号安全的函数

Fork Safety 指的是一个程序在执行 fork 后能够安全地继续运行，而不会导致数据竞争、死锁或其他问题。为了确保分叉安全性，通常需要遵循以下准则：

* 避免在 fork 之后使用不是分叉安全的系统调用。 一些系统调用在 fork 后可能不是线程安全的，因为它们可能依赖于进程的全局状态。
* 在 fork 之后立即调用 exec 系统调用。 这可以避免子进程中的线程干扰，因为 exec 会替换子进程的映像，而不是继续执行父进程的代码。
* 在多线程环境中谨慎使用 fork。 如果你的程序是多线程的，最好在 fork 之前将所有线程同步到一个安全点，或者避免在多线程环境中使用 fork。

[**Signal Safety**](https://man7.org/linux/man-pages/man7/signal-safety.7.html) 在linux中则有专门的定义：指在信号处理程序中可以安全调用的函数集合。通常来说，不可重入函数都不是信号安全的。

```shell
NAME
       signal-safety - async-signal-safe functions

DESCRIPTION
       An async-signal-safe function is one that can be safely called from within a signal handler.  Many functions are not async-signal-safe.  In particular, nonreentrant functions are generally unsafe to call from a signal handler.

       ......
```

* 为了避免使用非安全函数导致的问题，信号处理程序应该只调用异步信号安全函数，并且信号处理程序本身应该是可重入的，即它不依赖于全局变量或与主程序共享数据。
* 在执行非信号安全的函数或操作全局数据时，可以通过在主程序中屏蔽信号来避免信号处理程序的干扰。
  
POSIX.1标准规定了一组必须实现为异步信号安全的函数。这些函数包括但不限于abort, accept, access, alarm, bind, clock_gettime, close, connect, execl, execv, _exit, fork, getpid, kill, lseek, mkdir, open, pause, pipe, pthread_kill, read, recv, select, sem_post, send, setsockopt, sigaction, sigprocmask, sleep, socket, stat, time, umask, unlink, waitpid, write等。

man page中还特别指出，GNU C library中关于这个话题的一些案例：

* Before glibc 2.24, execl(3) and execle(3) employed realloc(3) internally and were consequently not async-signal-safe.  This was fixed in glibc 2.24.

* The glibc implementation of aio_suspend(3) is not async-signal-safe because it uses pthread_mutex_lock(3) internally.
