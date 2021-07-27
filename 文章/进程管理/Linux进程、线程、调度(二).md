### fork 、vfork、clone

![image](https://user-images.githubusercontent.com/87457873/127104795-7ec16416-cbee-4022-a36d-84de40a5e229.png)

Linux 内核的调度算法，是根据task_struct结构体来进行调度的。

### 写时拷贝技术

当p1把p2创建出来时，会把task_struct里描述的资源结构体对拷给p2。<br>
区分进程的标志，就是 p2的资源不是p1的资源。两个task_struct 的资源都相同，那就不叫两个进程了。

执行一个copy，但是任何修改都造成分裂，如：chroot,open，写memory，<br>
最难copy的是 mm 这个部分，因为要做写时拷贝。

Linux通过MMU进行虚拟地址到物理地址的转换，当进程执行fork()后，会把页表中的权限设置为RD-ONLY，当P1,P2去写该页时，CPU会收到page fault，申请新的内存。Linux再将页表中的virt1指向新的物理地址。

![image](https://user-images.githubusercontent.com/87457873/127104867-e304952e-ff6c-4094-8bd0-f2992277ed06.png)

```c
#include <sched.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
int data = 10;
int child_process()
{
    printf("Child process %d, data %d\n",getpid(),data);
    data = 20;
    printf("Child process %d, data %d\n",getpid(),data);
    _exit(0);
}

int main(int argc, char* argv[])
{
    int pid;
    pid = fork();
    if(pid==0) {
        child_process();
    }
    else{
        sleep(1);
        printf("Parent process %d, data %d\n",getpid(), data);
        exit(0);
    }
}
```

CoW，严重依赖CPU的MMU。Mmu-less Linux 无copy-on-write, 没有fork，而使用vfork。

使用vfork：父进程p1 vfork出子进程p2之后阻塞，直到子进程发生以下两种情况。<br>
1） exit<br>
2） exec<br>

![image](https://user-images.githubusercontent.com/87457873/127104939-ee0a4312-0921-4c47-b047-5ec659634140.png)

进程执行vfork时，P2的task_struct中的*mm 与 P1共享，P1的内存资源就是P2的内存资源。

pthread_create -> clone

![image](https://user-images.githubusercontent.com/87457873/127104959-d6f13b93-095f-4bda-b5dc-7c239ae69e34.png)

Linux创建线程的API，本质上去调 clone。要求把P2的所有资源的指针，都指向P1。线程，也被称为 Light weight process。

而Linux在clone线程时也十分灵活，可以选择共享/不共享部分资源。

![image](https://user-images.githubusercontent.com/87457873/127104969-6a37ba07-0291-4e63-9503-324d37cf00ff.png)

POSIX标准要求，进程里面如果有多个线程，在用户空间 getpid() 看到的都是同一个id，这个id其实是TGID。

一个进程里面创建了多个线程，在/proc 下 的是 tgid，/proc/tgid/task/{pidx,y,z}

pthread_self() 看到的是用户空间pthread线程库里获得的id 。

```c
#include <stdio.h>
#include <pthread.h>
#include <stdio.h>
#include <linux/unistd.h>
#include <sys/syscall.h>

static pid_t gettid( void )
{
    return syscall(__NR_gettid);
}

static void *thread_fun(void *param)
{
    printf("thread pid:%d, tid:%d pthread_self:%lu\n", getpid(), gettid(),pthread_self());
    while(1);
    return NULL;
}

int main(void)
{
    pthread_t tid1, tid2;
    int ret;
    printf("thread pid:%d, tid:%d pthread_self:%lu\n", getpid(), gettid(),pthread_self());
    ret = pthread_create(&tid1, NULL, thread_fun, NULL);
    
    if (ret == -1) {
        perror("cannot create new thread");
        return -1;
    }
    ret = pthread_create(&tid2, NULL, thread_fun, NULL);
    
    if (ret == -1) {
        perror("cannot create new thread");
        return -1;
    }
    if (pthread_join(tid1, NULL) != 0) {
        perror("call pthread_join function fail");
        return -1;
    }
    if (pthread_join(tid2, NULL) != 0) {
        perror("call pthread_join function fail");
        return -1;
    }
    return 0;
}
```

```
gcc thread.c -pthread
```
### 总结 fork , vfork, clone

由于执行fork()引入了 写时拷贝并且明确了子进程先执行，所以 vfork()的好处就仅限于不拷贝父进程的页表项mm_struct。vfork()系统调用的实现是通过向clone()系统调用传递一个特殊标志来进行。

vfork场景下父进程会先休眠，等唤醒子进程后，再唤醒父进程。

这么做的好处是：由于子进程被创建出来，与父进程共享地址空间，且只读。只有在执行exec的创建新的内存映射时才会拷贝父进程的数据，来创建新的地址空间。如果此时，父进程还在执行，就有可能产生脏数据，或发生死锁。

### 进程0和进程1
init进程是被Linux 0进程创建，0进程把init进程fork出来后，就退化成IDLE进程。这个进程，是特殊调度类，所有进程都停止或睡眠后，就会调度进程0运行，此时处于CPU低功耗状态。

### 孤儿进程与托孤，subreaper

![image](https://user-images.githubusercontent.com/87457873/127105118-3de38b9e-1d18-46b9-b189-afe098ec47da.png)

当父进程退出后，子进程会寻找subreaper 或 init进程。

