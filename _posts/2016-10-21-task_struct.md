---
layout: post
title: 进程描述符
category : basictheory
author: Max
tags : [task_struct, linux]
---




<item>
<title>进程描述符</title>
<content:encoded>

若要理解进程如何被内核创建，需要先知道进程在内核中被抽象成了什么——进程描述符（task_struct）。

Linux内核通过task_struct结构体来管理进程，这个结构体包含了一个进程所需的所有信息，是一个相当复杂的数据结构。它定义在<a href="http://lxr.free-electrons.com/source/include/linux/sched.h">include/linux/sched.h</a>文件中。

此处挑选一些常用信息进行说明。

<h2>1. 进程状态</h2>

<pre><code>volatile long state;    /* -1 unrunnable, 0 runnable, &gt;0 stopped */
</code></pre>

state域能够取5个互为排斥的值（通俗一点就是这五个值任意两个不能一起使用，只能单独使用）。系统中的每个进程都必然处于以上所列进程状态中的一种。

<table>
<thead>
<tr>
<th>状态</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>TASK_RUNNING</td>
<td>表示进程要么正在执行，要么正要准备执行（已经就绪），正在等待cpu时间片的调度</td>
</tr>
<tr>
<td>TASK_INTERRUPTIBLE</td>
<td>进程因为等待一些条件而被挂起（阻塞）而所处的状态。这些条件主要包括：硬中断、资源、一些信号……，一旦等待的条件成立，进程就会从该状态（阻塞）迅速转化成为就绪状态TASK_RUNNING</td>
</tr>
<tr>
<td>TASK_UNINTERRUPTIBLE</td>
<td>意义与TASK_INTERRUPTIBLE类似，除了不能通过接受一个信号来唤醒以外，对于处于TASK_UNINTERRUPIBLE状态的进程，哪怕我们传递一个信号或者有一个外部中断都不能唤醒他们。只有它所等待的资源可用的时候，他才会被唤醒。这个标志很少用，但是并不代表没有任何用处，其实他的作用非常大，特别是对于驱动刺探相关的硬件过程很重要，这个刺探过程不能被一些其他的东西给中断，否则就会让进城进入不可预测的状态</td>
</tr>
<tr>
<td>TASK_STOPPED</td>
<td>进程被停止执行，当进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后就会进入该状态</td>
</tr>
<tr>
<td>TASK_TRACED</td>
<td>表示进程被debugger等进程监视，进程执行被调试程序所停止，当一个进程被另外的进程所监视，每一个信号都会让进城进入该状态</td>
</tr>
</tbody>
</table>

其实还有两个附加的进程状态既可以被添加到state域中，又可以被添加到exit_state域中。只有当进程终止的时候，才会达到这两种状态.

<pre><code>/* task state */
int exit_state;
int exit_code, exit_signal;
</code></pre>

<table>
<thead>
<tr>
<th>状态</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>EXIT_ZOMBIE</td>
<td>进程的执行被终止，但是其父进程还没有使用wait()等系统调用来获知它的终止信息，此时进程成为僵尸进程</td>
</tr>
<tr>
<td>EXIT_DEAD</td>
<td>进程的最终状态</td>
</tr>
</tbody>
</table>

<h2>2. 进程标识符（PID）</h2>

<pre><code>pid_t pid;  
pid_t tgid;  
</code></pre>

Unix系统通过pid来标识进程，linux把不同的pid与系统中每个进程或轻量级进程关联，而unix程序员希望同一组线程具有共同的pid，遵照这个标准Linux引入线程组的概念。一个线程组所有线程与领头线程具有相同的pid，存入tgid字段，getpid()返回当前进程的tgid值而不是pid的值。

在CONFIG_BASE_SMALL配置为0的情况下，PID的取值范围是0到32767，即系统中的进程数最大为32768个（64位体系可扩大至4194303）。

<h2>3. 进程内核栈</h2>

<pre><code>void *stack;  
</code></pre>

对每个进程，Linux内核都把两个不同的数据结构紧凑的存放在一个单独为进程分配的内存区域中，一个是内核态的进程堆栈，另一个是紧挨着进程描述符的小数据结构thread_info，叫做线程描述符。

<h2>4. 进程标记</h2>

<pre><code>unsigned int flags; /* per process flags, defined below */  
</code></pre>

反应进程状态的信息，但不是运行状态，用于内核识别进程当前的状态，以备下一步操作。

<h2>5. 表示进程亲属关系的成员</h2>

<pre><code>/*
* pointers to (original) parent process, youngest child, younger sibling,
* older sibling, respectively.  (p-&gt;father can be replaced with
* p-&gt;real_parent-&gt;pid)
*/
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
/*
* children/sibling forms the list of my natural children
*/
struct list_head children;      /* list of my children */
struct list_head sibling;       /* linkage in my parent's children list */
struct task_struct *group_leader;       /* threadgroup leader */
</code></pre>

在Linux系统中，所有进程之间都有着直接或间接地联系，每个进程都有其父进程，也可能有零个或多个子进程。拥有同一父进程的所有进程具有兄弟关系。

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>real_parent</td>
<td>指向其父进程，如果创建它的父进程不再存在，则指向PID为1的init进程</td>
</tr>
<tr>
<td>parent</td>
<td>指向其父进程，当它终止时，必须向它的父进程发送信号。它的值通常与real_parent相同</td>
</tr>
<tr>
<td>children</td>
<td>表示链表的头部，链表中的所有元素都是它的子进程</td>
</tr>
<tr>
<td>sibling</td>
<td>用于把当前进程插入到兄弟链表中</td>
</tr>
<tr>
<td>group_leader</td>
<td>指向其所在进程组的领头进程</td>
</tr>
</tbody>
</table>

<h2>6. ptrace系统调用</h2>

<pre><code>unsigned int ptrace;  
struct list_head ptraced;  
struct list_head ptrace_entry;  
unsigned long ptrace_message;  
siginfo_t *last_siginfo; /* For ptrace use.  */  
</code></pre>

Ptrace 提供了一种父进程可以控制子进程运行，并可以检查和改变它的核心image。

它主要用于实现断点调试。一个被跟踪的进程运行中，直到发生一个信号。则进程被中止，并且通知其父进程。在进程中止的状态下，进程的内存空间可以被读写。父进程还可以使子进程继续执行，并选择是否是否忽略引起中止的信号。

成员ptrace被设置为0时表示不需要被跟踪。

<h2>7. 优先级</h2>

<pre><code>int prio, static_prio, normal_prio;
unsigned int rt_priority;
</code></pre>

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>static_prio</td>
<td>用于保存静态优先级，可以通过nice系统调用来进行修改</td>
</tr>
<tr>
<td>rt_priority</td>
<td>用于保存实时优先级</td>
</tr>
<tr>
<td>normal_prio</td>
<td>的值取决于静态优先级和调度策略</td>
</tr>
<tr>
<td>prio</td>
<td>用于保存动态优先级</td>
</tr>
</tbody>
</table>

实时优先级范围是0到MAX_RT_PRIO-1（即99），而普通进程的静态优先级范围是从MAX_RT_PRIO到MAX_PRIO-1（即100到139）。值越大静态优先级越低。

<h2>8. 进程地址空间</h2>

<pre><code>/*  http://lxr.free-electrons.com/source/include/linux/sched.h?V=4.5#L1453 */
struct mm_struct *mm, *active_mm;
/* per-thread vma caching */
u32 vmacache_seqnum;
struct vm_area_struct *vmacache[VMACACHE_SIZE];
#if defined(SPLIT_RSS_COUNTING)
struct task_rss_stat    rss_stat;
#endif

/*  http://lxr.free-electrons.com/source/include/linux/sched.h?V=4.5#L1484  */
#ifdef CONFIG_COMPAT_BRK
unsigned brk_randomized:1;
#endif
</code></pre>

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>mm</td>
<td>进程所拥有的用户空间内存描述符，内核线程无的mm为NULL</td>
</tr>
<tr>
<td>active_mm</td>
<td>active_mm指向进程运行时所使用的内存描述符， 对于普通进程而言，这两个指针变量的值相同。但是内核线程kernel thread是没有进程地址空间的，所以内核线程的tsk->mm域是空（NULL）。但是内核必须知道用户空间包含了什么，因此它的active_mm成员被初始化为前一个运行进程的active_mm值。</td>
</tr>
<tr>
<td>brk_randomized</td>
<td>用来确定对随机堆内存的探测。参见LKML上的介绍</td>
</tr>
<tr>
<td>rss_stat</td>
<td>用来记录缓冲信息</td>
</tr>
</tbody>
</table>

因此如果当前内核线程被调度之前运行的也是另外一个内核线程时候，那么其mm和avtive_mm都是NULL。

<h2>9. 判断标志</h2>

<pre><code>int exit_code, exit_signal;
int pdeath_signal;  /*  The signal sent when the parent dies  */
unsigned long jobctl;   /* JOBCTL_*, siglock protected */

/* Used for emulating ABI behavior of previous Linux versions */
unsigned int personality;

/* scheduler bits, serialized by scheduler locks */
unsigned sched_reset_on_fork:1;
unsigned sched_contributes_to_load:1;
unsigned sched_migrated:1;
unsigned :0; /* force alignment to the next boundary */

/* unserialized, strictly 'current' */
unsigned in_execve:1; /* bit to tell LSMs we're in execve */
unsigned in_iowait:1;
</code></pre>

字段|描述
|:---|:---
exit_code|用于设置进程的终止代号，这个值要么是_exit()或exit_group()系统调用参数（正常终止），要么是由内核提供的一个错误代号（异常终止）。
exit_signal|被置为-1时表示是某个线程组中的一员。只有当线程组的最后一个成员终止时，才会产生一个信号，以通知线程组的领头进程的父进程。
pdeath_signal|用于判断父进程终止时发送信号。
personality|用于处理不同的ABI，参见Linux-Man
in_execve|用于通知LSM是否被do_execve()函数所调用。详见补丁说明，参见LKML
in_iowait|用于判断是否进行iowait计数
sched_reset_on_fork|用于判断是否恢复默认的优先级或调度策略

<h2>10. 时间</h2>

<pre><code>cputime_t utime, stime, utimescaled, stimescaled;
cputime_t gtime;

u64 start_time;         /* monotonic time in nsec */
u64 real_start_time;    /* boot based time in nsec */

</code></pre>

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>utime/stime</td>
<td>用于记录进程在用户态/内核态下所经过的节拍数（定时器）</td>
</tr>
<tr>
<td>utimescaled/stimescaled</td>
<td>用于记录进程在用户态/内核态的运行时间，但它们以处理器的频率为刻度</td>
</tr>
<tr>
<td>gtime</td>
<td>以节拍计数的虚拟机运行时间（guest time）</td>
</tr>
<tr>
<td>start_time/real_start_time</td>
<td>进程创建时间，real_start_time还包含了进程睡眠时间，常用于/proc/pid/stat</td>
</tr>
</tbody>
</table>

<h2>11. 信号处理</h2>

<pre><code>/* signal handlers */
struct signal_struct *signal;
struct sighand_struct *sighand;

sigset_t blocked, real_blocked;
sigset_t saved_sigmask; /* restored if set_restore_sigmask() was used */
struct sigpending pending;

unsigned long sas_ss_sp;
size_t sas_ss_size;
</code></pre>

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>signal</td>
<td>指向进程的信号描述符</td>
</tr>
<tr>
<td>sighand</td>
<td>指向进程的信号处理程序描述符</td>
</tr>
<tr>
<td>blocked</td>
<td>表示被阻塞信号的掩码，real_blocked表示临时掩码</td>
</tr>
<tr>
<td>pending</td>
<td>存放私有挂起信号的数据结构</td>
</tr>
<tr>
<td>sas_ss_sp</td>
<td>是信号处理程序备用堆栈的地址，sas_ss_size表示堆栈的大小</td>
</tr>
</tbody>
</table>

<h2>12. 其它</h2>

待补充。

</content:encoded>

</item>
