从一个比较有意思的题开始说起，最近要找工作无意间看到一个关于unix/linux中fork()的面试题：

```text
1 #include<sys/types.h>
  2 #include<stdio.h>
  3 #include<unistd.h>
  4    int main(void)
  5         {
  6          int i;
  7          int buf[100]={1,2,3,4,5,6,7,8,9};
  8          for(i=0;i<2;i++)
  9           {
 10             fork();
 11             printf("+");
 12             //write("/home/pi/code/test_fork/test_fork.txt",buf,8);
 13             write(STDOUT_FILENO,"-",1);
 14           }
 15          return 0;
 16
 17 }
```

题目要求是从上面的代码中确定输出的“+”的数量，我后面加了一个“-”，再确定输出“-”的数量。

先给答案：“+”8次，“-”6次

```text
1 ---++--++-++++
```

上面的这段代码很简单，包含的内容却有很多，有进程产生、系统调用、不带缓冲I/O、标准I/O。

linux中产生一个进程的调用函数过程如下：

fork()---------->sys_fork()-------------->do_fork()---------->copy_process()

fork()、vfork()、_clone()库函数都根据各自需要的参数标志去调用clone()，然后由clone()去调用do_fork()。do_fork()完成了创建中的大部分工作，该函数调用copy_process()函数，

从用户空间调用fork()函数到执行系统调用产生软件中断陷入内核空间，在内核空间执行do_fork()函数，主要是复制父进程的页表、内核栈等，如果要执行子进程代码还要调用exac()函数拷贝硬盘上的代码到位内存上，由于刚创建的子进程没有申请内存，目前和父进程共用父进程的代码段、数据段等，没有存放子进程自己代码段数据段的内存，此时会产生一个缺页异常，为子进程申请内存，同时定制自己的全局描述GDT、局部描述符LDT、任务状态描述符TSS，下面从代码中分析这个过程然后在回答上面为什么“+”是8次，“-”6次。

调用fork()函数执行到了unistd.h中的宏函数syscall0

```text
/* XXX - _foo needs to be __foo, while __NR_bar could be _NR_bar. */
/*
 * Don't remove the .ifnc tests; they are an insurance against
 * any hard-to-spot gcc register allocation bugs.
 */
#define _syscall0(type,name) \
type name(void) \
{ \
  register long __a __asm__ ("r10"); \
  register long __n_ __asm__ ("r9") = (__NR_##name); \
  __asm__ __volatile__ (".ifnc %0%1,$r10$r9\n\t" \
            ".err\n\t" \
            ".endif\n\t" \
            "break 13" \
            : "=r" (__a) \
            : "r" (__n_)); \
  if (__a >= 0) \
     return (type) __a; \
  errno = -__a; \
  return (type) -1; \
}
```

将宏函数展开后变为

```text
1 /* XXX - _foo needs to be __foo, while __NR_bar could be _NR_bar. */
 2 /*
 3  * Don't remove the .ifnc tests; they are an insurance against
 4  * any hard-to-spot gcc register allocation bugs.
 5  */
 7 int fork(void) 
 8 { 
 9   register long __a __asm__ ("r10"); \
10   register long __n_ __asm__ ("r9") = (__NR_##name); \
11   __asm__ __volatile__ (".ifnc %0%1,$r10$r9\n\t" \
12             ".err\n\t" \
13             ".endif\n\t" \
14             "break 13" \
15             : "=r" (__a) \
16             : "r" (__n_)); \
17   if (__a >= 0) \
18      return (type) __a; \
19   errno = -__a; \
20   return (type) -1; \
21 }
```

\##的意思就是宏中的字符直接替换

如果name = fork，那么在宏中_ NR_##name就替换成了 _ NR_fork了。

_ NR _ ##name是系统调用号，##指的是两次宏展开．即用实际的系统调用名字代替"name",然后再把_ NR _ ...展开．如name == ioctl，则为_ NR_ioctl。
上面的汇编目前还是没有怎么弄懂-------
int $0x80 是所有系统调用函数的总入口，fork()是其中之一，“0”(_NR_fork) 意思是将fork在sys_call_table[]中对应的函数编号_NR_fork也就是2，将2传给eax寄存器。这个编号就是sys_fork()函数在sys_call_table中的偏移值，其他的系统调用在sys_call_table均存在偏移值()。
int $0x80 中断返回后，将执行return (type) -1----->展开就是return (int) __a;产生int $0x80软件中断，CPU从3级特权的进程跳到0特权级内核代码中执行。中断使CPU硬件自动将SS、ESP、EFLAGGS、CS、EIP这五个寄存器的值按照这个顺序压人父进程的内核栈，这些压栈的数据将在后续的copy_process()函数中用来初始化进程1的任务状态描述符TSS
CPU自动压栈完成后，跳转到system_call.s中的_system_call处执行，继续将DS、ES、FS、EDX、ECX、EBX压栈(这些压栈仍旧是为了初始化子进程中的任务状态描述符TSS做准备)。最终内核通过刚刚设置的eax的偏移值“2”查询sys_call_table[],知道此次系统调用对应的函数是sys_fork()。跳转到_sys_fork处执行。
注意：一个函数的参数不是由函数定义的，而是由函数定义以外的程序通过压栈的方式“做”出来的，是操作系统底层代码与应用程序代码写作手法的差异之一。我们知道在C语言中函数运行时参数是存在栈中的，根据这个原理操作系统设计者可以将前面程序强行压栈的值作为函数的参数，当调用这个函数时这些值就是函数的参数。

sys_fork函数

```text
asmlinkage int sys_fork(void)
{
#ifndef CONFIG_MMU
    /* fork almost works, enough to trick you into looking elsewhere:-( */
    return -EINVAL;
#else
    return do_fork(SIGCHLD, user_stack(__frame), __frame, 0, NULL, NULL);
#endif
}
```

do_fork函数

```text
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 */
long do_fork(unsigned long clone_flags,
          unsigned long stack_start,
          struct pt_regs *regs,
          unsigned long stack_size,
          int __user *parent_tidptr,
          int __user *child_tidptr)
{
    struct task_struct *p;
    int trace = 0;
    struct pid *pid = alloc_pid();
    long nr;

    if (!pid)
        return -EAGAIN;
    nr = pid->nr;
    if (unlikely(current->ptrace)) {
        trace = fork_traceflag (clone_flags);
        if (trace)
            clone_flags |= CLONE_PTRACE;
    }
dup_task_struct
    p = copy_process(clone_flags, stack_start, regs, stack_size, parent_tidptr, child_tidptr, pid);
    /*
     * Do this prior waking up the new thread - the thread pointer
     * might get invalid after that point, if the thread exits quickly.
     */
    if (!IS_ERR(p)) {
        struct completion vfork;

        if (clone_flags & CLONE_VFORK) {
            p->vfork_done = &vfork;
            init_completion(&vfork);
        }

        if ((p->ptrace & PT_PTRACED) || (clone_flags & CLONE_STOPPED)) {
            /*
             * We'll start up with an immediate SIGSTOP.
             */
            sigaddset(&p->pending.signal, SIGSTOP);
            set_tsk_thread_flag(p, TIF_SIGPENDING);
        }

        if (!(clone_flags & CLONE_STOPPED))
            wake_up_new_task(p, clone_flags);
        else
            p->state = TASK_STOPPED;

        if (unlikely (trace)) {
            current->ptrace_message = nr;
            ptrace_notify ((trace << 8) | SIGTRAP);
        }

        if (clone_flags & CLONE_VFORK) {
            freezer_do_not_count();
            wait_for_completion(&vfork);
            freezer_count();
            if (unlikely (current->ptrace & PT_TRACE_VFORK_DONE)) {
                current->ptrace_message = nr;
                ptrace_notify ((PTRACE_EVENT_VFORK_DONE << 8) | SIGTRAP);
            }
        }
    } else {
        free_pid(pid);
        nr = PTR_ERR(p);
    }
    return nr;
}
```

copy_process函数

```text
/*
 * This creates a new process as a copy of the old one,
 * but does not actually start it yet.
 *
 * It copies the registers, and all the appropriate
 * parts of the process environment (as per the clone
 * flags). The actual kick-off is left to the caller.
 */
static struct task_struct *copy_process(unsigned long clone_flags,
                    unsigned long stack_start,
                    struct pt_regs *regs,
                    unsigned long stack_size,
                    int __user *parent_tidptr,
                    int __user *child_tidptr,
                    struct pid *pid)
{
    int retval;
    struct task_struct *p = NULL;

    if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
        return ERR_PTR(-EINVAL);

    /*
     * Thread groups must share signals as well, and detached threads
     * can only be started up within the thread group.
     */
    if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
        return ERR_PTR(-EINVAL);

    /*
     * Shared signal handlers imply shared VM. By way of the above,
     * thread groups also imply shared VM. Blocking this case allows
     * for various simplifications in other code.
     */
    if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
        return ERR_PTR(-EINVAL);

    retval = security_task_create(clone_flags);
    if (retval)
        goto fork_out;

    retval = -ENOMEM;
    p = dup_task_struct(current);
    if (!p)
        goto fork_out;
sys_fork
    rt_mutex_init_task(p);

#ifdef CONFIG_TRACE_IRQFLAGS
    DEBUG_LOCKS_WARN_ON(!p->hardirqs_enabled);
    DEBUG_LOCKS_WARN_ON(!p->softirqs_enabled);
#endif
    retval = -EAGAIN;
    if (atomic_read(&p->user->processes) >=
            p->signal->rlim[RLIMIT_NPROC].rlim_cur) {
        if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) &&
                p->user != &root_user)
            goto bad_fork_free;
    }

    atomic_inc(&p->user->__count);
    atomic_inc(&p->user->processes);
    get_group_info(p->group_info);

    /*
     * If multiple threads are within copy_process(), then this check
     * triggers too late. This doesn't hurt, the check is only there
     * to stop root fork bombs.
     */
    if (nr_threads >= max_threads)
        goto bad_fork_cleanup_count;

    if (!try_module_get(task_thread_info(p)->exec_domain->module))
        goto bad_fork_cleanup_count;

    if (p->binfmt && !try_module_get(p->binfmt->module))
        goto bad_fork_cleanup_put_domain;

    p->did_exec = 0;
    delayacct_tsk_init(p);    /* Must remain after dup_task_struct() */
    copy_flags(clone_flags, p);
    p->pid = pid_nr(pid);
    retval = -EFAULT;
    if (clone_flags & CLONE_PARENT_SETTID)
        if (put_user(p->pid, parent_tidptr))
            goto bad_fork_cleanup_delays_binfmt;

    INIT_LIST_HEAD(&p->children);
    INIT_LIST_HEAD(&p->sibling);
    p->vfork_done = NULL;
    spin_lock_init(&p->alloc_lock);

    clear_tsk_thread_flag(p, TIF_SIGPENDING);
    init_sigpending(&p->pending);

    p->utime = cputime_zero;
    p->stime = cputime_zero;
     p->sched_time = 0;
#ifdef CONFIG_TASK_XACCT
    p->rchar = 0;        /* I/O counter: bytes read */
    p->wchar = 0;        /* I/O counter: bytes written */
    p->syscr = 0;        /* I/O counter: read syscalls */
    p->syscw = 0;        /* I/O counter: write syscalls */
#endif
    task_io_accounting_init(p);
    acct_clear_integrals(p);

     p->it_virt_expires = cputime_zero;
    p->it_prof_expires = cputime_zero;
     p->it_sched_expires = 0;
     INIT_LIST_HEAD(&p->cpu_timers[0]);
     INIT_LIST_HEAD(&p->cpu_timers[1]);
     INIT_LIST_HEAD(&p->cpu_timers[2]);

    p->lock_depth = -1;        /* -1 = no lock */
    do_posix_clock_monotonic_gettime(&p->start_time);
    p->security = NULL;
    p->io_context = NULL;
    p->io_wait = NULL;
    p->audit_context = NULL;
    cpuset_fork(p);
#ifdef CONFIG_NUMA
     p->mempolicy = mpol_copy(p->mempolicy);
     if (IS_ERR(p->mempolicy)) {
         retval = PTR_ERR(p->mempolicy);
         p->mempolicy = NULL;
         goto bad_fork_cleanup_cpuset;
     }
    mpol_fix_fork_child_flag(p);
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
    p->irq_events = 0;
#ifdef __ARCH_WANT_INTERRUPTS_ON_CTXSW
    p->hardirqs_enabled = 1;
#else
    p->hardirqs_enabled = 0;
#endif
    p->hardirq_enable_ip = 0;
    p->hardirq_enable_event = 0;
    p->hardirq_disable_ip = _THIS_IP_;
    p->hardirq_disable_event = 0;
    p->softirqs_enabled = 1;
    p->softirq_enable_ip = _THIS_IP_;
    p->softirq_enable_event = 0;
    p->softirq_disable_ip = 0;
    p->softirq_disable_event = 0;
    p->hardirq_context = 0;
    p->softirq_context = 0;
#endif
#ifdef CONFIG_LOCKDEP
    p->lockdep_depth = 0; /* no locks held yet */
    p->curr_chain_key = 0;
    p->lockdep_recursion = 0;
#endif

#ifdef CONFIG_DEBUG_MUTEXES
    p->blocked_on = NULL; /* not blocked yet */
#endif

    p->tgid = p->pid;
    if (clone_flags & CLONE_THREAD)
        p->tgid = current->tgid;

    if ((retval = security_task_alloc(p)))
        goto bad_fork_cleanup_policy;
    if ((retval = audit_alloc(p)))
        goto bad_fork_cleanup_security;
    /* copy all the process information */
    if ((retval = copy_semundo(clone_flags, p)))
        goto bad_fork_cleanup_audit;
    if ((retval = copy_files(clone_flags, p)))
        goto bad_fork_cleanup_semundo;
    if ((retval = copy_fs(clone_flags, p)))
        goto bad_fork_cleanup_files;
    if ((retval = copy_sighand(clone_flags, p)))
        goto bad_fork_cleanup_fs;
    if ((retval = copy_signal(clone_flags, p)))
        goto bad_fork_cleanup_sighand;
    if ((retval = copy_mm(clone_flags, p)))
        goto bad_fork_cleanup_signal;
    if ((retval = copy_keys(clone_flags, p)))
        goto bad_fork_cleanup_mm;
    if ((retval = copy_namespaces(clone_flags, p)))
        goto bad_fork_cleanup_keys;
    retval = copy_thread(0, clone_flags, stack_start, stack_size, p, regs);
    if (retval)
        goto bad_fork_cleanup_namespaces;

    p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
    /*
     * Clear TID on mm_release()?
     */
    p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr: NULL;
    p->robust_list = NULL;
#ifdef CONFIG_COMPAT
    p->compat_robust_list = NULL;
#endif
    INIT_LIST_HEAD(&p->pi_state_list);
    p->pi_state_cache = NULL;

    /*
     * sigaltstack should be cleared when sharing the same VM
     */
    if ((clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM)
        p->sas_ss_sp = p->sas_ss_size = 0;

    /*
     * Syscall tracing should be turned off in the child regardless
     * of CLONE_PTRACE.
     */
    clear_tsk_thread_flag(p, TIF_SYSCALL_TRACE);
#ifdef TIF_SYSCALL_EMU
    clear_tsk_thread_flag(p, TIF_SYSCALL_EMU);
#endif

    /* Our parent execution domain becomes current domain
       These must match for thread signalling to apply */
    p->parent_exec_id = p->self_exec_id;

    /* ok, now we should be set up.. */
    p->exit_signal = (clone_flags & CLONE_THREAD) ? -1 : (clone_flags & CSIGNAL);
    p->pdeath_signal = 0;
    p->exit_state = 0;

    /*
     * Ok, make it visible to the rest of the system.
     * We dont wake it up yet.
     */
    p->group_leader = p;
    INIT_LIST_HEAD(&p->thread_group);
    INIT_LIST_HEAD(&p->ptrace_children);
    INIT_LIST_HEAD(&p->ptrace_list);

    /* Perform scheduler related setup. Assign this task to a CPU. */
    sched_fork(p, clone_flags);

    /* Need tasklist lock for parent etc handling! */
    write_lock_irq(&tasklist_lock);

    /* for sys_ioprio_set(IOPRIO_WHO_PGRP) */
    p->ioprio = current->ioprio;

    /*
     * The task hasn't been attached yet, so its cpus_allowed mask will
     * not be changed, nor will its assigned CPU.
     *
     * The cpus_allowed mask of the parent may have changed after it was
     * copied first time - so re-copy it here, then check the child's CPU
     * to ensure it is on a valid CPU (and if not, just force it back to
     * parent's CPU). This avoids alot of nasty races.
     */
    p->cpus_allowed = current->cpus_allowed;
    if (unlikely(!cpu_isset(task_cpu(p), p->cpus_allowed) ||
            !cpu_online(task_cpu(p))))
        set_task_cpu(p, smp_processor_id());

    /* CLONE_PARENT re-uses the old parent */
    if (clone_flags & (CLONE_PARENT|CLONE_THREAD))
        p->real_parent = current->real_parent;
    else
        p->real_parent = current;
    p->parent = p->real_parent;

    spin_lock(&current->sighand->siglock);

    /*
     * Process group and session signals need to be delivered to just the
     * parent before the fork or both the parent and the child after the
     * fork. Restart if a signal comes in before we add the new process to
     * it's process group.
     * A fatal signal pending means that current will exit, so the new
     * thread can't slip out of an OOM kill (or normal SIGKILL).
      */
     recalc_sigpending();
    if (signal_pending(current)) {
        spin_unlock(&current->sighand->siglock);
        write_unlock_irq(&tasklist_lock);
        retval = -ERESTARTNOINTR;
        goto bad_fork_cleanup_namespaces;
    }

    if (clone_flags & CLONE_THREAD) {
        p->group_leader = current->group_leader;
        list_add_tail_rcu(&p->thread_group, &p->group_leader->thread_group);

        if (!cputime_eq(current->signal->it_virt_expires,
                cputime_zero) ||
            !cputime_eq(current->signal->it_prof_expires,
                cputime_zero) ||
            current->signal->rlim[RLIMIT_CPU].rlim_cur != RLIM_INFINITY ||
            !list_empty(&current->signal->cpu_timers[0]) ||
            !list_empty(&current->signal->cpu_timers[1]) ||
            !list_empty(&current->signal->cpu_timers[2])) {
            /*
             * Have child wake up on its first tick to check
             * for process CPU timers.
             */
            p->it_prof_expires = jiffies_to_cputime(1);
        }
    }

    if (likely(p->pid)) {
        add_parent(p);
        if (unlikely(p->ptrace & PT_PTRACED))
            __ptrace_link(p, current->parent);

        if (thread_group_leader(p)) {
            p->signal->tty = current->signal->tty;
            p->signal->pgrp = process_group(current);
            set_signal_session(p->signal, process_session(current));
            attach_pid(p, PIDTYPE_PGID, task_pgrp(current));
            attach_pid(p, PIDTYPE_SID, task_session(current));

            list_add_tail_rcu(&p->tasks, &init_task.tasks);
            __get_cpu_var(process_counts)++;
        }
        attach_pid(p, PIDTYPE_PID, pid);
        nr_threads++;
    }

    total_forks++;
    spin_unlock(&current->sighand->siglock);
    write_unlock_irq(&tasklist_lock);
    proc_fork_connector(p);
    return p;

bad_fork_cleanup_namespaces:
    exit_task_namespaces(p);
bad_fork_cleanup_keys:
    exit_keys(p);
bad_fork_cleanup_mm:
    if (p->mm)
        mmput(p->mm);
bad_fork_cleanup_signal:
    cleanup_signal(p);
bad_fork_cleanup_sighand:
    __cleanup_sighand(p->sighand);
bad_fork_cleanup_fs:
    exit_fs(p); /* blocking */
bad_fork_cleanup_files:
    exit_files(p); /* blocking */
bad_fork_cleanup_semundo:
    exit_sem(p);
bad_fork_cleanup_audit:
    audit_free(p);
bad_fork_cleanup_security:
    security_task_free(p);
bad_fork_cleanup_policy:
#ifdef CONFIG_NUMA
    mpol_free(p->mempolicy);
bad_fork_cleanup_cpuset:
#endif
    cpuset_exit(p);
bad_fork_cleanup_delays_binfmt:
    delayacct_tsk_free(p);
    if (p->binfmt)
        module_put(p->binfmt->module);
bad_fork_cleanup_put_domain:
    module_put(task_thread_info(p)->exec_domain->module);
bad_fork_cleanup_count:
    put_group_info(p->group_info);
    atomic_dec(&p->user->processes);
    free_uid(p->user);
bad_fork_free:
    free_task(p);
fork_out:
    return ERR_PTR(retval);
}
```

dup_task_struct函数，tsk = alloc_task_struct();dup_task_struct()函数主要是为子进程创建一个内核栈，主要赋值语句setup_thread_stack(tsk, orig);

在函数中调用alloc_task_struct()进行内存分配，alloc_task_struct()函数获取内存的方式内核里面有几种：

1、# define alloc_task_struct() kmem_cache_alloc(task_struct_cachep, GFP_KERNEL)

2、

```text
struct task_struct *alloc_task_struct(void)
{
    struct task_struct *p = kmalloc(THREAD_SIZE, GFP_KERNEL);
    if (p)
        atomic_set((atomic_t *)(p+1), 1);
    return p;
}
```

3、#define alloc_task_struct() ((struct task_struct *)_ _ get_free_pages(GFP_KERNEL | __GFP_COMP, KERNEL_STACK_SIZE_ORDER))

以上3中申请内存的方式最后一种是最底层的，直接分配页，第二种利用了页高速缓存，相当于是对第3中方式进行了封装，第1种在第2中的方式上进行分配，相当于调用了第2种页高速缓存的API进行内存分配的。

```text
static struct task_struct *dup_task_struct(struct task_struct *orig)
{
    struct task_struct *tsk;
    struct thread_info *ti;

    prepare_to_copy(orig);

    tsk = alloc_task_struct();
    if (!tsk)
        return NULL;

    ti = alloc_thread_info(tsk);
    if (!ti) {
        free_task_struct(tsk);
        return NULL;
    }

    *tsk = *orig;
    tsk->stack = ti;
    setup_thread_stack(tsk, orig);               //主要赋值语句将父进程的进程的thread_info赋值给子进程

#ifdef CONFIG_CC_STACKPROTECTOR
    tsk->stack_canary = get_random_int();
#endif

    /* One for us, one for whoever does the "release_task()" (usually parent) */
    atomic_set(&tsk->usage,2);
    atomic_set(&tsk->fs_excl, 0);
#ifdef CONFIG_BLK_DEV_IO_TRACE
    tsk->btrace_seq = 0;
#endif
    tsk->splice_pipe = NULL;
    return tsk;
}
```

现在我们主要分析copy_process函数，此函数中做了非常重要的，体现linux中父子进程创建机制的工作。

1、调用dup_task_struct()为子进程创建一个内核栈、thread_info结构和task_struct，这些值与当前进程的值相同。此时子进程和父进程的描述符是完全相同的。

p = dup_task_struct(current)---->(struct task_struct *tsk---------->tsk = alloc_task_struct()从slab层分配了一个关于进程描述符的slab)

2、检查并确保新创建这个子进程后，当前用户所拥有的进程数目没有超出给它分配的资源的限制。

3、子进程着手使自己与父进程区别开来，为进程的task_struct、tss做个性化设置，进程描述符内的许多成员都要被清0或设置为初始值。那些不是继承而来的进程描述符成员，主要是统计信息。task_struct中的大多数数据都依然未被修改。

4、为子进程创建第一个页表，将进程0的页表项内容赋给这个页表。

copy_process()————>copy_fs(),_copy_fs_struct(current->fs)中current指针表示当前进程也就是父进程的

copy_fs()函数为子进程复制父进程的页目录项

```text
static inline int copy_fs(unsigned long clone_flags, struct task_struct * tsk)
{
    if (clone_flags & CLONE_FS) {
        atomic_inc(&current->fs->count);
        return 0;
    }
    tsk->fs = __copy_fs_struct(current->fs);
    if (!tsk->fs)
        return -ENOMEM;
    return 0;
}
```

_copy_fs_struct()

```text
static inline struct fs_struct *__copy_fs_struct(struct fs_struct *old)
{
    struct fs_struct *fs = kmem_cache_alloc(fs_cachep, GFP_KERNEL);
    /* We don't need to lock fs - think why ;-) */
    if (fs) {
        atomic_set(&fs->count, 1);
        rwlock_init(&fs->lock);
        fs->umask = old->umask;
        read_lock(&old->lock);                         //进行加锁不能被打断
        fs->rootmnt = mntget(old->rootmnt);
        fs->root = dget(old->root);
        fs->pwdmnt = mntget(old->pwdmnt);
        fs->pwd = dget(old->pwd);
        if (old->altroot) {
            fs->altrootmnt = mntget(old->altrootmnt);
            fs->altroot = dget(old->altroot);
        } else {
            fs->altrootmnt = NULL;
            fs->altroot = NULL;
        }
        read_unlock(&old->lock);
    }
    return fs;
}
```

fs_struct数据结构，这个数据结构将VFS层里面的描述页目录对象的结构体进行了实例化，这样就可以为子进程创建一个页目录项，同时这个fs_strcut结构体和为子进程分配内核栈一样都是通过页高速缓存实现的：struct fs_struct *fs = kmem_cache_alloc(fs_cachep, GFP_KERNEL);

```text
struct fs_struct {
    atomic_t count;
    rwlock_t lock;
    int umask;
    struct dentry * root, * pwd, * altroot;                     //struct denty 页目录项结构体
    struct vfsmount * rootmnt, * pwdmnt, * altrootmnt;
};
```

copy_files()函数，为子进程复制父进程的页表，共享父进程的文件

```text
static int copy_files(unsigned long clone_flags, struct task_struct * tsk)
{
    struct files_struct *oldf, *newf;
    int error = 0;

    /*
     * A background process may not have any files ...
     */
    oldf = current->files;                         //将父进程的页表
    if (!oldf)
        goto out;

    if (clone_flags & CLONE_FILES) {
        atomic_inc(&oldf->count);
        goto out;
    }

    /*
     * Note: we may be using current for both targets (See exec.c)
     * This works because we cache current->files (old) as oldf. Don't
     * break this.
     */
    tsk->files = NULL;
    newf = dup_fd(oldf, &error);
    if (!newf)
        goto out;

    tsk->files = newf;
    error = 0;
out:
    return error;
}
```

dup_fd()　

```text
/*
 * Allocate a new files structure and copy contents from the
 * passed in files structure.
 * errorp will be valid only when the returned files_struct is NULL.
 */
 files_struct
static struct files_struct *dup_fd(struct files_struct *oldf, int *errorp)
{
    struct files_struct *newf;
    struct file **old_fds, **new_fds;
    int open_files, size, i;
    struct fdtable *old_fdt, *new_fdt;

    *errorp = -ENOMEM;
    newf = alloc_files();
    if (!newf)
        goto out;

    spin_lock(&oldf->file_lock);
    old_fdt = files_fdtable(oldf);
    new_fdt = files_fdtable(newf);
    open_files = count_open_files(old_fdt);

    /*
     * Check whether we need to allocate a larger fd array and fd set.
     * Note: we're not a clone task, so the open count won't change.
     */
    if (open_files > new_fdt->max_fds) {
        new_fdt->max_fds = 0;
        spin_unlock(&oldf->file_lock);
        spin_lock(&newf->file_lock);
        *errorp = expand_files(newf, open_files-1);
        spin_unlock(&newf->file_lock);
        if (*errorp < 0)
            goto out_release;
        new_fdt = files_fdtable(newf);
        /*
         * Reacquire the oldf lock and a pointer to its fd table
         * who knows it may have a new bigger fd table. We need
         * the latest pointer.
         */
        spin_lock(&oldf->file_lock);
        old_fdt = files_fdtable(oldf);
    }

    old_fds = old_fdt->fd;
    new_fds = new_fdt->fd;

    memcpy(new_fdt->open_fds->fds_bits,
        old_fdt->open_fds->fds_bits, open_files/8);
    memcpy(new_fdt->close_on_exec->fds_bits,
        old_fdt->close_on_exec->fds_bits, open_files/8);

    for (i = open_files; i != 0; i--) {
        struct file *f = *old_fds++;
        if (f) {
            get_file(f);
        } else {
            /*
             * The fd may be claimed in the fd bitmap but not yet
             * instantiated in the files array if a sibling thread
             * is partway through open().  So make sure that this
             * fd is available to the new process.
             */
            FD_CLR(open_files - i, new_fdt->open_fds);
        }
        rcu_assign_pointer(*new_fds++, f);
    }
    spin_unlock(&oldf->file_lock);

    /* compute the remainder to be cleared */
    size = (new_fdt->max_fds - open_files) * sizeof(struct file *);

    /* This is long word aligned thus could use a optimized version */
    memset(new_fds, 0, size);

    if (new_fdt->max_fds > open_files) {
        int left = (new_fdt->max_fds-open_files)/8;
        int start = open_files / (8 * sizeof(unsigned long));

        memset(&new_fdt->open_fds->fds_bits[start], 0, left);
        memset(&new_fdt->close_on_exec->fds_bits[start], 0, left);
    }

    return newf;

out_release:
    kmem_cache_free(files_cachep, newf);
out:
    return NULL;
}
```

files_struct结构体，files_struct结构保存了进程打开的所有文件表数据，描述一个正被打开的文件。

```text
struct files_struct {
    atomic_t        count;              //自动增量
    struct fdtable  *fdt;
    struct fdtable  fdtab;
    fd_set      close_on_exec_init;     //执行exec时
需要关闭的文件描述符初值集合
    fd_set      open_fds_init;          //当前打开文件
的文件描述符屏蔽字
    struct file         * fd_array[NR_OPEN_DEFAULT];
    spinlock_t      file_lock;  /* Protects concurrent
writers.  Nests inside tsk->alloc_lock */
};
```

alloc_files()函数

```text
static struct files_struct *alloc_files(void)
{
    struct files_struct *newf;
    struct fdtable *fdt;

    newf = kmem_cache_alloc(files_cachep, GFP_KERNEL);
    if (!newf)
        goto out;

    atomic_set(&newf->count, 1);

    spin_lock_init(&newf->file_lock);
    newf->next_fd = 0;
    fdt = &newf->fdtab;
    fdt->max_fds = NR_OPEN_DEFAULT;
    fdt->close_on_exec = (fd_set *)&newf->close_on_exec_init;
    fdt->open_fds = (fd_set *)&newf->open_fds_init;
    fdt->fd = &newf->fd_array[0];
    INIT_RCU_HEAD(&fdt->rcu);
    fdt->next = NULL;
    rcu_assign_pointer(newf->fdt, fdt);
out:
    return newf;
}
```

4、子进程的状态被设置为TASK_UNINTERRUPTEIBLE,保证子进程不会投入运行。

前面对于子进程个性化设置没有分析得很清楚，后面自己弄懂了再来补充。

先总结一下fork()的执行流程然后在来解决文章刚开始的问题。

从上面的分析可以看出fork()的流程大概是：

1、p = dup_task_struct(current);　为新进程创建一个内核栈、thread_iofo和task_struct,这里完全copy父进程的内容，所以到目前为止，父进程和子进程是没有任何区别的。

2、为新进程在其内存上建立内核堆栈

3、对子进程task_struct任务结构体中部分变量进行初始化设置，检查所有的进程数目是否已经超出了系统规定的最大进程数，如果没有的话，那么就开始设置进程描诉符中的初始值，从这开始，父进程和子进程就开始区别开了。

4、把父进程的有关信息复制给子进程，建立共享关系

5、设置子进程的状态为不可被TASK_UNINTERRUPTIBLE，从而保证这个进程现在不能被投入运行，因为还有很多的标志位、数据等没有被设置

6、复制标志位（falgs成员）以及权限位(PE_SUPERPRIV)和其他的一些标志

7、调用get_pid()给子进程获取一个有效的并且是唯一的进程标识符PID

8、return ret_from_fork;返回一个指向子进程的指针，开始执行

关于文章开始提出的问题，我们可以从前面的分析知道，子进程的产生是从父进程那儿复制的内核栈、页表项以及与父进程共享文件(对于父进程的文件只能读不能写)，所以子进程如果没有执行exac()函数载入自己的可执行代码，他和父进程将共享数据即代码段数据段，这就是为什么fork()一次感觉执行了两次printf()函数，至于为什么不是6次“+”这个和标准I/O里面的缓冲有关系，所以后面我用了一个不带缓冲的I/O函数进行了测试输出是6次“-”，在子进程复制父进程的内核栈、页表项、页表的时候页把缓存复制到了子进程中，所以多了两次。

可以从下面的图中看明白

![img](https://pic4.zhimg.com/80/v2-8697811d12166c392c829101b9c83ea7_720w.webp)

总结

linux创建一个新的进程是从复制父进程内核栈、页表项开始的，在系统内核里首先是将父进程的进程描述符进行拷贝，然后再根据自己的情况修改相应的参数，获取自己的进程号，再开始执行。

后续关于线程

在前面我们讲的是在linux中创建一个进程，其实在其中创建线程也和上面的流程一样，只是我们需要设置标志位让子进程与父进程共享数据。linux实现线程的机制非常独特，从内核的角度讲，linux没有线程这个说法，linux把所有的线程都当做进程来实现。内核没有准备特别的调度算法或者是定义特别的数据结构来表征线程。相反，线程仅仅被视为一个与其它进程共享某些资源的进程。每个线程都拥有唯一隶属于自己的task_struct，所以在内核中看起来像一个普通的进程只是线程和其它进程共享某些资源，如地址空间。

所以linux里面实现线程的方法和windows或者sun solaris等操作系统实现差异非常大。这些操作系统在内核里面专门提供了支持线程的机制。对于linux来说线程只是一种共享资源的手段。

线程创建时和普通的进程类似，只不过在调用clone()的时候需要传递一些参数标志来指明需要共享的资源。如：

CLONE_FILES:父子进程共享打开的文件

CLONE_FS：父子进程共享打开的文件系统信息

。。。。

后续关于进程终结

一般来说进程的析构是自身引起的。它发生在进程调用exit()系统调用时，既可以显示的调用这个系统调用，也可以隐式的从某个函数返回，C语言编译器会在main函数的返回点后面放置调用exit()的代码。当进程接收到它既不能处理也不能忽略的信号或者异常时，它还能被动的终结。调用do_exit()函数完成进程的终结。进程的终结就是一个释放进程占有的资源的过程。

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/547943571