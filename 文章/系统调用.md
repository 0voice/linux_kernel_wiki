## 一. 前言
  通过前面几篇文章，我们分析了从按下电源键到内核启动、完成初始化的整个过程。在后面的文章中我们将分别深入剖析Linux内核各个重要部分的源码。考虑到后面的部分我们会从用户态的代码开始入手一步一步深入，因此在分析这些之前，我们需要仔细看一看如何实现一个从用户态到内核态再回到用户态的系统调用的全过程，即系统调用的实现。

  本文的说明顺序如下：

* 首先从一个简单的例子开始分析glibc中对应的调用
* 针对32位和64位中调用的结构不同会分开两部分单独介绍，会介绍整个调用至完成的过程。即用户态->内核态->用户态
* 在整个调用过程中最重要的一步是中间访问系统调用表，该部分为了描述清楚单独拉出来最后介绍

## 二. GLIBC标准库的调用
  让我们从一个简单的程序开始
```c
#include <stdio.h>

int main(int argc, char **argv)
{
    FILE *fp;
    char buff[255];

    fp = fopen("test.txt", "r");
    fgets(buff, 255, fp);
    printf("%s\n", buff);
    fclose(fp);

    return 0;
}
```
  如上所示的程序主要调用了glibc中的函数，然后在其上进行了封装而成。比如fopen实际使用的是open，这里我们就以该函数为例来说明整个调用过程。首先open函数的系统调用在syscalls.list表中定义
```c
# File name Caller Syscall name Args Strong name Weak names
open - open Ci:siv __libc_open __open open
```
  根据此配置文件，glibc会调用脚本make_syscall.sh将其封装为宏，如SYSCALL_NAME open的形式。这些宏会通过T_PSEUDO来调用（位于syscall-template.S），而实际上使用的则是DO_CALL(syscall_name, args)
```c
T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret
T_PSEUDO_END (SYSCALL_SYMBOL)

#define T_PSEUDO(SYMBOL, NAME, N)    PSEUDO (SYMBOL, NAME, N)

#define PSEUDO(name, syscall_name, args)      \
    .text;                                    \
    ENTRY (name)                              \
    DO_CALL (syscall_name, args);             \
    cmpl $-4095, %eax;                        \
    jae SYSCALL_ERROR_LABEL
```
### 2.1 32位系统调用过程
  考虑到32位和64位代码结构有一些区别，因此这里需要分开讨论。在32位系统中，DO_CALL()位于i386 目录下的sysdep.h文件中
```c
/* Linux takes system call arguments in registers:
  syscall number  %eax       call-clobbered
  arg 1    %ebx       call-saved
  arg 2    %ecx       call-clobbered
  arg 3    %edx       call-clobbered
  arg 4    %esi       call-saved
  arg 5    %edi       call-saved
  arg 6    %ebp       call-saved
......
*/
#define DO_CALL(syscall_name, args)               \
    PUSHARGS_##args                               \
    DOARGS_##args                                 \
    movl $SYS_ify (syscall_name), %eax;           \
    ENTER_KERNEL                                  \
    POPARGS_##args
```
  这里，我们将请求参数放在寄存器里面，根据系统调用的名称，得到系统调用号，放在寄存器 eax 里面，然后执行ENTER_KERNEL。
```
# define ENTER_KERNEL int $0x80
```
  ENTER_KERNEL实际调用的是80软中断，以此陷入内核。这些中断在trap_init()中被定义并初始化。在前文中对trap_init()已有一些简单的叙述，后面在中断部分会再详细介绍。

  初始化好的中断表会等待到中断触发，触发的时候则调用对应的回调函数，这里的话就是entry_INT80_32。该中断首先通过push和SAVE_ALL保存所有的寄存器，存储在pt_regs中，然后调用do_syscall_32_irqs_on()函数。该函数将系统调用号从eax里面取出来，然后根据系统调用号，在系统调用表中找到相应的函数进行调用，并将寄存器中保存的参数取出来，作为函数参数。最后调用INTERRUPT_RETURN，实际使用的是iret指令将原来用户保存的现场包含代码段、指令指针寄存器等恢复，并返回至用户态执行。
```c
ENTRY(entry_INT80_32)
    ASM_CLAC
    pushl   %eax                    /* pt_regs->orig_ax */
    SAVE_ALL pt_regs_ax=$-ENOSYS    /* save rest */
    movl    %esp, %eax
    call    do_syscall_32_irqs_on
.Lsyscall_32_done:
......
.Lirq_return:
  INTERRUPT_RETURN
     
......     

static __always_inline void do_syscall_32_irqs_on(struct pt_regs *regs)
{
    struct thread_info *ti = current_thread_info();
    unsigned int nr = (unsigned int)regs->orig_ax;
......
    if (likely(nr < IA32_NR_syscalls)) {
        regs->ax = ia32_sys_call_table[nr](
            (unsigned int)regs->bx, (unsigned int)regs->cx,
            (unsigned int)regs->dx, (unsigned int)regs->si,
            (unsigned int)regs->di, (unsigned int)regs->bp);
    }
    syscall_return_slowpath(regs);
}
```

### 2.2 64位系统调用过程

  对于64位系统来说，DO_CALL位于x86_64 目录下的 sysdep.h 文件中
```c
/* The Linux/x86-64 kernel expects the system call parameters in
   registers according to the following table:
    syscall number  rax
    arg 1    rdi
    arg 2    rsi
    arg 3    rdx
    arg 4    r10
    arg 5    r8
    arg 6    r9
......
*/
#define DO_CALL(syscall_name, args)                \
  lea SYS_ify (syscall_name), %rax;                \
  syscall
```
  和之前一样，还是将系统调用名称转换为系统调用号，放到寄存器rax。这里是真正进行调用，不是用中断了，而是改用syscall指令了。并且，通过注释我们也可以知道，传递参数的寄存器也变了。syscall指令还使用了一种特殊的寄存器，我们叫特殊模块寄存器（Model Specific Registers，简称 MSR）。这种寄存器是 CPU 为了完成某些特殊控制功能为目的的寄存器，其中就有系统调用。

  在系统初始化的时候，trap_init() 除了初始化上面的中断模式，这里面还会调用 cpu_init->syscall_init()。这里面有这样的代码：
```c
wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```
  rdmsr() 和 wrmsr() 是用来读写特殊模块寄存器的。MSR_LSTAR 就是这样一个特殊的寄存器，当 syscall 指令调用的时候，会从这个寄存器里面拿出函数地址来调用，也就是调用 entry_SYSCALL_64。在 arch/x86/entry/entry_64.S中定义了 entry_SYSCALL_64函数。

  该函数开始于一条宏：SWAPGS_UNSAFE_STACK，其定义如下，主要是交换当前GS基寄存器中的值和特殊模块寄存器中包含的值，即进入内核栈。
```c
#define SWAPGS_UNSAFE_STACK    swapgs
```
  对于旧的栈，我们会将其存于rsp_scratch，并将栈指针移至当前进程的栈顶。
```c
movq    %rsp, PER_CPU_VAR(rsp_scratch)
movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```
  下一步，我们将栈段和旧的栈指针压入栈中
```c
pushq    $__USER_DS
pushq    PER_CPU_VAR(rsp_scratch)
```
  接下来，我们需要打开中断并保存很多寄存器到 pt_regs 结构里面，例如用户态的代码段、数据段、保存参数的寄存器，并校验当前线程的信息_TIF_WORK_SYSCALL_ENTRY，这里涉及到Linux的debugging和tracing技术，会单独在后文中详细分析。该部分代码具体如下所示。
```c
ENTRY(entry_SYSCALL_64)
    /* Construct struct pt_regs on stack */
    pushq   $__USER_DS                      /* pt_regs->ss */
    pushq   PER_CPU_VAR(rsp_scratch)        /* pt_regs->sp */
    pushq   %r11                            /* pt_regs->flags */
    pushq   $__USER_CS                      /* pt_regs->cs */
    pushq   %rcx                            /* pt_regs->ip */
    pushq   %rax                            /* pt_regs->orig_ax */
    pushq   %rdi                            /* pt_regs->di */
    pushq   %rsi                            /* pt_regs->si */
    pushq   %rdx                            /* pt_regs->dx */
    pushq   %rcx                            /* pt_regs->cx */
    pushq   $-ENOSYS                        /* pt_regs->ax */
    pushq   %r8                             /* pt_regs->r8 */
    pushq   %r9                             /* pt_regs->r9 */
    pushq   %r10                            /* pt_regs->r10 */
    pushq   %r11                            /* pt_regs->r11 */
    sub     $(6*8), %rsp                    /* pt_regs->bp, bx, r12-15 not saved */
    movq    PER_CPU_VAR(current_task), %r11
    testl   $_TIF_WORK_SYSCALL_ENTRY|_TIF_ALLWORK_MASK, TASK_TI_flags(%r11)
    jnz     entry_SYSCALL64_slow_path
  ```
  
  各寄存器的作用如下所示：

* rax：系统调用数目
* rcx：函数返回的用户空间地址
* r11：寄存器标记
* rdi：系统调用回调函数的第一个参数
* rsi：系统调用回调函数的第二个参数
* rdx：系统调用回调函数的第三个参数
* r10：系统调用回调函数的第四个参数
* r8 ：系统调用回调函数的第五个参数
* r9 ：系统调用回调函数的第六个参数
* rbp,rbx,r12-r15：通用的callee-preserved寄存器

  在此之后，其实存在着两个处理分支：entry_SYSCALL64_slow_path 和 entry_SYSCALL64_fast_path，这里是根据_TIF_WORK_SYSCALL_ENTRY判断的结果进行选择，这里涉及到ptrace部分的知识，暂时先不介绍了，会在后面单独开一文详细研究。如果设置了_TIF_ALLWORK_MASK或者_TIF_WORK_SYSCALL_ENTRY，则跳转至slow_path，否则继续运行fast_path。
```c
#define _TIF_WORK_SYSCALL_ENTRY \
    (_TIF_SYSCALL_TRACE | _TIF_SYSCALL_EMU | _TIF_SYSCALL_AUDIT |   \
    _TIF_SECCOMP | _TIF_SINGLESTEP | _TIF_SYSCALL_TRACEPOINT |     \
    _TIF_NOHZ)
```

#### 2.2.1 fastpath分支
  该分支主要分为以下部分内容

* 再次检测TRACE部分，如果有标记则跳转至slow_path
* 检测__SYSCALL_MASK，如果CONFIG_X86_X32_ABI未设置我们就比较rax寄存器的值和最大系统调用数__NR_syscall_max，否则则标记eax寄存器为__x32_SYSCALL_BIT，再进行比较
* ja指令会在CF和ZF设置为0时进行跳转，即如果不满足条件则会跳转至-ENOSYS，否则继续执行
* 将第四个参数从r10放入rcx以保持x86_64 C ABI编译
* 执行sys_call_table，去系统调用表中查找系统调用

```c
entry_SYSCALL_64_fastpath:
    /*
     * Easy case: enable interrupts and issue the syscall.  If the syscall
     * needs pt_regs, we'll call a stub that disables interrupts again
     * and jumps to the slow path.
     */
    TRACE_IRQS_ON
    ENABLE_INTERRUPTS(CLBR_NONE)
    
#if __SYSCALL_MASK == ~0
    cmpq	$__NR_syscall_max, %rax
#else
    andl	$__SYSCALL_MASK, %eax
    cmpl	$__NR_syscall_max, %eax
#endif

    ja	1f				/* return -ENOSYS (already in pt_regs->ax) */
    movq	%r10, %rcx

    /*
    * This call instruction is handled specially in stub_ptregs_64.
    * It might end up jumping to the slow path.  If it jumps, RAX
    * and all argument registers are clobbered.
    */
    call	*sys_call_table(, %rax, 8)
    
......
# ifdef CONFIG_X86_X32_ABI
#  define __SYSCALL_MASK (~(__X32_SYSCALL_BIT))
# else
#  define __SYSCALL_MASK (~0)
# endif

#define __X32_SYSCALL_BIT    0x40000000
```

#### 2.2.2 slow_path分支
  slow_path部分的源码如下
```c
entry_SYSCALL64_slow_path:
    /* IRQs are off. */
    SAVE_EXTRA_REGS
    movq    %rsp, %rdi
    call    do_syscall_64           /* returns with IRQs disabled */
```

  slow_path会调用entry_SYSCALL64_slow_pat->do_syscall_64()，执行完毕后恢复寄存器，最后调用USERGS_SYSRET64，实际使用sysretq指令返回。

```c
return_from_SYSCALL_64:
    RESTORE_EXTRA_REGS
    TRACE_IRQS_IRETQ
    movq  RCX(%rsp), %rcx
    movq  RIP(%rsp), %r11
    movq  R11(%rsp), %r11
......
syscall_return_via_sysret:
    /* rcx and r11 are already restored (see code above) */
    RESTORE_C_REGS_EXCEPT_RCX_R11
    movq  RSP(%rsp), %rsp
    USERGS_SYSRET64
```
  在 do_syscall_64 里面，从 rax里面拿出系统调用号，然后根据系统调用号，在系统调用表 sys_call_table 中找到相应的函数进行调用，并将寄存器中保存的参数取出来，作为函数参数。
```c
__visible void do_syscall_64(struct pt_regs *regs)
{
    struct thread_info *ti = current_thread_info();
    unsigned long nr = regs->orig_ax;
......
    if (likely((nr & __SYSCALL_MASK) < NR_syscalls)) {
        regs->ax = sys_call_table[nr & __SYSCALL_MASK](
                    regs->di, regs->si, regs->dx,
                    regs->r10, regs->r8, regs->r9);
    }
    syscall_return_slowpath(regs);
}
```
  至此，32位和64位又回到了同样的位置：查找系统调用表sys_call_table。

## 三. 系统调用表的生成
  32位和64位的sys_call_table均位于arch/x86/entry/syscalls/目录下，分别为syscall_32.tbl和syscall_64.tbl。如下所示为32位和64位中open函数的定义
```c
5 i386 open sys_open compat_sys_open

2 common open sys_open
```
  第一列的数字是系统调用号。可以看出，32 位和 64 位的系统调用号是不一样的。第三列是系统调用的名字，第四列是系统调用在内核的实现函数。不过，它们都是以 sys_ 开头。系统调用在内核中的实现函数要有一个声明。声明往往在 include/linux/syscalls.h文件中。例如 sys_open 是这样声明的：
```c
asmlinkage long sys_open(const char __user *filename,
                                int flags, umode_t mode);
```
  真正的实现这个系统调用，一般在一个.c 文件里面，例如 sys_open 的实现在 fs/open.c 里面。其中采用了宏的方式对函数名进行了封装，实际拆开是一样的。
```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
    if (force_o_largefile())
        flags |= O_LARGEFILE;
    return do_sys_open(AT_FDCWD, filename, flags, mode);
}

......

asmlinkage long sys_open(const char __user * filename, int flags, int mode)
{
    long ret;
    
    if (force_o_largefile())
        flags |= O_LARGEFILE;
    
    ret = do_sys_open(AT_FDCWD, filename, flags, mode);
    asmlinkage_protect(3, ret, filename, flags, mode);
    return ret;
}
```
  其中SYSCALL_DEFINE3 是一个宏系统调用最多六个参数，根据参数的数目选择宏。具体是这样定义如下所示，首先使用SYSCALL_METADATA()宏解决syscall_metada结构体的初始化，该结构体包括了不同的有用区域包括系统调用的名字、系统调用表中对应的序号、系统调用的参数、参数类型链表等。
```c
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)                          		\
        SYSCALL_METADATA(sname, x, __VA_ARGS__)                 		\
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

#define __PROTECT(...) asmlinkage_protect(__VA_ARGS__)
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       \
                __attribute__((alias(__stringify(SyS##name))));         \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
        {                                                               \
                long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
                __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                return ret;                                             \
        }                                                               \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__)

..........
    
#define SYSCALL_METADATA(sname, nb, ...)                             \
    ...                                                              \
    ...                                                              \
    ...                                                              \
    struct syscall_metadata __used                                   \
              __syscall_meta_##sname = {                             \
                    .name           = "sys"#sname,                   \
                    .syscall_nr     = -1,                            \
                    .nb_args        = nb,                            \
                    .types          = nb ? types_##sname : NULL,     \
                    .args           = nb ? args_##sname : NULL,      \
                    .enter_event    = &event_enter_##sname,          \
                    .exit_event     = &event_exit_##sname,           \
                    .enter_fields   = LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), 				   \
             };                                                                            																		\

    static struct syscall_metadata __used                            \
              __attribute__((section("__syscalls_metadata")))        \
             *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```
  在编译的过程中，需要根据 syscall_32.tbl 和 syscall_64.tbl 生成自己的syscalls_32.h 和 syscalls_64.h。生成方式在 arch/x86/entry/syscalls/Makefile 中。这里面会使用两个脚本

第一个脚本arch/x86/entry/syscalls/syscallhdr.sh，会在文件中生成 #define __NR_open；

第二个脚本 arch/x86/entry/syscalls/syscalltbl.sh，会在文件中生成__SYSCALL(__NR_open, sys_open)。

这样最终生成syscalls_32.h 和 syscalls_64.h 就保存了系统调用号和系统调用实现函数之间的对应关系，如下所示
```c
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
__SYSCALL_COMMON(2, sys_open, sys_open)
__SYSCALL_COMMON(3, sys_close, sys_close)
__SYSCALL_COMMON(5, sys_newfstat, sys_newfstat)
...
...
...
```
  其中__SYSCALL_COMMON宏定义如下，主要是将对应的数字序号和系统调用名对应
```c
#define __SYSCALL_COMMON(nr, sym, compat) __SYSCALL_64(nr, sym, compat)
#define __SYSCALL_64(nr, sym, compat) [nr] = sym,
```
  最终形成的表如下
```c
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
    [0] = sys_read,
    [1] = sys_write,
    [2] = sys_open,
    ...
    ...
    ...
};
```
  最后，所有的系统调用会存储在arch/x86/entry/目录下的syscall_32.c和syscall_64.c中，里面包含了syscalls_32.h 和 syscalls_64.h 头文件，其形式如下：
```c
__visible const sys_call_ptr_t ia32_sys_call_table[__NR_syscall_compat_max+1] = {
        /*
         * Smells like a compiler bug -- it doesn't work
         * when the & below is removed.
         */
        [0 ... __NR_syscall_compat_max] = &sys_ni_syscall,
#include <asm/syscalls_32.h>
};

/* System call table for x86-64. */
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
  /*
   * Smells like a compiler bug -- it doesn't work
   * when the & below is removed.
   */
  [0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_64.h>
};
```
  其中__NR_syscall_max宏定义规定了最大系统调用数量，该数量取决于操作系统的架构，在X86下定义如下
  
```c
#define __NR_syscall_max 547
```

  这里还需要注意sys_call_ptr_t表示指向系统调用表的指针，定义为函数指针
```c
typedef void (*sys_call_ptr_t)(void);
```
  系统调用表数组中的每一个系统调用均会指向sys_ni_syscall，该函数表示一个未实现的系统调用（not-implement），从而系统调用表的初始化。
```c
asmlinkage long sys_ni_syscall(void)
{
    return -ENOSYS;
}

ENOSYS Function not implemented (POSIX.1)
```
  由此，整个系统调用表的生成过程就全部说明完了，而在实际产生系统调用的时候，过程则刚好相反：

* 用户态调用syscall
* syscall导致中断，程序由用户态陷入内核态
* 内核C函数执行syscalls_32/64.c，并由此获得对应关系最终在对应的源码中找到函数实现
* 针对对应的sys_syscall_name函数，做好调用准备工作，如初始化系统调用入口、保存寄存器、切换新的栈、构造新的task以备中断回调等。
* 调用函数实现
* 切换寄存器、栈，返回用户态

## 四. 总结
  本文较为深入的分析了系统调用的整个过程，并着重分析了系统调用表的形成和使用原理，如有遗漏错误还请多多指正。
