## 进程生命周期

在Linux内核里，无论是进程还是线程，统一使用 task_struct{} 结构体来表示，也就是统一抽象为任务（task）。task_struct{} 定义在 include/linux/sched.h 文件中，十分复杂，这里简单了解下。

```cpp
// include/linux/sched.h
// ... 省略

struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
	/* -1 unrunnable, 0 runnable, >0 stopped: */
	volatile long			state;
	int				exit_state;
	int				exit_code;
	int				exit_signal;

	/*
	 * This begins the randomizable portion of task_struct. Only
	 * scheduling-critical items should be added above here.
	 */
	randomized_struct_fields_start

	void				*stack;
	refcount_t			usage;
	/* Per task flags (PF_*), defined further below: */
	unsigned int			flags;
	unsigned int			ptrace;

#ifdef CONFIG_SMP
	int				on_cpu;
	struct __call_single_node	wake_entry;
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/* Current CPU: */
	unsigned int			cpu;
#endif
	unsigned int			wakee_flips;
	unsigned long			wakee_flip_decay_ts;
	struct task_struct		*last_wakee;
  // ...省略
  struct sched_info		sched_info;
	struct list_head		tasks;   // 链表，将所有task_struct串起来

	pid_t				pid;    // process id，指的是线程id
	pid_t				tgid;   // thread group ID，指的是进程的主线程id
  struct task_struct		*group_leader;  // 指向的是进程的主线程

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

  // ... 省略
}
```

**查阅相关资料后，对Linux中进程的生命周期总结如下：**

![img](https://pic1.zhimg.com/80/v2-e5a63b6d74aaca755243cda0f15a75fc_720w.webp)

从图上可以看出，进程的睡眠状态是最多的，那进程一般在何时进入睡眠状态呢？答案是I/O操作时，因为I/O操作的速度与CPU运行速度相比，相差太大，所以此时进程会释放CPU，进入睡眠状态。

![img](https://pic1.zhimg.com/80/v2-0fbc601c3e6455cf70b41588ea249f44_720w.webp)



进程状态相关的定义同样在 include/linux/sched.h 文件的开头部分，以 #define TASK_KILLABLE (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE) 为例，TASK_WAKEKILL表示用于在接收到致命信号时唤醒进程，将它与 TASK_UNINTERRUPTIBLE 按位或，就得到了 TASK_KILLABLE。代码注释中提到了 fs/proc/array.c，所以也将其代码贴出，作为补充。

```cpp
// include/linux/sched.h
// ... 省略

/*
 * Task state bitmask. NOTE! These bits are also
 * encoded in fs/proc/array.c: get_task_state().
 *
 * We have two separate sets of flags: task->state
 * is about runnability, while task->exit_state are
 * about the task exiting. Confusing, but this way
 * modifying one set can't modify the other one by
 * mistake.
 */

/* Used in tsk->state: */
#define TASK_RUNNING			0x0000
#define TASK_INTERRUPTIBLE		0x0001
#define TASK_UNINTERRUPTIBLE		0x0002
#define __TASK_STOPPED			0x0004
#define __TASK_TRACED			0x0008
/* Used in tsk->exit_state: */
#define EXIT_DEAD			0x0010
#define EXIT_ZOMBIE			0x0020
#define EXIT_TRACE			(EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_PARKED			0x0040
#define TASK_DEAD			0x0080
#define TASK_WAKEKILL			0x0100
#define TASK_WAKING			0x0200
#define TASK_NOLOAD			0x0400
#define TASK_NEW			0x0800
#define TASK_STATE_MAX			0x1000

/* Convenience macros for the sake of set_current_state: */
#define TASK_KILLABLE			(TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED			(TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED			(TASK_WAKEKILL | __TASK_TRACED)

#define TASK_IDLE			(TASK_UNINTERRUPTIBLE | TASK_NOLOAD)

/* Convenience macros for the sake of wake_up(): */
#define TASK_NORMAL			(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)

/* get_task_state(): */
#define TASK_REPORT			(TASK_RUNNING | TASK_INTERRUPTIBLE | \
					 TASK_UNINTERRUPTIBLE | __TASK_STOPPED | \
					 __TASK_TRACED | EXIT_DEAD | EXIT_ZOMBIE | \
					 TASK_PARKED)

#define task_is_traced(task)		((task->state & __TASK_TRACED) != 0)

#define task_is_stopped(task)		((task->state & __TASK_STOPPED) != 0)

#define task_is_stopped_or_traced(task)	((task->state & (__TASK_STOPPED | __TASK_TRACED)) != 0)

// ... 省略
// fs/proc/array.c
// ... 省略

/*
 * The task state array is a strange "bitmap" of
 * reasons to sleep. Thus "running" is zero, and
 * you can test for combinations of others with
 * simple bit tests.
 */
static const char * const task_state_array[] = {

	/* states in TASK_REPORT: */
	"R (running)",		/* 0x00 */
	"S (sleeping)",		/* 0x01 */
	"D (disk sleep)",	/* 0x02 */
	"T (stopped)",		/* 0x04 */
	"t (tracing stop)",	/* 0x08 */
	"X (dead)",		/* 0x10 */
	"Z (zombie)",		/* 0x20 */
	"P (parked)",		/* 0x40 */

	/* states beyond TASK_REPORT: */
	"I (idle)",		/* 0x80 */
};

static inline const char *get_task_state(struct task_struct *tsk)
{
	BUILD_BUG_ON(1 + ilog2(TASK_REPORT_MAX) != ARRAY_SIZE(task_state_array));
	return task_state_array[task_state_index(tsk)];
}

// ... 省略
```

在单核的CPU上，同一时刻只有一个task会被调度，所以即使看到了 R 状态，也不代表进程就被分配到了CPU时间片。但了解了进程状态之后，我们再通过 top, ps aux 等命令查看进程，**分析问题起来效率就更高了:**

```cpp
top - 17:24:07 up 10:20,  1 user,  load average: 0.15, 0.08, 0.02
Tasks: 216 total,   1 running, 215 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.4 us,  0.3 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   5945.2 total,   2655.4 free,   1580.8 used,   1709.1 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   4084.6 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                            
   1914 demonlee  20   0 5252100 395940 132816 S   1.0   6.5   1:33.09 gnome-shell                                                                        
    824 root      20   0 2495652  89512  48936 S   0.7   1.5   0:05.59 dockerd                                                                            
   1687 demonlee  20   0 1153156  81436  50128 S   0.3   1.3   0:15.90 Xorg                                                                               
   1957 demonlee  20   0  206556  28348  18504 S   0.3   0.5   0:00.26 ibus-x11                                                                           
   2897 demonlee  20   0  874684  61296  44532 S   0.3   1.0   0:06.22 gnome-terminal-                                                                    
  19984 demonlee  20   0   20632   4036   3376 R   0.3   0.1   0:00.02 top                                                                                
      1 root      20   0  169076  12952   8288 S   0.0   0.2   0:04.61 systemd                                                                            
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.01 kthreadd                                                                           
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp                                                                             
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par_gp                                                                         
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/0:0H-kblockd                                                               
      9 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_percpu_wq                                                                       
     10 root      20   0       0      0      0 S   0.0   0.0   0:00.08 ksoftirqd/0                                                                        
     11 root      20   0       0      0      0 I   0.0   0.0   0:03.60 rcu_sched                                                                          
     12 root      rt   0       0      0      0 S   0.0   0.0   0:00.51 migration/0                                                                        
     13 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_inject/0                                                                      
     14 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/0                                                                            
     15 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/1                                                                            
     16 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_inject/1                                                                      
     17 root      rt   0       0      0      0 S   0.0   0.0   0:00.89 migration/1                                                                        
     18 root      20   0       0      0      0 S   0.0   0.0   0:00.09 ksoftirqd/1                                                                        
     20 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/1:0H                                                                       
     21 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/2                                                                            
     22 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_inject/2                                                                      
     23 root      rt   0       0      0      0 S   0.0   0.0   0:00.83 migration/2                                                                        
     24 root      20   0       0      0      0 S   0.0   0.0   0:00.07 ksoftirqd/2                                                                        
     26 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/2:0H-kblockd                                                               
     27 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/3                                                                            
     28 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_inject/3                                                                      
     29 root      rt   0       0      0      0 S   0.0   0.0   0:00.77 migration/3                                                                        
     30 root      20   0       0      0      0 S   0.0   0.0   0:00.18 ksoftirqd/3                                                                        
     32 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/3:0H-kblockd                                                               
     33 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kdevtmpfs                                                                          
     34 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 netns                                                                              
     35 root      20   0       0      0      0 S   0.0   0.0   0:00.00 rcu_tasks_kthre                                                                    
     36 root      20   0       0      0      0 S   0.0   0.0   0:00.00 rcu_tasks_rude_                                                                    
     37 root      20   0       0      0      0 S   0.0   0.0   0:00.00 rcu_tasks_trace                                                                    
     38 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kauditd                                                                            
     39 root      20   0       0      0      0 S   0.0   0.0   0:00.03 khungtaskd                                                                         
demonlee@demonlee-ubuntu:~$
```

最后，再补充一个知识点：使用ps命令查看进程时，会发现状态上面有其他符号，比如 S+、Z+ 等，如下所示，

```cpp
demonlee@demonlee-ubuntu:~$ 
demonlee    1704  0.0  0.6 557904 37256 ?        Sl   05:01   0:00 /usr/libexec/goa-daemon
demonlee    1707  0.0  0.1 172652  6936 tty2     Ssl+ 05:01   0:00 /usr/lib/gdm3/gdm-x-session --run-script env GNOME_SHELL_SESSION_MODE=ubuntu /usr/bin/g
demonlee    1714  0.0  0.1 323388  9068 ?        Sl   05:01   0:00 /usr/libexec/goa-identity-service
demonlee    1720  0.1  1.3 1151880 80576 tty2    Sl+  05:01   1:05 /usr/lib/xorg/Xorg vt2 -displayfd 3 -auth /run/user/1000/gdm/Xauthority -background non
demonlee    1723  0.0  0.1 325356  9016 ?        Ssl  05:01   0:02 /usr/libexec/gvfs-afc-volume-monitor
demonlee    1728  0.0  0.1 244336  6532 ?        Ssl  05:01   0:00 /usr/libexec/gvfs-mtp-volume-monitor
demonlee    1759  0.0  0.2 197052 14276 tty2     Sl+  05:01   0:00 /usr/libexec/gnome-session-binary --systemd --systemd --session=ubuntu
...
```

这个 + 是啥意思呢，其实manps中就有描述，只是我们从来都没认真看说明文档：

```cpp
PROCESS STATE CODES
       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a
       process:

               D    uninterruptible sleep (usually IO)
               I    Idle kernel thread
               R    running or runnable (on run queue)
               S    interruptible sleep (waiting for an event to complete)
               T    stopped by job control signal
               t    stopped by debugger during the tracing
               W    paging (not valid since the 2.6.xx kernel)
               X    dead (should never be seen)
               Z    defunct ("zombie") process, terminated but not reaped by its parent

       For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users)
               N    low-priority (nice to other users)
               L    has pages locked into memory (for real-time and custom IO)
               s    is a session leader
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
               +    is in the foreground process group
```

-----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/442905446