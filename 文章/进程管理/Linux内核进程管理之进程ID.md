Linux 内核使用 `task_struct` 数据结构来关联所有与进程有关的数据和结构，Linux 内核所有涉及到进程和程序的所有算法都是围绕该数据结构建立的，是内核中最重要的数据结构之一。该数据结构在内核文件 `include/linux/sched.h` 中定义，在Linux 3.8 的内核中，该数据结构足足有 380 行之多，在这里我不可能逐项去描述其表示的含义，本篇文章只关注该数据结构如何来组织和管理进程ID的。

## 1、**进程ID类型**

要想了解内核如何来组织和管理进程ID，先要知道进程ID的类型：

- **PID**：这是 Linux 中在其命名空间中唯一标识进程而分配给它的一个号码，称做进程ID号，简称PID。在使用 fork 或 clone 系统调用时产生的进程均会由内核分配一个新的唯一的PID值。
- **TGID**：在一个进程中，如果以CLONE_THREAD标志来调用clone建立的进程就是该进程的一个线程，它们处于一个线程组，该线程组的ID叫做TGID。处于相同的线程组中的所有进程都有相同的TGID；线程组组长的TGID与其PID相同；一个进程没有使用线程，则其TGID与PID也相同。
- **PGID**：另外，独立的进程可以组成进程组（使用setpgrp系统调用），进程组可以简化向所有组内进程发送信号的操作，例如用管道连接的进程处在同一进程组内。进程组ID叫做PGID，进程组内的所有进程都有相同的PGID，等于该组组长的PID。
- **SID**：几个进程组可以合并成一个会话组（使用setsid系统调用），可以用于终端程序设计。会话组中所有进程都有相同的SID。

## 2、**PID 命名空间**

命名空间是为操作系统层面的虚拟化机制提供支撑，目前实现的有六种不同的命名空间，分别为mount命名空间、UTS命名空间、IPC命名空间、用户命名空间、PID命名空间、网络命名空间。命名空间简单来说提供的是对全局资源的一种抽象，将资源放到不同的容器中（不同的命名空间），各容器彼此隔离。命名空间有的还有层次关系，如PID命名空间，图1 为命名空间的层次关系图。

![img](https://pic3.zhimg.com/80/v2-9ac007352713d497921ef39e70fcfb02_720w.webp)

在上图有四个命名空间，一个父命名空间衍生了两个子命名空间，其中的一个子命名空间又衍生了一个子命名空间。以PID命名空间为例，由于各个命名空间彼此隔离，所以每个命名空间都可以有 PID 号为 1 的进程；但又由于命名空间的层次性，父命名空间是知道子命名空间的存在，因此子命名空间要映射到父命名空间中去，因此上图中 level 1 中两个子命名空间的六个进程分别映射到其父命名空间的PID 号5~10。

命名空间增大了 PID 管理的复杂性，对于某些进程可能有多个PID——在其自身命名空间的PID以及其父命名空间的PID，凡能看到该进程的命名空间都会为其分配一个PID。因此就有：

- **全局ID**：在内核本身和初始命名空间中唯一的ID，在系统启动期间开始的 init 进程即属于该初始命名空间。系统中每个进程都对应了该命名空间的一个PID，叫全局ID，保证在整个系统中唯一。
- **局部ID**：对于属于某个特定的命名空间，它在其命名空间内分配的ID为局部ID，该ID也可以出现在其他的命名空间中。

## 3、**进程ID管理数据结构**

Linux 内核在设计管理ID的数据结构时，要充分考虑以下因素：

1. 如何快速地根据进程的 task_struct、ID类型、命名空间找到局部ID
2. 如何快速地根据局部ID、命名空间、ID类型找到对应进程的 task_struct
3. 如何快速地给新进程在可见的命名空间内分配一个唯一的 PID

如果将所有因素考虑到一起，将会很复杂，下面将会由简到繁设计该结构。

### 3.1**一个PID对应一个task_struct**

如果先不考虑进程之间的关系，不考虑命名空间，仅仅是一个PID号对应一个task_struct，那么我们可以设计这样的数据结构：

```text
struct task_struct {
    //...
    struct pid_link pids;
    //...
};

struct pid_link {
    struct hlist_node node;  
    struct pid *pid;          
};

struct pid {
    struct hlist_head tasks;        //指回 pid_link 的 node
    int nr;                       //PID
    struct hlist_node pid_chain;    //pid hash 散列表结点
};
```

每个进程的 task_struct 结构体中有一个指向 pid 结构体的指针，pid 结构体包含了 PID 号。

- pid_hash[]: 这是一个hash表的结构，根据 pid 的 nr 值哈希到其某个表项，若有多个 pid 结构对应到同一个表项，这里解决冲突使用的是散列表法。这样，就能解决开始提出的第2个问题了，根据PID值怎样快速地找到task_struct结构体：

- - 首先通过 PID 计算 pid 挂接到哈希表 pid_hash[] 的表项
  - 遍历该表项，找到 pid 结构体中 nr 值与 PID 值相同的那个 pid
  - 再通过该 pid 结构体的 tasks 指针找到 node
  - 最后根据内核的 container_of 机制就能找到 task_struct 结构体



- pid_map：这是一个位图，用来唯一分配PID值的结构，图中灰色表示已经分配过的值，在新建一个进程时，只需在其中找到一个为分配过的值赋给 pid 结构体的 nr，再将pid_map 中该值设为已分配标志。这也就解决了上面的第3个问题——如何快速地分配一个全局的PID。

至于上面的第1个问题就更加简单，已知 task_struct 结构体，根据其 pid_link 的 pid 指针找到 pid 结构体，取出其 nr 即为 PID 号。

### 3.2**进程ID有类型之分**

如果考虑进程之间有复杂的关系，如线程组、进程组、会话组，这些组均有组ID，分别为 TGID、PGID、SID，所以原来的 task_struct 中pid_link 指向一个 pid 结构体需要增加几项，用来指向到其组长的 pid 结构体，相应的 struct pid 原本只需要指回其 PID 所属进程的task_struct，现在要增加几项，用来链接那些以该 pid 为组长的所有进程组内进程。数据结构如下：

```text
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};

struct task_struct {
    //...
    pid_t pid;     //PID
    pid_t tgid;    //thread group id

    struct task_struct *group_leader;   // threadgroup leader

    struct pid_link pids[PIDTYPE_MAX];

    //...
};

struct pid_link {
    struct hlist_node node;  
    struct pid *pid;          
};

struct pid {
    struct hlist_head tasks[PIDTYPE_MAX];
    int nr;                         //PID
    struct hlist_node pid_chain;    // pid hash 散列表结点
};
```

上面 ID 的类型 PIDTYPE_MAX 表示 ID 类型数目。之所以不包括线程组ID，是因为内核中已经有指向到线程组的 task_struct 指针 group_leader，线程组 ID 无非就是 group_leader 的PID。

假如现在有三个进程A、B、C为同一个进程组，进程组长为A

### 3.3**增加进程PID命名空间**

若在第二种情形下再增加PID命名空间，一个进程就可能有多个PID值了，因为在每一个可见的命名空间内都会分配一个PID，这样就需要改变 pid 的结构了，如下：

```text
struct pid
{
    unsigned int level;  
    /* lists of tasks that use this pid */
    struct hlist_head tasks[PIDTYPE_MAX];
    struct upid numbers[1];
};

struct upid {
    int nr;
    struct pid_namespace *ns;
    struct hlist_node pid_chain;
};
```

在 pid 结构体中增加了一个表示该进程所处的命名空间的层次level，以及一个可扩展的 upid 结构体。对于struct upid，表示在该命名空间所分配的进程的ID，ns指向是该ID所属的命名空间，pid_chain 表示在该命名空间的散列表。

## 4、**进程ID管理函数**

有了上面的复杂的数据结构，再加上散列表等数据结构的操作，就可以写出我们前面所提到的三个问题的函数了：

### 4.1**获得局部ID**

根据进程的 task_struct、ID类型、命名空间，可以很容易获得其在命名空间内的局部ID：

1. 获得与task_struct 关联的pid结构体。辅助函数有 task_pid、task_tgid、task_pgrp和task_session，分别用来获取不同类型的ID的pid 实例，如获取 PID 的实例：
   **static****inline****struct** pid ***task_pid**(**struct** task_struct *****task) { **return** task**->**pids[PIDTYPE_PID].pid; }
   获取线程组的ID，前面也说过，TGID不过是线程组组长的PID而已，所以：
   **static****inline****struct** pid ***task_tgid**(**struct** task_struct *****task) { **return** task**->**group_leader**->**pids[PIDTYPE_PID].pid; }
   而获得PGID和SID，首先需要找到该线程组组长的task_struct，再获得其相应的 pid：
   **static****inline****struct** pid ***task_pgrp**(**struct** task_struct *****task) { **return** task**->**group_leader**->**pids[PIDTYPE_PGID].pid; } **static****inline****struct** pid ***task_session**(**struct** task_struct *****task) { **return** task**->**group_leader**->**pids[PIDTYPE_SID].pid; }
2. 获得 pid 实例之后，再根据 pid 中的numbers 数组中 uid 信息，获得局部PID。
   **pid_t****pid_nr_ns**(**struct** pid *****pid, **struct** pid_namespace *****ns) { **struct** upid *****upid; **pid_t** nr **=** 0; **if** (pid **&&** ns**->**level **<=** pid**->**level) { upid **=****&**pid**->**numbers[ns**->**level]; **if** (upid**->**ns **==** ns) nr **=** upid**->**nr; } **return** nr; }
   这里值得注意的是，由于PID命名空间的层次性，父命名空间能看到子命名空间的内容，反之则不能，因此，函数中需要确保当前命名空间的level 小于等于产生局部PID的命名空间的level。
   除了这个函数之外，内核还封装了其他函数用来从 pid 实例获得 PID 值，如 pid_nr、pid_vnr 等。在此不介绍了。

结合这两步，内核提供了更进一步的封装，提供以下函数：

```text
pid_t task_pid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_tgid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_pigd_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_session_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
```

从函数名上就能推断函数的功能，其实不外于封装了上面的两步。

### 4.2**查找进程task_struct**

根据局部ID、以及命名空间，怎样获得进程的task_struct结构体呢？也是分两步：

1. 获得 pid 实体。根据局部PID以及命名空间计算在 pid_hash 数组中的索引，然后遍历散列表找到所要的 upid， 再根据内核的 container_of 机制找到 pid 实例。代码如下：
   **struct** pid ***find_pid_ns**(**int** nr, **struct** pid_namespace *****ns) { **struct** hlist_node *****elem; **struct** upid *****pnr; *//遍历散列表* hlist_for_each_entry_rcu(pnr, elem, **&**pid_hash[pid_hashfn(nr, ns)], pid_chain) *//pid_hashfn() 获得hash的索引***if** (pnr**->**nr **==** nr **&&** pnr**->**ns **==** ns) *//比较 nr 与 ns 是否都相同***return** container_of(pnr, **struct** pid, *//根据container_of机制取得pid 实体* numbers[ns**->**level]); **return** NULL; }
2. 根据ID类型取得task_struct 结构体。
   **struct** task_struct ***pid_task**(**struct** pid *****pid, **enum** pid_type type) { **struct** task_struct *****result **=** NULL; **if** (pid) { **struct** hlist_node *****first; first **=** rcu_dereference_check(hlist_first_rcu(**&**pid**->**tasks[type]), lockdep_tasklist_lock_is_held()); **if** (first) result **=** hlist_entry(first, **struct** task_struct, pids[(type)].node); } **return** result; }

内核还提供其它函数用来实现上面两步：

```text
struct task_struct *find_task_by_pid_ns(pid_t nr, struct pid_namespace *ns);
struct task_struct *find_task_by_vpid(pid_t vnr);
struct task_struct *find_task_by_pid(pid_t vnr);
```

具体函数实现的功能也比较简单。

### 4.3**生成唯一的PID**

内核中使用下面两个函数来实现分配和回收PID的：

```text
static int alloc_pidmap(struct pid_namespace *pid_ns);
static void free_pidmap(struct upid *upid);
```

在这里我们不关注这两个函数的实现，反而应该关注分配的 PID 如何在多个命名空间中可见，这样需要在每个命名空间生成一个局部ID，函数 alloc_pid 为新建的进程分配PID，简化版如下：

```text
struct pid *alloc_pid(struct pid_namespace *ns)
{
    struct pid *pid;
    enum pid_type type;
    int i, nr;
    struct pid_namespace *tmp;
    struct upid *upid;

    tmp = ns;
    pid->level = ns->level;
    // 初始化 pid->numbers[] 结构体
    for (i = ns->level; i >= 0; i--) {
        nr = alloc_pidmap(tmp);            //分配一个局部ID
        pid->numbers[i].nr = nr;
        pid->numbers[i].ns = tmp;
        tmp = tmp->parent;
    }

    // 初始化 pid->task[] 结构体
    for (type = 0; type < PIDTYPE_MAX; ++type)
        INIT_HLIST_HEAD(&pid->tasks[type]);

    // 将每个命名空间经过哈希之后加入到散列表中
    upid = pid->numbers + ns->level;
    for ( ; upid >= pid->numbers; --upid) {
        hlist_add_head_rcu(&upid->pid_chain,
                &pid_hash[pid_hashfn(upid->nr, upid->ns)]);
        upid->ns->nr_hashed++;
    }

    return pid;
}
```

----

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/546814252