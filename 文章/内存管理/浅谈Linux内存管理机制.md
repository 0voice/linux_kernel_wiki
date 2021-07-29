## 活学活用

OOM Killer 在 Linux 系统里如果内存不足时，会杀死一个正在运行的进程来释放一些内存。

Linux 里的程序都是调用 malloc() 来申请内存，如果内存不足，直接 malloc() 返回失败就可以，为什么还要去杀死正在运行的进程呢？Linux允许进程申请超过实际物理内存上限的内存。因为 malloc() 申请的是内存的虚拟地址，系统只是给了程序一个地址范围，由于没有写入数据，所以程序并没有得到真正的物理内存。物理内存只有程序真的往这个地址写入数据的时候，才会分配给程序。

![image](https://user-images.githubusercontent.com/87457873/127455549-4c2bec21-7aea-4f5f-af1a-99f0774047f5.png)

## 内存管理

对于内存的访问，用户态的进程使用虚拟地址，内核的也基本都是使用虚拟地址

## 物理内存空间布局

![image](https://user-images.githubusercontent.com/87457873/127455595-9ab58b7a-2400-4322-85bf-70a9fd7ba837.png)

## 虚拟内存与物理内存的映射

![image](https://user-images.githubusercontent.com/87457873/127455619-abb98623-fa06-4d21-849a-3fd066e9bf44.png)

## 进程“独占”虚拟内存及虚拟内存划分

为了保证操作系统的稳定性和安全性。用户程序不可以直接访问硬件资源，如果用户程序需要访问硬件资源，必须调用操作系统提供的接口，这个调用接口的过程也就是系统调用。每一次系统调用都会存在两个内存空间之间的相互切换，通常的网络传输也是一次系统调用，通过网络传输的数据先是从内核空间接收到远程主机的数据，然后再从内核空间复制到用户空间，供用户程序使用。这种从内核空间到用户空间的数据复制很费时，虽然保住了程序运行的安全性和稳定性，但是牺牲了一部分的效率。

如何分配用户空间和内核空间的比例也是一个问题，是更多地分配给用户空间供用户程序使用，还是首先保住内核有足够的空间来运行。在当前的Windows 32位操作系统中，默认用户空间：内核空间的比例是1:1，而在32位Linux系统中的默认比例是3:1（3GB用户空间、1GB内核空间）（这里只是地址空间，映射到物理地址，可没有某个物理地址的内存只能存储内核态数据或用户态数据的说法）。

![image](https://user-images.githubusercontent.com/87457873/127455684-ba618153-36ba-45bb-a140-0c61eccda417.png)

![image](https://user-images.githubusercontent.com/87457873/127455693-b47c1227-0223-48f0-afdd-9057abac5753.png)

左右两侧均表示虚拟地址空间，左侧以描述内核空间为主，右侧以描述用户空间为主。

![image](https://user-images.githubusercontent.com/87457873/127455885-e3183e52-b9a5-4b06-abaf-b5e583c330e6.png)

在内核里面也会有内核的代码，同样有 Text Segment、Data Segment 和 BSS Segment，别忘了内核代码也是 ELF 格式的。

**在代码上的体现**

```c
// 持有task_struct 便可以访问进程在内存中的所有数据
struct task_struct {
    ...
    struct mm_struct                *mm;
    struct mm_struct                *active_mm;
    ...
    void  *stack;   // 指向内核栈的指针
}
```

内核使用内存描述符mm_struct来表示进程的地址空间，该描述符表示着进程所有地址空间的信息

![image](https://user-images.githubusercontent.com/87457873/127455987-7785f4e9-b33c-46be-bf3c-423f4bfe9a20.png)

在用户态，进程觉着整个空间是它独占的，没有其他进程存在。但是到了内核里面，无论是从哪个进程进来的，看到的都是同一个内核空间，看到的都是同一个进程列表。虽然内核栈是各用个的，但是如果想知道的话，还是能够知道每个进程的内核栈在哪里的。所以，如果要访问一些公共的数据结构，需要进行锁保护。

![image](https://user-images.githubusercontent.com/87457873/127456009-3467796e-02ce-4a58-8006-44493dad7090.png)

## 地址空间内的栈

栈是主要用途就是支持函数调用。

大多数的处理器架构，都有实现硬件栈。有专门的栈指针寄存器，以及特定的硬件指令来完成 入栈/出栈 的操作。

### 用户栈和内核栈的切换

内核在创建进程的时候，在创建task_struct的同时，会为进程创建相应的堆栈。每个进程会有两个栈，一个用户栈，存在于用户空间，一个内核栈，存在于内核空间。当进程在用户空间运行时，cpu堆栈指针寄存器里面的内容是用户堆栈地址，使用用户栈；当进程在内核空间时，cpu堆栈指针寄存器里面的内容是内核栈空间地址，使用内核栈。

当进程因为中断或者系统调用而陷入内核态之行时，进程所使用的堆栈也要从用户栈转到内核栈。

如何相互切换呢？

进程陷入内核态后，先把用户态堆栈的地址保存在内核栈之中，然后设置堆栈指针寄存器的内容为内核栈的地址，这样就完成了用户栈向内核栈的转换；当进程从内核态恢复到用户态执行时，在内核态执行的最后，将保存在内核栈里面的用户栈的地址恢复到堆栈指针寄存器即可。这样就实现了内核栈和用户栈的互转。

那么，我们知道从内核转到用户态时用户栈的地址是在陷入内核的时候保存在内核栈里面的，但是在陷入内核的时候，我们是如何知道内核栈的地址的呢？

关键在进程从用户态转到内核态的时候，进程的内核栈总是空的。这是因为，一旦进程从内核态返回到用户态后，内核栈中保存的信息无效，会全部恢复。因此，每次进程从用户态陷入内核的时候得到的内核栈都是空的，直接把内核栈的栈顶地址给堆栈指针寄存器就可以了。

### 为什么需要单独的进程内核栈？

内核地址空间所有进程空闲，但内核栈却不共享。为什么需要单独的进程内核栈？因为同时可能会有多个进程在内核运行。

所有进程运行的时候，都可能通过系统调用陷入内核态继续执行。假设第一个进程 A 陷入内核态执行的时候，需要等待读取网卡的数据，主动调用 schedule() 让出 CPU；此时调度器唤醒了另一个进程 B，碰巧进程 B 也需要系统调用进入内核态。那问题就来了，如果内核栈只有一个，那进程 B 进入内核态的时候产生的压栈操作，必然会破坏掉进程 A 已有的内核栈数据；一但进程 A 的内核栈数据被破坏，很可能导致进程 A 的内核态无法正确返回到对应的用户态了。

进程内核栈在进程创建的时候，通过 slab 分配器从 thread_info_cache 缓存池中分配出来，其大小为 THREAD_SIZE，一般来说是一个页大小 4K；

### 进程切换带来的用户栈切换和内核栈切换

```c
// 持有task_struct 便可以访问进程在内存中的所有数据
struct task_struct {
    ...
    struct mm_struct                *mm;
    struct mm_struct                *active_mm;
    ...
    void  *stack;   // 指向内核栈的指针
}
```

从进程 A 切换到进程 B，用户栈要不要切换呢？当然要，在切换内存空间的时候就切换了，每个进程的用户栈都是独立的，都在内存空间里面。

那内核栈呢？已经在 __switch_to 里面切换了，也就是将 current_task 指向当前的 task_struct。里面的 void *stack 指针，指向的就是当前的内核栈。

内核栈的栈顶指针呢？在 __switch_to_asm 里面已经切换了栈顶指针，并且将栈顶指针在 __switch_to加载到了 TSS 里面。

用户栈的栈顶指针呢？如果当前在内核里面的话，它当然是在内核栈顶部的 pt_regs 结构里面呀。当从内核返回用户态运行的时候，pt_regs 里面有所有当时在用户态的时候运行的上下文信息，就可以开始运行了。

主线程的用户栈和一般现成的线程栈

![image](https://user-images.githubusercontent.com/87457873/127456263-0f0dcfb0-22d3-4daf-bc33-bc228de941a4.png)

对应着jvm 一个线程一个栈

### 中断栈

中断有点类似于我们经常说的事件驱动编程，而这个事件通知机制是怎么实现的呢，硬件中断的实现通过一个导线和 CPU 相连来传输中断信号，软件上会有特定的指令，例如执行系统调用创建线程的指令，而 CPU 每执行完一个指令，就会检查中断寄存器中是否有中断，如果有就取出然后执行该中断对应的处理程序。

当系统收到中断事件后，进行中断处理的时候，也需要中断栈来支持函数调用。由于系统中断的时候，系统当然是处于内核态的，所以中断栈是可以和内核栈共享的。但是具体是否共享，这和具体处理架构密切相关。ARM 架构就没有独立的中断栈。

## 内存管理的进程和硬件背景

### 页表的位置

每个进程都有独立的地址空间，为了这个进程独立完成映射，每个进程都有独立的进程页表，这个页表的最顶级的 pgd 存放在 task_struct 中的 mm_struct 的 pgd 变量里面。

在一个进程新创建的时候，会调用 fork，对于内存的部分会调用 copy_mm，里面调用 dup_mm。

```c
// Allocate a new mm structure and copy contents from the mm structure of the passed in task structure.
static struct mm_struct *dup_mm(struct task_struct *tsk){
    struct mm_struct *mm, *oldmm = current->mm;
    mm = allocate_mm();
    memcpy(mm, oldmm, sizeof(*mm));
    if (!mm_init(mm, tsk, mm->user_ns))
        goto fail_nomem;
    err = dup_mmap(mm, oldmm);
    return mm;
}
```

除了创建一个新的 mm_struct，并且通过memcpy将它和父进程的弄成一模一样之外，我们还需要调用 mm_init 进行初始化。接下来，mm_init 调用 mm_alloc_pgd，分配全局页目录项，赋值给mm_struct 的 pdg 成员变量。

```c
static inline int mm_alloc_pgd(struct mm_struct *mm){
    mm->pgd = pgd_alloc(mm);
    return 0;
}
```

一个进程的虚拟地址空间包含用户态和内核态两部分。为了从虚拟地址空间映射到物理页面，页表也分为用户地址空间的页表和内核页表。在内核里面，映射靠内核页表，这里内核页表会拷贝一份到进程的页表

如果是用户态进程页表，会有 mm_struct 指向进程顶级目录 pgd，对于内核来讲，也定义了一个 mm_struct，指向 swapper_pg_dir（指向内核最顶级的目录 pgd）。

```c
struct mm_struct init_mm = {
    .mm_rb		= RB_ROOT,
    // pgd 页表最顶级目录
    .pgd		= swapper_pg_dir,
    .mm_users	= ATOMIC_INIT(2),
    .mm_count	= ATOMIC_INIT(1),
    .mmap_sem	= __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist		= LIST_HEAD_INIT(init_mm.mmlist),
    .user_ns	= &init_user_ns,
    INIT_MM_CONTEXT(init_mm)
};

```

### 页表的应用

一个进程 fork 完毕之后，有了内核页表（内核初始化时即弄好了内核页表， 所有进程共享），有了自己顶级的 pgd，但是对于用户地址空间来讲，还完全没有映射过（用户空间页表一开始是不完整的，只有最顶级目录pgd这个“光杆司令”）。这需要等到这个进程在某个 CPU 上运行，并且对内存访问的那一刻了

当这个进程被调度到某个 CPU 上运行的时候，要调用 context_switch 进行上下文切换。对于内存方面的切换会调用 switch_mm_irqs_off，这里面会调用 load_new_mm_cr3。

cr3 是 CPU 的一个寄存器，它会指向当前进程的顶级 pgd。如果 CPU 的指令要访问进程的虚拟内存，它就会自动从cr3 里面得到 pgd 在物理内存的地址，然后根据里面的页表解析虚拟内存的地址为物理内存，从而访问真正的物理内存上的数据。

这里需要注意两点。第一点，cr3 里面存放当前进程的顶级 pgd，这个是硬件的要求。cr3 里面需要存放 pgd 在物理内存的地址，不能是虚拟地址。第二点，用户进程在运行的过程中，访问虚拟内存中的数据，会被 cr3 里面指向的页表转换为物理地址后，才在物理内存中访问数据，这个过程都是在用户态运行的，地址转换的过程无需进入内核态。

![image](https://user-images.githubusercontent.com/87457873/127456588-09c55439-71a1-4726-9e8c-b1779dd61db8.png)

这就可以解释，为什么页表数据在 task_struct 的mm_struct里却又 可以融入硬件地址翻译机制了。

### 通过缺页中断来“填充”页表

内存管理并不直接分配物理内存，只有等你真正用的那一刻才会开始分配。只有访问虚拟内存的时候，发现没有映射多物理内存，页表也没有创建过，才触发缺页异常。进入内核调用 do_page_fault，一直调用到 __handle_mm_fault，__handle_mm_fault 调用 pud_alloc 和 pmd_alloc，来创建相应的页目录项，最后调用 handle_pte_fault 来创建页表项。

```c
static noinline void
__do_page_fault(struct pt_regs *regs, unsigned long error_code,
        unsigned long address){
    struct vm_area_struct *vma;
    struct task_struct *tsk;
    struct mm_struct *mm;
    tsk = current;
    mm = tsk->mm;
    // 判断缺页是否发生在内核
    if (unlikely(fault_in_kernel_space(address))) {
        if (vmalloc_fault(address) >= 0)
            return;
    }
    ......
    // 找到待访问地址所在的区域 vm_area_struct
    vma = find_vma(mm, address);
    ......
    fault = handle_mm_fault(vma, address, flags);
    ......

static int __handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
        unsigned int flags){
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .flags = flags,
        .pgoff = linear_page_index(vma, address),
        .gfp_mask = __get_fault_gfp_mask(vma),
    };
    struct mm_struct *mm = vma->vm_mm;
    pgd_t *pgd;
    p4d_t *p4d;
    int ret;
    pgd = pgd_offset(mm, address);
    p4d = p4d_alloc(mm, pgd, address);
    ......
    vmf.pud = pud_alloc(mm, p4d, address);
    ......
    vmf.pmd = pmd_alloc(mm, vmf.pud, address);
    ......
    return handle_pte_fault(&vmf);
}
```

以handle_pte_fault 的一种场景 do_anonymous_page为例：先通过 pte_alloc 分配一个页表项，然后通过 alloc_zeroed_user_highpage_movable 分配一个页，接下来要调用 mk_pte，将页表项指向新分配的物理页，set_pte_at 会将页表项塞到页表里面。

```c
static int do_anonymous_page(struct vm_fault *vmf){
    struct vm_area_struct *vma = vmf->vma;
    struct mem_cgroup *memcg;
    struct page *page;
    int ret = 0;
    pte_t entry;
    ......
    if (pte_alloc(vma->vm_mm, vmf->pmd, vmf->address))
        return VM_FAULT_OOM;
    ......
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    ......
    entry = mk_pte(page, vma->vm_page_prot);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry));
    vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, vmf->address,
            &vmf->ptl);
    ......
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
    ......
}

```
