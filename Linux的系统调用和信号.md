---
title: Linux的系统调用和信号

categories:

  - Linux

tags:

  - 异步

  - 异常

excerpt: 本文简单的介绍了在Linux中常见的同步手段
---
<!-- more -->

## 系统调用

系统调用构成了操作系统为用户态任务提供硬件资源访问能力的接口，而广泛应用的 POSIX API 则对这些系统调用的行为和形式进行了标准化定义。

### 系统调用的整个过程会发生什么？

- 首先，用户态进程执行到系统调用的函数的时候，系统调用所需要传递的参数都已被放入寄存器中。此时，通过特殊的指令，如syscall，ecall触发系统调用陷入内核态，将返回地址以及系统调用号放入寄存器中，同时跳转到中断向量表设定的系统调用入口
- 在进行系统调用的处理过程中，根据寄存器中保留的系统调用号，调用对应的系统调用，同时保存进程的上下文以便恢复后续的执行
- 从寄存器中读取具体系统调用所需要的参数，然后交给内核处理具体的系统调用
- 处理结束之后，恢复上下文，并且将结果写入寄存器中，同时调用特殊的返回指令返回用户态，并恢复先前的执行
- 至此，进程恢复用户态，成功的完成了一次系统调用

### 系统调用的参数是怎么传递的？

- 一般情况下，参数的传递是通过寄存器实现的。
- 但是如果所需要传递的参数过多，那么需要使用一片缓冲区，将指向缓冲区的指针传递。在内核态的时候，使用访问用户态地址空间的接口从用户态读入参数。

### Linux内核关于加速系统调用的一些设计

有一些系统调用，并不涉及到内核态下的写，比如获取时钟，当前执行的CPU编号。对于这些只读的系统调用，如果采用上述的系统调用方法，那么是耗时的。这里引入了 `VDSO (visual dynamic share object)`的概念，在进程的用户态地址空间映射了内核的代码，来避免了**用户态→内核态→用户态**的切换开销。

## 信号

信号是一个很简短的消息，通常用一个数字去标识。在编写程序中，通过一组以SIG开头的宏来标识信号。信号在UNIX中即被设计出来，用来完成进程间的通讯，和内核通知进程发生的事。

### 信号的处理流程

#### 数据结构关系回顾

任务中定义 `task_struct`中有关信号的定义的部分(截取部分)：

```c
	/* Signal handlers: */
	struct signal_struct		*signal;
	struct sighand_struct __rcu		*sighand;
	sigset_t			blocked;
	sigset_t			real_blocked;
	/* Restored if set_restore_sigmask() was used: */
	sigset_t			saved_sigmask;
	struct sigpending		pending;
	unsigned long			sas_ss_sp;
	size_t				sas_ss_size;
	unsigned int			sas_ss_flags;

	struct callback_head		*task_works;
```

其中，`struct sigpending		pending;`是每个任务级的私有的信号 `pending`队列

然后是 `signal_struct`的具体定义（截取部分)：

```c
struct signal_struct {
	refcount_t		sigcnt;
	atomic_t		live;
	int			nr_threads;
	int			quick_threads;
	struct list_head	thread_head;

	wait_queue_head_t	wait_chldexit;	/* for wait4() */

	/* current thread group signal load-balancing target: */
	struct task_struct	*curr_target;

	/* shared signal handling: */
	struct sigpending	shared_pending;
...
}
```

` struct signal_struct *signal`是保存**线程组级**的信号处理状态（共享信号队列、线程数、wait4() 等状态）。整个进程的多个线程共享同一个  `signal_struct`。

![1754636746266](image/Linux的系统调用和信号/1754636746266.png "信号结构体示意")

#### 信号发送到任务的过程

- 首先判断信号是发给 **整个线程组** 还是某个 **特定任务**

  * 发送给整个线程组 → 加入 `task_struct->signal_struct->shared_pending`
  * 任务级信号 → 加入 `task_struct->pending`

* 检查目标线程的  **blocked mask** ：

  * 如果该信号在 `blocked` 中 → 进入挂起队列（pending），暂不递送。
  * 如果不在 `blocked` 中 → 标记 TIF_SIGPENDING，让调度点检查并递送。
* 当线程返回用户态前（`do_signal()` 调用处）或被 `ptrace` 操作时，检查并处理 pending 信号。

  * 当一个线程从内核态要 **切换回用户态** （比如系统调用结束、异常/中断处理完毕）时，内核会检查：

    * 这个线程的 `pending`（私有挂起信号队列）
    * 以及 `signal_struct->shared_pending`（线程组挂起信号队列）
  * 如果有未屏蔽的挂起信号，内核会调用 `do_signal()` 来递送它们。
  * 所以信号 **不会在内核态任意时刻打断执行** ，而是在准备返回用户态的“安全点”才执行（除非是致命信号 `SIGKILL` 这种立即生效的）。

#### 任务怎么处理发送的信号

任务必须以三种方式对收到的信号作出应答：

- 忽略
- 执行信号所定义的缺醒操作（如SIGKILL，SIGSTOP是必须执行对应的缺醒操作的)
- 调用相应的信号处理函数捕获处理信号
