内核为每个进程分配一个task_struct结构时，实际上分配两个连续的物理页面(8192字节)，如图所示。底部用作task_struct结构(大小约为1K字节)，结构的上面用作内核堆栈(大小约为7K字节)。访问进程自身的task_struct结构，使用宏操作current, 在2.4中定义如下：

![img](https://pic2.zhimg.com/80/v2-175f0020378c22ce70736eb4889c32e5_720w.webp)

根据内核的配置，THREAD_SIZE既可以是4K字节(1个页面)也可以是8K字节(2个页面)。thread_info是52个字节长。
下图是当设为8KB时候的内核堆栈：Thread_info在这个内存区的开始处，内核堆栈从末端向下增长。进程描述符不是在这个内存区中，而分别通过task与thread_info指针使thread_info与进程描述符互联。所以获得当前进程描述符的current定义如下:

![img](https://pic2.zhimg.com/80/v2-883080a042dcaaba752d1603f81f3ab9_720w.webp)

下面是thread_info结构体的定义：

```text
struct thread_info {
        struct task_struct    *task;           /* main task structure */
        struct exec_domain    *exec_domain;    /* execution domain */
        __u32            flags;                /* low level flags */
        __u32            status;               /* thread synchronous flags */
        __u32            cpu;                  /* current CPU */
        int            preempt_count;          /* 0 => preemptable, <0 => BUG */
        mm_segment_t            addr_limit;
        struct restart_block     restart_block;
        void __user             *sysenter_return;
    #ifdef CONFIG_X86_32
        unsigned long previous_esp; /* ESP of the previous stack in
                                       case of nested (IRQ) stacks
                                       */
        __u8                supervisor_stack[0];
    #endif
        unsigned int        sig_on_uaccess_error:1;
        unsigned int        uaccess_err:1;    /* uaccess failed */
    };
```

可以看到在thread_info中个task_struct结构体，里面包含的是进程描述符，其他的参数如下：（可以略过）

(1) unsigned short used_math;

是否使用FPU。

(2) char comm[16];

进程正在运行的可执行文件的文件名。

(3) struct rlimit rlim[RLIM_NLIMITS];

结 构rlimit用于资源管理，定义在linux/include/linux/resource.h中，成员共有两项:rlim_cur是资源的当前最大 数目;rlim_max是资源可有的最大数目。在i386环境中，受控资源共有RLIM_NLIMITS项，即10项，定义在 linux/include/asm/resource.h中，见下表:

(4) int errno;

最后一次出错的系统调用的错误号，0表示无错误。系统调用返回时，全程量也拥有该错误号。

(5) long debugreg[8];

保存INTEL CPU调试寄存器的值，在ptrace系统调用中使用。

(6) struct exec_domain *exec_domain;

Linux可以运行由80386平台其它UNIX操作系统生成的符合iBCS2标准的程序。关于此类程序与Linux程序差异的消息就由 exec_domain结构保存。

(7) unsigned long personality;

Linux 可以运行由80386平台其它UNIX操作系统生成的符合iBCS2标准的程序。 Personality进一步描述进程执行的程序属于何种UNIX平台的“个性”信息。通常有PER_Linux、PER_Linux_32BIT、 PER_Linux_EM86、PER_SVR3、PER_SCOSVR3、PER_WYSEV386、PER_ISCR4、PER_BSD、 PER_XENIX和PER_MASK等，参见include/linux/personality.h。

(8) struct linux_binfmt *binfmt;

指向进程所属的全局执行文件格式结构，共有a。out、script、elf和java等四种。结构定义在include/linux /binfmts.h中(core_dump、load_shlib(fd)、load_binary、use_count)。

(9) int exit_code，exit_signal;

引起进程退出的返回代码exit_code，引起错误的信号名exit_signal。

(10) int dumpable:1;

布尔量，表示出错时是否可以进行memory dump。

(11) int did_exec:1;

按POSIX要求设计的布尔量，区分进程是正在执行老程序代码，还是在执行execve装入的新代码。

(12) int tty_old_pgrp;

进程显示终端所在的组标识。

(13) struct tty_struct *tty;

指向进程所在的显示终端的信息。如果进程不需要显示终端，如0号进程，则该指针为空。结构定义在include/linux/tty.h中。

(14) struct wait_queue *wait_chldexit;

在进程结束时，或发出系统调用wait4后，为了等待子进程的结束，而将自己(父进程)睡眠在该队列上。结构定义在include/linux /wait.h中。

**13. 进程队列的全局变量**

(1) current;

当前正在运行的进程的指针，在SMP中则指向CPU组中正被调度的CPU的当前进程:

\#define current(0+current_set[smp_processor_id()])/*sched.h*/

struct task_struct *current_set[NR_CPUS];

(2) struct task_struct init_task;

即0号进程的PCB，是进程的“根”，始终保持初值INIT_TASK。

(3) struct task_struct *task[NR_TASKS];

进程队列数组，规定系统可同时运行的最大进程数(见kernel/sched.c)。NR_TASKS定义在include/linux/tasks.h 中，值为512。每个进程占一个数组元素(元素的下标不一定就是进程的pid)，task[0]必须指向init_task(0号进程)。可以通过 task[]数组遍历所有进程的PCB。但Linux也提供一个宏定义for_each_task()(见 include/linux/sched.h)，它通过next_task遍历所有进程的PCB:

\#define for_each_task(p) \

for(p=&init_task;(p=p->next_task)!=&init_task;)

(4) unsigned long volatile jiffies;

Linux的基准时间(见kernal/sched.c)。系统初始化时清0，以后每隔10ms由时钟中断服务程序do_timer()增1。

(5) int need_resched;

重新调度标志位(见kernal/sched.c)。当需要Linux调度时置位。在系统调用返回前(或者其它情形下)，判断该标志是否置位。置位的话，马上调用schedule进行CPU调度。

(6) unsigned long intr_count;

记 录中断服务程序的嵌套层数(见kernal/softirq.c)。正常运行时，intr_count为0。当处理硬件中断、执行任务队列中的任务或者执 行bottom half队列中的任务时，intr_count非0。这时，内核禁止某些操作，例如不允许重新调度。

接下来写一个模块，打印当前的进程名字：

```text
#include <linux/init.h>
#include <linux/thread_info.h>
#include <linux/module.h>
#include <linux/sched.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("binary_tree");

int test_init()
{
    printk("hello binary_tree!\n");

    int i=0;
    struct thread_info *  info;
    struct task_struct *  t;

    unsigned long addr =(unsigned long)&i;//
    unsigned long base = addr & ~ 0x1fff;//屏蔽低13位  8K
    info = (struct thread_info *)base;//
    t=  info -> task;
    printk("it is name is %s\n",t-> comm);//打印出进程名字
    return 0;
}

void test_exit()
{
    printk("bye! bye ! binary_tree! \n");
}


module_init(test_init);
module_exit(test_exit);
```

在栈中，struct_task的存在方式如下：

![img](https://pic3.zhimg.com/80/v2-9383324bde77195778798d571aac74a2_720w.webp)

可以看到在struct_task中有一个staks的结构体，就是一个循环双向链表，因此，我们可以模拟出一个PS命令：

（今天布置的习题）

```text
#include <linux/init.h>
#include <linux/thread_info.h>
#include <linux/module.h>
#include <linux/sched.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("bunfly");

int test_init()
{
    int i = 0;
    struct task_struct *t;
    struct thread_info *info;

    unsigned long addr = (unsigned long)&i;
    unsigned long base = addr & ~0x1fff;
    info = (struct thread_info *)base;
    t = info->task;

    struct task_struct* flag  = t;
    struct task_struct *next = container_of(t->tasks.next, struct task_struct, tasks);
                                            //首地址    子类的类型， 父类
    //获得t下一个进程符next;
    printk("now comm is %s\n",t->comm);

    struct task_struct *nnext = container_of(next->tasks.next, struct task_struct, tasks);
    //获得next下一个进程nnext
    while(nnext  != flag)
    {
        next=nnext;
        nnext=container_of(next->tasks.next, struct task_struct, tasks);
        printk("next comm is %s\n",next->comm);
    }
    //依次循环打印下一个进程
    printk("nnext comm is %s\n",nnext->comm);
    return 0;
}

void test_exit()
{
    printk("exit\n");
}

module_init(test_init);
module_exit(test_exit);
```

在代码中，用到一个函数：container_of(ptr,type,mem),其中三个参数可分别描述为：ptr, type,mem,分别代表：

**ptr**:父类在子类对象中的首地址;

**type**:子类的类型：

**mem**:父类的实体（成员）:

![img](https://pic4.zhimg.com/80/v2-9c3883b480a5eac7f33fa7a91a36feeb_720w.webp)

其具体的功能就是求下面是对container_of函数的具体实现：（重点讲解的）

```text
1 #include<stdio.h>
  2 
  3 #define container_of(ptr,type,mem) (type*)((unsigned long )(ptr)-(unsigned long)(&((type*)0)->mem));
  4 struct person
  5 {   
  6     int age;
  7     struct person* next;
  8 };
  9 
 10 struct man
 11 {   
 12     int len;
 13     int size;
 14     char name;
 15     struct person p;
 16 };
 17 
 18 int main()
 19 {   
 20     struct man haha;
 21     
 22     haha.len=100;
 23     haha.p.age=20;
 24     struct man* head = &haha.p;//已知父类在子类中的首地址　　　　//3
 25 //  struct man* tmp=malloc(1);
 26 //  int size = (unsigned long)(&tmp->p)-(unsigned long)(tmp);
 27 　　　　//2
 28 //  struct man* tmp=0;
 29 //  int size = (unsigned long)(&tmp->p);
 30 //  struct man* m=(struct man*)( (unsigned long )(head)-size );
 31     　　　　　//1
 32     //struct man* tmp=0;
 33     //int size = (unsigned long)(&tmp->p);
 34     //struct man* m=(struct man*)( (unsigned long )(head)-size );
 35    
 36    // struct man* m=(struct man*)((unsigned long )(head)-(unsigned long)(&((struct man*)0)->p));
 37     struct man* m = container_of(head,struct man,p);
 38 
 39 //  printf("head addr is  %p\n",&head);
 40 //  printf("m addr is  %p\n",&m);
 41 //  printf("head of len is %d\n",head->len);
 42 //  printf("head of age is %d\n",(head->p.age)  );
 43     
 44     printf("m of len is %d\n",m->len);
 45     printf("m of age is %d\n",m->p.age);
 46 
 47 }
~
```

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549156208