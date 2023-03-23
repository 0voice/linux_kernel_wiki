## 1、前言

在前文中，我们分析了内核中进程和线程的统一结构体task_struct，本文将继续分析进程、线程的创建和派生的过程。首先介绍如何将一个程序编辑为执行文件最后成为进程执行，然后会介绍线程的执行，最后会分析如何通过已有的进程、线程实现多进程、多线程。因为进程和线程有诸多相似之处，也有一些不同之处，因此本文会对比进程和线程来加深理解和记忆。

## 2、 进程的创建

以C语言为例，我们在Linux下编写C语言代码，然后通过gcc编译和链接生成可执行文件后直接执行即可完成一个进程的创建和工作。下面将详细介绍这个创建进程的过程。在 Linux 下面，二进制的程序也要有严格的格式，这个格式我们称为 ELF（Executable and Linkable Format，可执行与可链接格式）。这个格式可以根据编译的结果不同，分为不同的格式。主要包括

1、可重定位的对象文件(Relocatable file)

由汇编器汇编生成的 .o 文件

2、可执行的对象文件(Executable file)

可执行应用程序

3、可被共享的对象文件(Shared object file)

动态库文件，也就是 .so 文件

下面在进程创建过程中会详细说明三种文件。

### 2. 1 编译

写完C程序后第一步就是程序编译（其实还有IDE的预编译，那些属于编辑器操作这里不表）。编译指令如下所示

```text
gcc -c -fPIC xxxx.c
```

-c表示编译、汇编指定的源文件，不进行链接。-fPIC表示生成与位置无关（Position-Independent Code）代码，即采用相对地址而非绝对地址，从而满足共享库加载需求。在编译的时候，先做预处理工作，例如将头文件嵌入到正文中，将定义的宏展开，然后就是真正的编译过程，最终编译成为.o 文件，这就是 ELF 的第一种类型，可重定位文件（Relocatable File）。之所以叫做可重定位文件，是因为对于编译好的代码和变量，将来加载到内存里面的时候，都是要加载到一定位置的。比如说，调用一个函数，其实就是跳到这个函数所在的代码位置执行；再比如修改一个全局变量，也是要到变量的位置那里去修改。但是现在这个时候，还是.o 文件，不是一个可以直接运行的程序，这里面只是部分代码片段。因此.o 里面的位置是不确定的，但是必须要重新定位以适应需求。

![img](https://pic1.zhimg.com/80/v2-b1b1c6683f929bec6653c449a1718864_720w.webp)

ELF文件的开头是用于描述整个文件的。这个文件格式在内核中有定义，分别为 struct elf32_hdr 和struct elf64_hdr。

**其他各个section作用如下所示：**

- .text：放编译好的二进制可执行代码
- .rodata：只读数据，例如字符串常量、const 的变量
- .data：已经初始化好的全局变量
- .bss：未初始化全局变量，运行时会置 0
- .symtab：符号表，记录的则是函数和变量
- .rel.text： .text部分的重定位表
- .rel.data：.data部分的重定位表
- .strtab：字符串表、字符串常量和变量名

这些节的元数据信息也需要有一个地方保存，就是最后的节头部表（Section Header Table）。在这个表里面，每一个 section 都有一项，在代码里面也有定义 struct elf32_shdr和struct elf64_shdr。在 ELF 的头里面，有描述这个文件的接头部表的位置，有多少个表项等等信息。

### 2.2 链接

链接分为静态链接和动态链接。静态链接库会和目标文件通过链接生成一个可执行文件，而动态链接则会通过链接形成动态连接器，在可执行文件执行的时候动态的选择并加载其中的部分或全部函数。

**二者的各自优缺点如下所示：**

- **静态链接库的优点**

(1) 代码装载速度快，执行速度略比动态链接库快；

(2) 只需保证在开发者的计算机中有正确的.LIB文件，在以二进制形式发布程序时不需考虑在用户的计算机上.LIB文件是否存在及版本问题，可避免DLL地狱等问题。

- **静态链接库的缺点**

使用静态链接生成的可执行文件体积较大，包含相同的公共代码，造成浪费

- **动态链接库的优点**

(1) 更加节省内存并减少页面交换；

(2) DLL文件与EXE文件独立，只要输出接口不变（即名称、参数、返回值类型和调用约定不变），更换DLL文件不会对EXE文件造成任何影响，因而极大地提高了可维护性和可扩展性；

(3) 不同编程语言编写的程序只要按照函数调用约定就可以调用同一个DLL函数；

(4)适用于大规模的软件开发，使开发过程独立、耦合度小，便于不同开发者和开发组织之间进行开发和测试。

- **动态链接库的缺点**

使用动态链接库的应用程序不是自完备的，它依赖的DLL模块也要存在，如果使用载入时动态链接，程序启动时发现DLL不存在，系统将终止程序并给出错误信息。而使用运行时动态链接，系统不会终止，但由于DLL中的导出函数不可用，程序会加载失败；速度比静态连接慢。当某个模块更新后，如果新的模块与旧的模块不兼容，那么那些需要该模块才能运行的软件均无法执行。这在早期Windows中很常见。

**下面分别介绍静态链接和动态链接：**

#### 2.2.1 静态链接

静态链接库.a文件（Archives）的执行指令如下

```text
ar cr libXXX.a XXX.o XXXX.o
```

当需要使用该静态库的时候，会将.o文件从.a文件中依次抽取并链接到程序中，指令如下

```text
gcc -o XXXX XXX.O -L. -lsXXX
```

-L表示在当前目录下找.a 文件，-lsXXXX会自动补全文件名，比如加前缀 lib，后缀.a，变成libXXX.a，找到这个.a文件后，将里面的 XXXX.o 取出来，和 XXX.o 做一个链接，形成二进制执行文件XXXX。在这里，重定位会从.o中抽取函数并和.a中的文件抽取的函数进行合并，找到实际的调用位置，形成最终的可执行文件(Executable file)，即ELF的第二种格式文件。

![img](https://pic1.zhimg.com/80/v2-78827771a0af63369ebee996d6d094bc_720w.webp)



对比ELF第一种格式可重定位文件，这里可执行文件略去了重定位表相关段落。此处将ELF文件分为了代码段、数据段和不加载到内存中的部分，并加上了段头表（Segment Header Table）用以记录管理，在代码中定义为struct elf32_phdr和 struct elf64_phdr，这里面除了有对于段的描述之外，最重要的是 p_vaddr，这个是这个段加载到内存的虚拟地址。这部分会在内存篇章详细介绍。

#### 2.2.2 动态链接

动态链接库（Shared Libraries)的作用主要是为了解决静态链接大量使用会造成空间浪费的问题，因此这里设计成了可以被多个程序共享的形式，其执行命令如下

```text
gcc -shared -fPIC -o libXXX.so XXX.o
```

当一个动态链接库被链接到一个程序文件中的时候，最后的程序文件并不包括动态链接库中的代码，而仅仅包括对动态链接库的引用，并且不保存动态链接库的全路径，仅仅保存动态链接库的名称。

```text
gcc -o XXX XXX.O -L. -lXXX
```

当运行这个程序的时候，首先寻找动态链接库，然后加载它。默认情况下，系统在 /lib 和/usr/lib 文件夹下寻找动态链接库。如果找不到就会报错，我们可以设定 LD_LIBRARY_PATH环境变量，程序运行时会在此环境变量指定的文件夹下寻找动态链接库。动态链接库，就是 ELF 的第三种类型，共享对象文件（Shared Object）。

**动态链接的ELF相对于静态链接主要多了以下部分：**

- .interp段，里面是ld-linux.so，负责运行时的链接动作
- .plt（Procedure Linkage Table），过程链接表
- .got.plt（Global Offset Table），全局偏移量表

当程序编译时，会对每个函数在PLT中建立新的项，如PLT[n]，而动态库中则存有该函数的实际地址，记为GOT[m]。

**整体寻址过程如下所示：**

- PLT[n]向GOT[m]寻求地址

- GOT[m]初始并无地址，需要采取以下方式获取地址

- - 回调PLT[0]
  - PLT[0]调用GOT[2]，即ld-linux.so
  - ld-linux.so查找所需函数实际地址并存放在GOT[m]中

由此，我们建立了PLT[n]到GOT[m]的对应关系，从而实现了动态链接。

### 2.3 加载运行

完成了上述的编译、汇编、链接，我们最终形成了可执行文件，并加载运行。在内核中，有这样一个数据结构，用来定义加载二进制文件的方法。

```cpp
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *cprm);
    unsigned long min_coredump;     /* minimal dump size */
} __randomize_layout;
```

**对于ELF文件格式，其对应实现为：**

```cpp
static struct linux_binfmt elf_format = {
    .module         = THIS_MODULE,
    .load_binary    = load_elf_binary,
    .load_shlib     = load_elf_library,
    .core_dump      = elf_core_dump,
    .min_coredump   = ELF_EXEC_PAGESIZE,
};
```

其中加载的函数指针指向的函数和内核镜像加载是同一份函数，实际上通过exec函数完成调用。exec 比较特殊，它是一组函数：

- 包含 p 的函数（execvp, execlp）会在 PATH 路径下面寻找程序；不包含 p 的函数需要输入程序的全路径；
- 包含 v 的函数（execv, execvp, execve）以数组的形式接收参数；
- 包含 l 的函数（execl, execlp, execle）以列表的形式接收参数；
- 包含 e 的函数（execve, execle）以数组的形式接收环境变量。

当我们通过shell运行可执行文件或者通过fork派生子类，均是通过该类函数实现加载。

## 3、线程的创建之用户态

线程的创建对应的函数是pthread_create()，线程不是一个完全由内核实现的机制，它是由内核态和用户态合作完成的。pthread_create()不是一个系统调用，是 Glibc 库的一个函数，所以我们还要从 Glibc 说起。但是在开始之前，我们先要提一下，线程的创建到了内核态和进程的派生会使用同一个函数：__do_fork()，这也很容易理解，因为对内核态来说，线程和进程是同样的task_struct结构体。本节介绍线程在用户态的创建，而内核态的创建则会和进程的派生放在一起说明。

在Glibc的ntpl/pthread_create.c中定义了__pthread_create_2_1()函数，该函数主要进行了以下操作

处理线程的属性参数。例如前面写程序的时候，我们设置的线程栈大小。如果没有传入线程属性，就取默认值。

```cpp
const struct pthread_attr *iattr = (struct pthread_attr *) attr;
struct pthread_attr default_attr;
//c11 thrd_create
bool c11 = (attr == ATTR_C11_THREAD);
if (iattr == NULL || c11)
{
  ......
    iattr = &default_attr;
}
```

就像在内核里每一个进程或者线程都有一个 task_struct 结构，在用户态也有一个用于维护线程的结构，就是这个 pthread 结构。

```cpp
struct pthread *pd = NULL;
```

凡是涉及函数的调用，都要使用到栈。每个线程也有自己的栈，接下来就是创建线程栈了。

```cpp
int err = ALLOCATE_STACK (iattr, &pd);
```

**ALLOCATE_STACK 是一个宏，对应的函数allocate_stack()主要做了以下这些事情：**

- 如果在线程属性里面设置过栈的大小，则取出属性值；
- 为了防止栈的访问越界在栈的末尾添加一块空间 guardsize，一旦访问到这里就会报错；
- 线程栈是在进程的堆里面创建的。如果一个进程不断地创建和删除线程，我们不可能不断地去申请和清除线程栈使用的内存块，这样就需要有一个缓存。get_cached_stack 就是根据计算出来的 size 大小，看一看已经有的缓存中，有没有已经能够满足条件的。如果缓存里面没有，就需要调用__mmap创建一块新的缓存，系统调用那一节我们讲过，如果要在堆里面 malloc 一块内存，比较大的话，用__mmap；
- 线程栈也是自顶向下生长的，每个线程要有一个pthread 结构，这个结构也是放在栈的空间里面的。在栈底的位置，其实是地址最高位；
- 计算出guard内存的位置，调用 setup_stack_prot 设置这块内存的是受保护的；
- 填充pthread 这个结构里面的成员变量 stackblock、stackblock_size、guardsize、specific。这里的 specific 是用于存放Thread Specific Data 的，也即属于线程的全局变量；
- 将这个线程栈放到 stack_used 链表中，其实管理线程栈总共有两个链表，一个是 stack_used，也就是这个栈正被使用；另一个是stack_cache，就是上面说的，一旦线程结束，先缓存起来，不释放，等有其他的线程创建的时候，给其他的线程用。

```cpp
# define ALLOCATE_STACK(attr, pd) allocate_stack (attr, pd, &stackaddr)
static int
allocate_stack (const struct pthread_attr *attr, struct pthread **pdp,
                ALLOCATE_STACK_PARMS)
{
    struct pthread *pd;
    size_t size;
    size_t pagesize_m1 = __getpagesize () - 1;
......
    /* Get the stack size from the attribute if it is set.  Otherwise we
       use the default we determined at start time.  */
    if (attr->stacksize != 0)
        size = attr->stacksize;
    else
    {
        lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
        size = __default_pthread_attr.stacksize;
        lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
    }
......
    /* Allocate some anonymous memory.  If possible use the cache.  */
    size_t guardsize;
    void *mem;
    const int prot = (PROT_READ | PROT_WRITE
                   | ((GL(dl_stack_flags) & PF_X) ? PROT_EXEC : 0));
    /* Adjust the stack size for alignment.  */
    size &= ~__static_tls_align_m1;
    /* Make sure the size of the stack is enough for the guard and
       eventually the thread descriptor.  */
    guardsize = (attr->guardsize + pagesize_m1) & ~pagesize_m1;
    size += guardsize;
......    
    /* Try to get a stack from the cache.  */  
    pd = get_cached_stack (&size, &mem);
    if (pd == NULL)
    {
        /* If a guard page is required, avoid committing memory by first
           allocate with PROT_NONE and then reserve with required permission
           excluding the guard page.  */
        mem = __mmap (NULL, size, (guardsize == 0) ? prot : PROT_NONE,
        MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
        /* Place the thread descriptor at the end of the stack.  */
#if TLS_TCB_AT_TP
        pd = (struct pthread *) ((char *) mem + size) - 1;
#elif TLS_DTV_AT_TP
        pd = (struct pthread *) ((((uintptr_t) mem + size - __static_tls_size) & ~__static_tls_align_m1) - TLS_PRE_TCB_SIZE);
#endif
        /* Now mprotect the required region excluding the guard area. */
        char *guard = guard_position (mem, size, guardsize, pd, pagesize_m1);
        setup_stack_prot (mem, size, guard, guardsize, prot);
        pd->stackblock = mem;
        pd->stackblock_size = size;
        pd->guardsize = guardsize;
        pd->specific[0] = pd->specific_1stblock;
        /* And add to the list of stacks in use.  */
        stack_list_add (&pd->list, &stack_used);
    }
  
    *pdp = pd;
    void *stacktop;
# if TLS_TCB_AT_TP
    /* The stack begins before the TCB and the static TLS block.  */
    stacktop = ((char *) (pd + 1) - __static_tls_size);
# elif TLS_DTV_AT_TP
    stacktop = (char *) (pd - 1);
# endif
    *stack = stacktop;
...... 
}
```

## 4、线程的内核态创建及进程的派生

多进程是一种常见的程序实现方式，采用的系统调用为fork()函数。前文中已经详细叙述了系统调用的整个过程，对于fork()来说，最终会在系统调用表中查找到对应的系统调用sys_fork完成子进程的生成，而sys_fork 会调用 _do_fork()。

```cpp
SYSCALL_DEFINE0(fork)
{
......
    return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
```

关于__do_fork()先按下不表，再接着看看线程。我们接着pthread_create ()看。其实有了用户态的栈，接着需要解决的就是用户态的程序从哪里开始运行的问题。start_routine() 就是给线程的函数，start_routine()， 参数 arg，以及调度策略都要赋值给 pthread。接下来 __nptl_nthreads 加一，说明又多了一个线程。

```cpp
pd->start_routine = start_routine;
pd->arg = arg;
pd->schedpolicy = self->schedpolicy;
pd->schedparam = self->schedparam;
/* Pass the descriptor to the caller.  */
*newthread = (pthread_t) pd;
atomic_increment (&__nptl_nthreads);
retval = create_thread (pd, iattr, &stopped_start, STACK_VARIABLES_ARGS, &thread_ran);
```

真正创建线程的是调用 create_thread() 函数，这个函数定义如下。同时，这里还规定了当完成了内核态线程创建后回调的位置：start_thread()。

```cpp
static int
create_thread (struct pthread *pd, const struct pthread_attr *attr,
bool *stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
{
    const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM | CLONE_SIGHAND | CLONE_THREAD | CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID | 0);
    ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS, clone_flags, pd, &pd->tid, tp, &pd->tid)；
    /* It's started now, so if we fail below, we'll have to cancel it
       and let it clean itself up.  */
    *thread_ran = true;
}
```

在 start_thread() 入口函数中，才真正的调用用户提供的函数，在用户的函数执行完毕之后，会释放这个线程相关的数据。例如，线程本地数据 thread_local variables，线程数目也减一。如果这是最后一个线程了，就直接退出进程，另外 __free_tcb() 用于释放 pthread。

```cpp
#define START_THREAD_DEFN \
  static int __attribute__ ((noreturn)) start_thread (void *arg)

START_THREAD_DEFN
{
    struct pthread *pd = START_THREAD_SELF;
    /* Run the code the user provided.  */
    THREAD_SETMEM (pd, result, pd->start_routine (pd->arg));
    /* Call destructors for the thread_local TLS variables.  */
    /* Run the destructor for the thread-local data.  */
    __nptl_deallocate_tsd ();
    if (__glibc_unlikely (atomic_decrement_and_test (&__nptl_nthreads)))
        /* This was the last thread.  */
        exit (0);
    __free_tcb (pd);
    __exit_thread ();
}
```

__free_tcb ()会调用 __deallocate_stack()来释放整个线程栈，这个线程栈要从当前使用线程栈的列表 stack_used 中拿下来，放到缓存的线程栈列表 stack_cache中，从而结束了线程的生命周期。

```cpp
void
internal_function
__free_tcb (struct pthread *pd)
{
    ......
    __deallocate_stack (pd);
}

void
internal_function
__deallocate_stack (struct pthread *pd)
{
    /* Remove the thread from the list of threads with user defined
       stacks.  */
    stack_list_del (&pd->list);
    /* Not much to do.  Just free the mmap()ed memory.  Note that we do
       not reset the 'used' flag in the 'tid' field.  This is done by
       the kernel.  If no thread has been created yet this field is
       still zero.  */
    if (__glibc_likely (! pd->user_stack))
        (void) queue_stack (pd);
}
  ARCH_CLONE其实调用的是 __clone()。

# define ARCH_CLONE __clone

/* The userland implementation is:
   int clone (int (*fn)(void *arg), void *child_stack, int flags, void *arg),
   the kernel entry is:
   int clone (long flags, void *child_stack).

   The parameters are passed in register and on the stack from userland:
   rdi: fn
   rsi: child_stack
   rdx: flags
   rcx: arg
   r8d: TID field in parent
   r9d: thread pointer
%esp+8: TID field in child

   The kernel expects:
   rax: system call number
   rdi: flags
   rsi: child_stack
   rdx: TID field in parent
   r10: TID field in child
   r8:  thread pointer  */
    .text
ENTRY (__clone)
    movq    $-EINVAL,%rax
......
    /* Insert the argument onto the new stack.  */
    subq    $16,%rsi
    movq    %rcx,8(%rsi)

    /* Save the function pointer.  It will be popped off in the
       child in the ebx frobbing below.  */
    movq    %rdi,0(%rsi)

    /* Do the system call.  */
    movq    %rdx, %rdi
    movq    %r8, %rdx
    movq    %r9, %r8
    mov     8(%rsp), %R10_LP
    movl    $SYS_ify(clone),%eax
......
    syscall
......
PSEUDO_END (__clone)
```

内核中的clone()定义如下。如果在进程的主线程里面调用其他系统调用，当前用户态的栈是指向整个进程的栈，栈顶指针也是指向进程的栈，指令指针也是指向进程的主线程的代码。此时此刻执行到这里，调用 clone的时候，用户态的栈、栈顶指针、指令指针和其他系统调用一样，都是指向主线程的。但是对于线程来说，这些都要变。因为我们希望当 clone 这个系统调用成功的时候，除了内核里面有这个线程对应的 task_struct，当系统调用返回到用户态的时候，用户态的栈应该是线程的栈，栈顶指针应该指向线程的栈，指令指针应该指向线程将要执行的那个函数。所以这些都需要我们自己做，将线程要执行的函数的参数和指令的位置都压到栈里面，当从内核返回，从栈里弹出来的时候，就从这个函数开始，带着这些参数执行下去。

```cpp
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
     int __user *, parent_tidptr,
     int __user *, child_tidptr,
     unsigned long, tls)
{
    return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
```

线程和进程到了这里殊途同归，进入了同一个函数__do_fork()工作。其源码如下所示，主要工作包括复制结构copy_process()和唤醒新进程wak_up_new()两部分。其中线程会根据create_thread()函数中的clone_flags完成上文所述的栈顶指针和指令指针的切换，以及一些线程和进程的微妙区别。

```cpp
long _do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr,
        unsigned long tls)
{
    struct task_struct *p;
    int trace = 0;
    long nr;

......
    p = copy_process(clone_flags, stack_start, stack_size,
       child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
......
    if (IS_ERR(p))
        return PTR_ERR(p);
    struct pid *pid;
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);

    if (clone_flags & CLONE_PARENT_SETTID)
        put_user(nr, parent_tidptr);
    if (clone_flags & CLONE_VFORK) {
        p->vfork_done = &vfork;
        init_completion(&vfork);
        get_task_struct(p);
    }
    wake_up_new_task(p);
......
    put_pid(pid);
    return nr;
};
```

### 4.1 任务结构体复制

如下所示为copy_process()函数源码精简版，task_struct结构复杂也注定了复制过程的复杂性，因此此处省略了很多，仅保留了各个部分的主要调用函数

```cpp
static __latent_entropy struct task_struct *copy_process(
          unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *child_tidptr,
          struct pid *pid,
          int trace,
          unsigned long tls,
          int node)
{
    int retval;
    struct task_struct *p;
......
    //分配task_struct结构
    p = dup_task_struct(current, node);  
......
    //权限处理
    retval = copy_creds(p, clone_flags);
......
    //设置调度相关变量
    retval = sched_fork(clone_flags, p);    
......
    //初始化文件和文件系统相关变量
    retval = copy_files(clone_flags, p);
    retval = copy_fs(clone_flags, p);  
......
    //初始化信号相关变量
    init_sigpending(&p->pending);
    retval = copy_sighand(clone_flags, p);
    retval = copy_signal(clone_flags, p);  
......
    //拷贝进程内存空间
    retval = copy_mm(clone_flags, p);
...... 
    //初始化亲缘关系变量
    INIT_LIST_HEAD(&p->children);
    INIT_LIST_HEAD(&p->sibling);
......
    //建立亲缘关系
    //源码放在后面说明  
};
```

1、copy_process()首先调用了dup_task_struct()分配task_struct结构，dup_task_struct() 主要做了下面几件事情：

- 调用 alloc_task_struct_node 分配一个 task_struct结构；
- 调用 alloc_thread_stack_node 来创建内核栈，这里面调用 __vmalloc_node_range 分配一个连续的 THREAD_SIZE 的内存空间，赋值给 task_struct 的 void *stack成员变量；
- 调用 arch_dup_task_struct(struct task_struct *dst, struct task_struct *src)，将 task_struct 进行复制，其实就是调用 memcpy；
- 调用setup_thread_stack设置 thread_info。

```cpp
static struct task_struct *dup_task_struct(struct task_struct *orig, int node)
{
    struct task_struct *tsk;
    unsigned long *stack;
......
    tsk = alloc_task_struct_node(node);
    if (!tsk)
        return NULL;

    stack = alloc_thread_stack_node(tsk, node);
    if (!stack)
        goto free_tsk; 
    if (memcg_charge_kernel_stack(tsk))
        goto free_stack;

    stack_vm_area = task_stack_vm_area(tsk);

    err = arch_dup_task_struct(tsk, orig);
......    
    setup_thread_stack(tsk, orig);
......    
};
```

2、接着，调用copy_creds处理权限相关内容

- 调用prepare_creds，准备一个新的 struct cred *new。如何准备呢？其实还是从内存中分配一个新的 struct cred结构，然后调用 memcpy 复制一份父进程的 cred；
- 接着 p->cred = p->real_cred = get_cred(new)，将新进程的“我能操作谁”和“谁能操作我”两个权限都指向新的 cred。

```cpp
/*
 * Copy credentials for the new process created by fork()
 *
 * We share if we can, but under some circumstances we have to generate a new
 * set.
 *
 * The new process gets the current process's subjective credentials as its
 * objective and subjective credentials
 */
int copy_creds(struct task_struct *p, unsigned long clone_flags)
{
    struct cred *new;
    int ret;
......
    new = prepare_creds();
    if (!new)
        return -ENOMEM;
......
    atomic_inc(&new->user->processes);
    p->cred = p->real_cred = get_cred(new);
    alter_cred_subscribers(new, 2);
    validate_creds(new);
    return 0;
}
```

3、设置调度相关的变量。该部分源码先不展示，会在进程调度中详细介绍。sched_fork主要做了下面几件事情：

- 调用__sched_fork，在这里面将on_rq设为 0，初始化sched_entity，将里面的 exec_start、sum_exec_runtime、prev_sum_exec_runtime、vruntime 都设为 0。这几个变量涉及进程的实际运行时间和虚拟运行时间。是否到时间应该被调度了，就靠它们几个；
- 设置进程的状态 p->state = TASK_NEW；
- 初始化优先级 prio、normal_prio、static_prio；
- 设置调度类，如果是普通进程，就设置为 p->sched_class = &fair_sched_class；
- 调用调度类的 task_fork 函数，对于 CFS 来讲，就是调用 task_fork_fair。在这个函数里，先调用 update_curr，对于当前的进程进行统计量更新，然后把子进程和父进程的 vruntime 设成一样，最后调用 place_entity，初始化 sched_entity。这里有一个变量 sysctl_sched_child_runs_first，可以设置父进程和子进程谁先运行。如果设置了子进程先运行，即便两个进程的 vruntime 一样，也要把子进程的 sched_entity 放在前面，然后调用 resched_curr，标记当前运行的进程 TIF_NEED_RESCHED，也就是说，把父进程设置为应该被调度，这样下次调度的时候，父进程会被子进程抢占。

4、初始化文件和文件系统相关变量

- copy_files 主要用于复制一个任务打开的文件信息。

- - 对于进程来说，这些信息用一个结构 files_struct 来维护，每个打开的文件都有一个文件描述符。在 copy_files 函数里面调用 dup_fd，在这里面会创建一个新的 files_struct，然后将所有的文件描述符数组 fdtable 拷贝一份。
  - 对于线程来说，由于设置了CLONE_FILES 标识位变成将原来的files_struct 引用计数加一，并不会拷贝文件。

```cpp
static int copy_files(unsigned long clone_flags, struct task_struct *tsk)
{
    struct files_struct *oldf, *newf;
    int error = 0;

    /*
    * A background process may not have any files ...
    */
    oldf = current->files;
    if (!oldf)
        goto out;
    if (clone_flags & CLONE_FILES) {
        atomic_inc(&oldf->count);
        goto out;
    }
    newf = dup_fd(oldf, &error);
    if (!newf)
        goto out;

    tsk->files = newf;
    error = 0;
out:
    return error;
}
```

- copy_fs 主要用于复制一个任务的目录信息。

- - 对于进程来说，这些信息用一个结构 fs_struct 来维护。一个进程有自己的根目录和根文件系统 root，也有当前目录 pwd 和当前目录的文件系统，都在 fs_struct 里面维护。copy_fs 函数里面调用 copy_fs_struct，创建一个新的 fs_struct，并复制原来进程的 fs_struct。
  - 对于线程来说，由于设置了CLONE_FS 标识位变成将原来的fs_struct 的用户数加一，并不会拷贝文件系统结构。

```cpp
static int copy_fs(unsigned long clone_flags, struct task_struct *tsk)
{
    struct fs_struct *fs = current->fs;
    if (clone_flags & CLONE_FS) {
        /* tsk->fs is already what we want */
        spin_lock(&fs->lock);
        if (fs->in_exec) {
            spin_unlock(&fs->lock);
            return -EAGAIN;
        }
        fs->users++;
        spin_unlock(&fs->lock);
        return 0;
    }
    tsk->fs = copy_fs_struct(fs);
    if (!tsk->fs)
        return -ENOMEM;
    return 0;
}
```

5、初始化信号相关变量

- 整个进程里的所有线程共享一个shared_pending，这也是一个信号列表，是发给整个进程的，哪个线程处理都一样。由此我们可以做到发给进程的信号虽然可以被一个线程处理，但是影响范围应该是整个进程的。例如，kill 一个进程，则所有线程都要被干掉。如果一个信号是发给一个线程的 pthread_kill，则应该只有线程能够收到。

- copy_sighand

- - 对于进程来说，会分配一个新的 sighand_struct。这里最主要的是维护信号处理函数，在 copy_sighand 里面会调用 memcpy，将信号处理函数 sighand->action 从父进程复制到子进程。
  - 对于线程来说，由于设计了CLONE_SIGHAND标记位，会对引用计数加一并退出，没有分配新的信号变量。

```cpp
static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
{
    struct sighand_struct *sig;
    if (clone_flags & CLONE_SIGHAND) {
        refcount_inc(&current->sighand->count);
        return 0;
    }
    sig = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);
    rcu_assign_pointer(tsk->sighand, sig);
    if (!sig)
        return -ENOMEM;
    refcount_set(&sig->count, 1);
    spin_lock_irq(&current->sighand->siglock);
    memcpy(sig->action, current->sighand->action, sizeof(sig->action));
    spin_unlock_irq(&current->sighand->siglock);
    return 0;
}
```

init_sigpending 和 copy_signal 用于初始化信号结构体，并且复制用于维护发给这个进程的信号的数据结构。copy_signal 函数会分配一个新的 signal_struct，并进行初始化。对于线程来说也是直接退出并未复制。

```cpp
static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
{
    struct signal_struct *sig;
    if (clone_flags & CLONE_THREAD)
        return 0;
    sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL);
......
    /* list_add(thread_node, thread_head) without INIT_LIST_HEAD() */
    sig->thread_head = (struct list_head)LIST_HEAD_INIT(tsk->thread_node);
    tsk->thread_node = (struct list_head)LIST_HEAD_INIT(sig->thread_head);
    init_waitqueue_head(&sig->wait_chldexit);
    sig->curr_target = tsk;
    init_sigpending(&sig->shared_pending);
    INIT_HLIST_HEAD(&sig->multiprocess);
    seqlock_init(&sig->stats_lock);
    prev_cputime_init(&sig->prev_cputime);
......
};
```

6、复制进程内存空间

- 进程都有自己的内存空间，用 mm_struct 结构来表示。copy_mm() 函数中调用 dup_mm()，分配一个新的 mm_struct 结构，调用 memcpy 复制这个结构。dup_mmap() 用于复制内存空间中内存映射的部分。前面讲系统调用的时候，我们说过，mmap 可以分配大块的内存，其实 mmap 也可以将一个文件映射到内存中，方便可以像读写内存一样读写文件，这个在内存管理那节我们讲。
- 线程不会复制内存空间，因此因为CLONE_VM标识位而直接指向了原来的mm_struct。

```cpp
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    struct mm_struct *mm, *oldmm;
    int retval;
......
    /*
    * Are we cloning a kernel thread?
    * We need to steal a active VM for that..
    */
    oldmm = current->mm;
    if (!oldmm)
        return 0;
    /* initialize the new vmacache entries */
    vmacache_flush(tsk);
    if (clone_flags & CLONE_VM) {
        mmget(oldmm);
        mm = oldmm;
        goto good_mm;
    }
    retval = -ENOMEM;
    mm = dup_mm(tsk);
    if (!mm)
        goto fail_nomem;
good_mm:
    tsk->mm = mm;
    tsk->active_mm = mm;
    return 0;
fail_nomem:
    return retval;
}
```

7、分配 pid，设置 tid，group_leader，并且建立任务之间的亲缘关系。

- group_leader：进程的话 group_leader就是它自己，和旧进程分开。线程的话则设置为当前进程的group_leader。
- tgid: 对进程来说是自己的pid，对线程来说是当前进程的pid
- real_parent : 对进程来说即当前进程，对线程来说则是当前进程的real_parent

```cpp
static __latent_entropy struct task_struct *copy_process(......) {
......    
    p->pid = pid_nr(pid);
    if (clone_flags & CLONE_THREAD) {
        p->exit_signal = -1;
        p->group_leader = current->group_leader;
        p->tgid = current->tgid;
    } else {
        if (clone_flags & CLONE_PARENT)
          p->exit_signal = current->group_leader->exit_signal;
        else
          p->exit_signal = (clone_flags & CSIGNAL);
        p->group_leader = p;
        p->tgid = p->pid;
    }
......
    if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
        p->real_parent = current->real_parent;
        p->parent_exec_id = current->parent_exec_id;
    } else {
        p->real_parent = current;
        p->parent_exec_id = current->self_exec_id;
    } 
......  
};
```

### 4.2 新进程的唤醒

```cpp
_do_fork 做的第二件大事是通过调用 wake_up_new_task()唤醒进程。

void wake_up_new_task(struct task_struct *p)
{
    struct rq_flags rf;
    struct rq *rq;
......
    p->state = TASK_RUNNING;
......
    activate_task(rq, p, ENQUEUE_NOCLOCK);
    trace_sched_wakeup_new(p);
    check_preempt_curr(rq, p, WF_FORK);
......
}
```

首先，我们需要将进程的状态设置为 TASK_RUNNING。activate_task() 函数中会调用 enqueue_task()。

```cpp
void activate_task(struct rq *rq, struct task_struct *p, int flags)
{
    if (task_contributes_to_load(p))
        rq->nr_uninterruptible--;

    enqueue_task(rq, p, flags);

    p->on_rq = TASK_ON_RQ_QUEUED;
}

static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
.....
    p->sched_class->enqueue_task(rq, p, flags);
}
```

如果是 CFS 的调度类，则执行相应的 enqueue_task_fair()。在 enqueue_task_fair() 中取出的队列就是 cfs_rq，然后调用 enqueue_entity()。在 enqueue_entity() 函数里面，会调用 update_curr()，更新运行的统计量，然后调用 __enqueue_entity，将 sched_entity 加入到红黑树里面，然后将 se->on_rq = 1 设置在队列上。回到 enqueue_task_fair 后，将这个队列上运行的进程数目加一。然后，wake_up_new_task 会调用 check_preempt_curr，看是否能够抢占当前进程。

```cpp
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &p->se;
......
    for_each_sched_entity(se) {
        if (se->on_rq)
            break;
        cfs_rq = cfs_rq_of(se);
        enqueue_entity(cfs_rq, se, flags);

        cfs_rq->h_nr_running++;
        cfs_rq->idle_h_nr_running += idle_h_nr_running;

        /* end evaluation on encountering a throttled cfs_rq */
        if (cfs_rq_throttled(cfs_rq))
            goto enqueue_throttle;

        flags = ENQUEUE_WAKEUP;
    }
......
}
```

在 check_preempt_curr 中，会调用相应的调度类的 rq->curr->sched_class->check_preempt_curr(rq, p, flags)。对于CFS调度类来讲，调用的是 check_preempt_wakeup。在 check_preempt_wakeup函数中，前面调用 task_fork_fair的时候，设置 sysctl_sched_child_runs_first 了，已经将当前父进程的 TIF_NEED_RESCHED 设置了，则直接返回。否则，check_preempt_wakeup 还是会调用 update_curr 更新一次统计量，然后 wakeup_preempt_entity 将父进程和子进程 PK 一次，看是不是要抢占，如果要则调用 resched_curr 标记父进程为 TIF_NEED_RESCHED。如果新创建的进程应该抢占父进程，在什么时间抢占呢？别忘了 fork 是一个系统调用，从系统调用返回的时候，是抢占的一个好时机，如果父进程判断自己已经被设置为 TIF_NEED_RESCHED，就让子进程先跑，抢占自己。

```cpp
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
    struct task_struct *curr = rq->curr;
    struct sched_entity *se = &curr->se, *pse = &p->se;
    struct cfs_rq *cfs_rq = task_cfs_rq(curr);
......
    if (test_tsk_need_resched(curr))
        return;
......
    find_matching_se(&se, &pse);
    update_curr(cfs_rq_of(se));
    if (wakeup_preempt_entity(se, pse) == 1) {
        goto preempt;
    }
    return;
preempt:
    resched_curr(rq);
......
}
```

至此，我们就完成了任务的整个创建过程，并根据情况唤醒任务开始执行。

## 5、总结

本文十分之长，因为内容极多，源码复杂，本来想拆分为两篇文章，但是又因为过于紧密的联系因此合在了一起。本文介绍了进程的创建和线程的创建，而多进程的派生因为使用和线程内核态创建一样的函数因此放在了一起边对比边说明。由此，进程、线程的结构体以及创建过程就全部分析完了，下文将继续分析进程、线程的调度。

-----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/442916346