## 一，进程

程序是指储存在外部存储(如硬盘)的一个可执行文件, 而进程是指处于执行期间的程序, 进程包括 代码段(textsection) 和 数据段(data section) , 除了代码段和数据段外, 进程一般还包含打开的文件, 要处理的信号和CPU上下文等等。

## 二，进程描述符

Linux进程使用 struct task_struct根据描述(include/linux/sched.h), 如下:

```text
struct task_struct {/** offsets of these are hardcoded elsewhere - touch with care */ 
  volatile long state; /* -1 unrunnable, 0 runnable, >0 stopped */ 
  unsigned long flags; /* per process flags, defined below */ 
  int sigpending; mm_segment_t addr_limit; /* thread address space: 0-0xBFFFFFFF for user-thead 0-0xFFFFFFFF for kernel-thread */ 
  struct exec_domain *exec_domain; 
  volatile long need_resched;
  unsigned long ptrace;
  int lock_depth; /* Lock depth */ /** offset 32 begins here on 32-bit platforms. We keep * all fields in a single cacheline that are needed for * the goodness() loop in schedule(). */
  long counter; 
  long nice; 
  unsigned long policy; 
  struct mm_struct *mm;
  int processor;
  ... }
```

**Linux把所有的进程使用双向链表连接起来, 如下图(来源<Linux设计与实现>):**



![img](https://pic3.zhimg.com/80/v2-43d9ec266d5c37594e4de7d5758ba50e_720w.webp)



Linux内核为了加快获取当前进程的的task_struct结构, 使用了一个技巧, 就是把task_struct放置在内核栈的栈底, 这样就可以通过esp寄存器快速获取到当前运行进程的task_struct结构. 如下图:



![img](https://pic4.zhimg.com/80/v2-eedd18a394eedd6829849cc25e89dc17_720w.webp)



**获取当前运行进程的task_struct代码如下:**

```text
static inline struct task_struct * get_current(void) { 
          struct task_struct *current; __asm__(
        "andl %%esp,%0; ":"=r" (current) : "0" (~8191UL)
); 
return current;
}
```

## 三，进程状态

**进程描述符的state字段用于保存进程的当前状态, 进程的状态有以下几种:**

1. TASK_RUNNING (运行) -- 进程处于可执行状态, 在这个状态下的进程要么正在被CPU执行, 要么在等待执行(CPU被其他进程占用的情况下)
2. TASK_INTERRUPTIBLE (可中断等待) -- 进程处于等待状态, 其在等待某些条件成立或者接收到某些信号, 进程会被唤醒变为运行状态
3. TASK_UNINTERRUPTIBLE (不可中断等待) -- 进程处于等待状态, 其在等待某些条件成立, 进程会被唤醒变为运行状态, 但不能被信号唤醒.
4. TASK_TRACED (被追踪) -- 进程处于被追踪状态, 例如通过ptrace命令对进程进行调试.
5. TASK_STOPPED (停止) -- 进程处于停止状态, 进程不能被执行. 一般接收到SIGSTOP, SIGTSTP,SIGTTIN, SIGTTOU信号进程会变成TASK_STOPPED状态.

**各种状态间的转换如下图:**





![img](https://pic3.zhimg.com/80/v2-c0b28d06c4e8f8436d3e94efd820af9e_720w.webp)



## 四，进程的创建

在Linux系统中，进程的创建使用fork()系统调用，fork()调用会创建一个与父进程一样的子进程，唯一不同就是fork()的返回值，父进程返回的是子进程的进程ID，而子进程返回的是0。Linux创建子进程时使用了 写时复制（Copy On Write） ，也就是创建子进程时使用的是父进程的内存空间，当子进程或者父进程修改数据时才会复制相应的内存页。当调用fork()系统调用时会陷入内核空间并且调用sys_fork()函数，sys_fork()函数会调用do_fork()函数，代码如下(arch/i386/kernel/process.c)：

```text
asmlinkage int sys_fork(struct pt_regs regs) { 
  return do_fork(SIGCHLD, regs.esp, &regs, 0); 
}
```

do_fork()主要的工作是申请一个进程描述符, 然后初始化进程描述符的各个字段, 包括调用 copy_files() 函数复制打开的文件, 调用 copy_sighand() 函数复制信号处理函数, 调用 copy_mm() 函数复制进程虚拟内存空间, 调用 copy_namespace() 函数复制命名空间. **代码如下:**

```text
int do_fork( 
  unsigned long clone_flags, unsigned long stack_start, struct pt_regs *regs, 
  unsigned long stack_size) { 
  ... p = alloc_task_struct(); // 申请进程描述符 ... 
  if (copy_files(clone_flags, p)) goto bad_fork_cleanup;
  if (copy_fs(clone_flags, p)) goto bad_fork_cleanup_files; 
  if (copy_sighand(clone_flags, p)) goto bad_fork_cleanup_fs;
  if (copy_mm(clone_flags, p)) goto bad_fork_cleanup_sighand; 
  if (copy_namespace(clone_flags, p)) goto bad_fork_cleanup_mm; 
  retval = copy_thread(0, clone_flags, stack_start, stack_size, p, regs); 
  ... wake_up_process(p); ... 
}
```

值得注意的是do_fork() 还调用了 copy_thread() 这个函数, copy_thread()这个函数主要用于设置进程的CPU执行上下文 struct thread_struct 结构代码如下:

```text
int copy_thread(
  int nr, unsigned long clone_flags, unsigned long esp, 
  unsigned long unused, 
  struct task_struct * p, struct pt_regs * regs) { 
  struct pt_regs * childregs; // 指向栈顶(见图2) 
  childregs = ((struct pt_regs *) (THREAD_SIZE + (unsigned long) p)) - 1;
  struct_cpy(childregs, regs); // 复制父进程的栈信息 
  childregs->eax = 0; // 这个是子进程调用fork()之后的返回值, 也就是0 
  childregs->esp = esp;
  p->thread.esp = (unsigned long) childregs; // 子进程当前的栈地址, 调用 switch_to()的时候esp设置为这个地址 
  p->thread.esp0 = (unsigned long) (childregs+1); // 子进程内核空间栈地址 
  p->thread.eip = (unsigned long) ret_from_fork; // 子进程将要执行的代码地址 
  savesegment(fs,p->thread.fs
             ); savesegment(gs,p->thread.gs); 
  unlazy_fpu(current); 
  struct_cpy(&p->thread.i387,
             
  &current->thread.i387); 
  return 0;
}
```

do_fork() 函数最后调用 wake_up_process() 函数唤醒子进程, 让子进程进入运行状态

## 五，内核线程

Linux内核有很多任务需要去做, 例如定时把缓冲器中的数据刷到硬盘上, 当内存不足的时候进行内存的回收等, 这些所有工作都需要通过内核线程来完成. 内核线程与普通进程的主要区别就是: 内核线程没有自己的 虚拟空间结构(struct mm) , 每次内核线程执行的时候都是借助当前运行进程的虚拟内存空间结构来运行, 因为内核线程只会运行在内核状态, 而每个进程的内核态空间都是一样的, 所以借助其他进程的虚拟内存空间结构来运行是完成可行的内核线程使用 kernel_thread() 函数来创建, 代码如下:

```text
int kernel_thread(int (*fn)(void *), void * arg, unsigned long flags)
      {
                  long retval, d0;
                  __asm__ __volatile__(
                  "movl %%esp,%%esi\n\t"
                  "int $0x80\n\t" /* Linux/i386 system call */ 
                  "cmpl %%esp,%%esi\n\t" /* child or parent? */ 
                  "je 1f\n\t" /* parent - jump */
                  /* Load the argument into eax, and push it. That way, it does * not matter  
                whether the called function is compiled with * -mregparm or not. */
               "movl %4,%%eax\n\t"
                  "pushl %%eax\n\t" 
                  "call *%5\n\t" /* call fn */ 
                  "movl %3,%0\n\t" /* exit */
                  "int $0x80\n" "1:\t"
                  :"=&a" (retval), "=&S" (d0) :"0" (__NR_clone), "i" (__NR_exit), "r" (arg), "r" (fn), "b" (flags | CLONE_VM) : "memory");
return retval; 
}
```

因为这个函数是使用嵌入汇编来实现的, 所以有点难懂, 不过主要过程就是通过调用 _clone() 函数来创建一个新的进程, 而创建进程是通过传入CLONE_VM 标志来指定进程借用其他进程的虚拟内存空间结构。

**特别说明一下：**d0局部变量的作用是为了在创建内核线程时保证 struct pt_regs 结构的完整，这是因为创建内核线程是在内核态进行的，所以在内核态调用系统调用是不会压入 ss 和esp寄存器的，这样就会导致系统调用的 struct pt_regs参数信息不完整，所以 kernel_thread() 函数定义了一个 d0 局部变量是为了补充没压栈的ss和esp的。

-----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/441346358