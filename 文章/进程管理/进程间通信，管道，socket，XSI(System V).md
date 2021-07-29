进程通信(IPC)分为PIPE(管道)、Socket(套接字)和XSI(System_V)。XSI又分为msg(消息队列)、sem(信号量数组)和shm(共享内存)。这些手段都是用于进程间通信的，只有进程间通讯才需要借助第三方机制，线程之间通讯是不需要借助第三方机制，因为线程之间的地址空间是共享的。线程之间可以通过互斥量，死锁，唤醒，信号等来进行通讯。

## 管道(PIPE->FIFO) 内核帮你创建和维护

### 管道的特点:

* 管道是半双工的，也就是同一时间数据只能从一端流向另一段。就像水一样，两端水同时流入管道，那么数据就会乱
* 管道的两端一端作为读端，一端是写端
* 管道具有自适应的特点， 默认会适应速度比较慢的一方，管道被写满或读空时速度快的一方会自动阻塞

```c
pipe - create pipe
#include <unistd.h>
int pipe(int pipefd[2]);
// 也就只有两端，一端读，一端写
```

pipe用于创建管道，pipefd是一个数组，表示管道的两端文件描述符，pipefd[0]端作为读端，pipefd[1]作为写端。

pipe产生的是匿名管道，在磁盘的任何位置上找不到这个管道文件，而且匿名管道只能用于具有亲缘关系的进程之间通信(还要分亲缘关系)

一般情况下有亲缘关系的进程之间使用管道进行通信时，会把自己不用的一端文件描述符关闭

```c
#include "../include/apue.h"

#define BUFSIZE 1024

int main(){
    int pd[2];
    char buf[BUFSIZE];
    pid_t pid;
    int len;
    // 创建匿名管道
    if(pipe(pd)<0)
        err_sys("pipe()");

    pid = fork();
    if(pid == 0){ // 子进程 读取管道数据
        // 关闭写端
        close(pd[1]);
        // 从管道中读取数据，如果子进程比父进程先被调度会阻塞等待数据写入
        len = read(pd[0],buf,BUFSIZE);
        puts(buf);
        /***
         * 管道fork之前创建
         * 父子进程都有一份
         * 所有退出之前要确保管道两端都关闭
         * ***/
        close(pd[0]);
        exit(0);
    }else{ // 父进程 向管道写入数据
        // 关闭读端
        close(pd[0]);
        // 写端
        write(pd[1],"Hello,world!",100);
        close(pd[1]);
        wait(NULL);
        exit(0);
    }
}
```

创建了一个匿名的管道，在pd[2]数组中凑齐了读写双方，子进程同样继承了具有读写双方的数组pd[2]

当关闭之后就是取决于我们需要对管道的数据流方向做准备。要么从子进程流向父进程，要么从父进程流向子进程。

![image](https://user-images.githubusercontent.com/87457873/127452218-fee35e85-718a-4303-92bc-965065e3eac9.png)

mkfifo函数

```c
mkfifo - make a FIFO special file (a named pipe)
#include  <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname,mode_t mode);
// pathname: 管道文件的路径和文件名
// mode: 创建管道文件的权限。还是老规矩，传入的mode值要与系统的umask值做运算(mode&~umask)
// 成功返回0，失败返回-1并设置errno
```

mkfifo函数用于创建命名管道，作用与匿名管道相同，不过可以在不同的进程之间使用，相当于对一个普通文件进行写操作就可以了。

这个管道文件是任何有权限的进程都可以使用的，两端都像操作一个普通文件一样对它进行打开、读写、关闭动作就可以了，只要一端写入数据另一端就可以读出来。

命名管道文件

```c
#include "../include/apue.h"
#include <fcntl.h>

#define PATHNAME "./mkfifof.txt"

int main(void){
    pid_t pid;
    int fd = -1;
    char buf[BUFSIZ] = "";

    // 创建一个命名管道，通过ls -l查看这个管道的属性
    if(mkfifo(PATHNAME,0664)<0){
        err_sys("mkfifo");
    }

    fflush(NULL); // 刷新缓冲区
    pid = fork();

    if(pid<0)
        err_sys("fork()");

    if(!pid){
        pid = fork(); // 继续fork 三个进程了
        if(pid<0)err_sys("fork()2");
        if(!pid) exit(0); // 爸爸走了
       // child2
        fd = open(PATHNAME,O_RDWR);
        if(fd<0) err_sys("open()");
        // 阻塞，等待条件满足
        read(fd,buf,BUFSIZ);
        printf("%s\n",buf);
        write(fd," World!",8);
        close(fd);
        exit(0);
    }else{
        fd = open(PATHNAME,O_RDWR);
        if(fd < 0) err_sys("open()");
        // 写
        write(fd,"hello",6);
        sleep(1); // 要是不休眠就没有给另一个进程机会写，最后自娱自乐，第二个进程也打不开文件
        read(fd,buf,BUFSIZ);
        close(fd);
        puts(buf);
        // 这个进程最后退出，所以把管道文件删除，不然下次在创建的时候会报文件已存在的错误
        remove(PATHNAME);
        exit(0);
    }
    return 0;
}
```

看到了下面的创建，不是普通的文件，而是管道文件

```c
prw-r--r--  1 transcheung  staff      0  2 14 11:41 mkfifof.txt
```

### 协同进程 管道是半双工的 两进程一个只能读，一个只能写

要实现双工通信，必须采用两个管道，一个进程对一个管道只读，对另一个管道只写。

```c
#include "../include/apue.h"

#define BUFSIZE 1024

int main(){
    int pd[2];
    int ipd[2];
    char buf[BUFSIZE];
    char dbuf[BUFSIZE];
    pid_t pid;
    int len;
    // 创建匿名管道
    if(pipe(pd)<0)
        err_sys("pipe()");
    if(pipe(ipd)<0) err_sys("pipe2");
    pid = fork();
    if(pid == 0){ // 子进程 读取管道数据
        // 关闭写端
       // close(pd[1]);
        // 从管道中读取数据，如果子进程比父进程先被调度会阻塞等待数据写入
        //len = read(pd[0],buf,BUFSIZE);
       // puts(buf);
        /***
         * 管道fork之前创建
         * 父子进程都有一份
         * 所有退出之前要确保管道两端都关闭
         * ***/
        close(pd[0]);
        write(pd[1],"hello,child!",15);
        // sleep(10)；
        //sleep(10);
        len = read(ipd[0],dbuf,BUFSIZE);
        puts(dbuf);
        close(pd[1]);
        close(ipd[0]);
        exit(0);
    }else{ // 父进程 向管道写入数据
        // 关闭读端
        //close(pd[0]);
        // 写端
       // write(pd[1],"Hello,world!",15);
        close(pd[1]);
        len = read(pd[0],buf,BUFSIZE);
        puts(buf);
        // 关闭读端
        // sleep(5);
        write(ipd[1],"hello parent!",BUFSIZE);
        close(pd[0]);
        close(ipd[1]);
        wait(NULL);
        exit(0);
    }
}
```

两个管道，实现你来我往，mkfifo同理，两个文件就好了。

## popen与pclose

popen和pclose提供了原子操作，创建一个管道，fork一个子进程，关闭未使用的管道端，执行一个个shell运行命令，然后等待命令终止

```c
#include <stdio.h>
FILE *popen(const char *cmdstring,const char *type);
// 成功返回文件指针，出错，返回NULL
int pclose(FILE *fp);
```

popen先执行fork，然后调用exec执行cmdstring，并且返回一个标准IO文件指针。如果type是"r"，则文件指针连接到cmdstring的标准输出；如果是"w"，则文件指针连接到cmfstring的标准输入

![image](https://user-images.githubusercontent.com/87457873/127452540-8c4dd73b-68b1-4357-a4c0-015667790601.png)

由图可以看出，stdout和stdin是较于子进程而言的。

pclose关闭标准IO流，等待命令终止，返回shell的终止状态。如果shell不能被执行，则pclose返回的终止状态与shell已执行exit一样。

    cmdstring
    fp = popen("ls *.c","r")
    fp = popen("cmd 2>&1","r") 

## XSI IPC System V规范的进程间通信手段，而不是POSIX标准

## 多进程与多线程

多线程使用的基本是POSIZ标准提供的接口函数，而多进程则是基于System V。在信号量这种常用的同步互斥手段方面，POSIX在无竞争条件下是不会陷入内核的，而SystemV即无论何时都会陷入内核。这就给多线程每次调用都会陷入内核，丧失了线程的清亮优势。所以多线程之间的通信不是用System V。

* ipcs命令可以查看CSI IPC的使用情况
* ipcrm 命令可以删除指定的XSI IPC

通过查看ipcs

```
IPC status from <running system> as of Fri Feb 15 10:31:54 CST 2019T     ID     KEY        MODE       OWNER    GROUP
Message Queues:

T     ID     KEY        MODE       OWNER    GROUP
Shared Memory:

T     ID     KEY        MODE       OWNER    GROUP
Semaphores:
```

第一部分是消息队列，第二部分是共享内存，第三部分是信号量数组

每一列都有一列叫做"key"，使用XSI IPC通信的进程就是通过同一个key值操作同一个共享资源的。key是一个正整数，与文件描述符不同的是，生成一个新key值时，不采用当前可用的数值的最小值，而是类似生成进程ID的方式，key的值连续➕1，直到达到一个整数的最大正值，然后回转到0从头开始累加。

## XSI消息队列 让通信双方传送结构体数据，提高传送数据灵活性

通信，就需要在通信之前双方约定通信协议，协议就是通信双方约定的数据交换格式。

从消息队列开始一直到Socket，都会看到类似的程序架构，无论是消息队列还是Socket，都需要约定通信协议，而且都是按照一定的步骤才能实现通讯。

消息队列在约定协议的时候，需要自定义结构体里要强制添加一个long mtype成员 这个成员的作用是用于区分多种消息类型中的不同类型的数据包，当只有一种类型的包时这个成员没什么作用，但是也要一定必须带上。

既然时通讯也要区分 主动端(先发包的一方) 和 被动端(先收包的一方，先运行)，它们运行的时机不同，作用不同甚至调用函数也不同，所以我们的后面的每个例子几乎都要编译处2个不同的可执行程序来测试。

## msg、sem和shm都有一系列函数遵循

* xxxget() // 创建
* xxxop() // 相关操作
* xxxctl() // 其他的控制或销毁

```c
msgget - get a Syatem V message queue identifier

#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgget(key_t key,int msgflg);
```

msgget函数的作用是创建一个消息队列，消息读列是双工的，两边都可以读写。

* key相当于通信双方的街头暗号，拥有相同key的双方才可以通信

key值必须是唯一的，系统中有个ftok函数可以用于获取key，通过文件inode和salt进行hash运算来生成唯一的key， 只要两个进程使用相同的文件和salt就可以生成一样的key值了。

* msgflg: 特殊要求。无论有多少特殊要求，只要使用了IPC_CREAT，就必须按位或一个权限，权限不是想指定多大就能多大，要用它&=~umask。

同一个消息队列只需要创建一次，所以谁先运行起来谁有责任创建消息队列，后运行起来的就不需要创建了。

同理，对于后启动的进程来说，消息队列不是他创建的，就没必要销毁了。

## msgrcv函数和msgsnd函数 从msgid这个消息队列中接收数据

```c
msgrcv,msgsnd - message operations
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgsnd(int msgid, const void *msgp,size_t msgsz,int msgflg);

ssize_t msgrcv(int msgid,void *msgp,size_t msgsz,long msgtyp,int msgflg);
// msgp成员的定义要类似msgbuf这个结构体，第一成员必须是long类型的mtype，并且必须是>0的值
struct msgbuf{
    long mtype; // 消息类型 必须>0
    char mtext[1]; // 消息数据字段
};
```

* msgrcv函数从msgid这个消息队列中接收数据，并将接收到的数据放到msgp结构体中，这段空间有msgsz这个字节的大小，msgsz的值要减掉强制的成员mtype的大小(sizeof(long))。
* msgtyp是msgp结构体中的mtype的成员，表示需要接收那种类型的消息。虽然msg是消息队列，但是 它并不完全遵循队列的形式，可以让接收者挑消息接收。 如果不挑消息可以填写0，这样就按照队列中的消息顺序返回。
* msgflg是特殊要求位图，没有写0
* msgsnd函数向msgid这个消息队列发送msgp结构体数据，msgp的大小是msgsz，msgflg是特殊要求没有写0。

## msgctl函数 跟iocrtl、fcntl函数用法类似

```c
msgctl - message control operations
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgctl(int msgid,int cmd,struct msqid_ds *buf);
// 通过cmd指定具体命令，然后通过buf为cmd命令设定参数，当然有些命令是需要参数的，有些命令则不需要参数
```

最常用的cmd就是IPC_RMID，表示删除(结束)某个IPC通信，并且这个命令不需要buf参数，直接传入NULL即可。

buf结构体里面的成员很多。

```c
/***
 * 共同的协议proto.h
 * ***/
/*定义双方都需要使用的数据或对象*/
#ifndef PROTO_H__
#define PROTO_H__
#define NAMESIZE 32
// 通讯双方生成key值共同使用的文件
#define KEYPATH "./test.txt"
// 通讯双方生成key值共同使用的salt值
#define KEYPROJ 'a'

// 消息类型，只要是大于0的合法整数即可
#define MSGTYPE 10

// 通讯双方约定的协议
struct msg_st{
    long mtype;
    char name[NAMESIZE];
    int math;
    int chinese;
};
#endif //  PROTO_H__
```

接收端要先运行，先创建接收端的消息队列

```c
#include "../include/apue.h"
#include <sys/ipc.h>
#include <sys/msg.h>

#include "proto.h"

int main(void){
    key_t key;
    int msgid;
    struct msg_st rbuf;

    // 通过/tmp/out文件和字符'a'生成唯一的key，文件必须真实存在
    key = ftok(KEYPATH,KEYPROJ); // 来自proto.h
    if(key<0) err_sys("ftok()");
    
    // 接收端先启动，所以消息队列由接收端创建
    if((msgid = msgget(key,IPC_CREAT|0600))<0) err_sys("msgget");
    
    // 不停的接收消息 轮询
    while(1){
        // 没有消息就会阻塞等待
        if(msgrcv(msgid,&rbuf,sizeof(rbuf)-sizeof(long),0,0)<0) err_sys("msgrcv");

        /***
         * 用结构体中强制添加的成员判断消息类型
         * 当然这个栗子只有一种消息类型，所以不判断也可以
         * 如果包含多种消息类型就可以协议组switch ...case结构
         * ***/
        if(rbuf.mtype == MSGTYPE){
            printf("Name = %s\n",rbuf.name);
            printf("Math = %d\n",rbuf.math);
            printf("Chinese = %d\n",rbuf.chinese);

        }
    }
    /***
     * 谁创建谁销毁
     * 这个程序无法正常结束只能等信号杀死
     * 使用信号杀死之后可以用ipcs命令查看，消息队列应该没有被销毁
     * 使用ipcrm删掉
     * ***/
    msgctl(msgid,IPC_RMID,NULL); // 销毁
    exit(0);
}
```

发送端

```c
#include "../include/apue.h"
#include <sys/ipc.h>
#include <sys/msg.h>
#include <string.h>
#include <time.h>

#include "proto.h"

int main(){
    key_t key;
    int msgid;
    struct msg_st sbuf;

    // 设置随机数种子
    srand(time(NULL));
    // 用于接收方相同的文件和salt生成一样的key
    key = ftok(KEYPATH,KEYPROJ);
    if(key<0) err_sys("ftok()");
    // 取得消息队列
    msgid = msgget(key,0);
    if(msgid<0) err_sys("msgget");

    // 要发送的结构体赋值
    sbuf.mtype = MSGTYPE;
    strcpy(sbuf.name,"Trans");
    sbuf.math = rand()%100;
    sbuf.chinese = rand()%100;

    if(msgsnd(msgid,&sbuf,sizeof(sbuf)-sizeof(long),0)<0) err_sys("msgsnd");
    
    puts("ok!");
    exit(0);
}
```

最后使用ipcs查看，消息队列显式

```
IPC status from <running system> as of Fri Feb 15 12:21:02 CST 2019
T     ID     KEY        MODE       OWNER    GROUP
Message Queues:
q  65536 0x6104a0ae --rw------- transcheung    staff

T     ID     KEY        MODE       OWNER    GROUP
Shared Memory:

T     ID     KEY        MODE       OWNER    GROUP
Semaphores:
```

KEYPROJ直接充当salt值，接收方先运行，所以接收方先创建消息队列，发送方要使用相同的文件和salt生成于接收方相同的key值，这样才能使用同一个消息队列。

发送方发送一个结构体，接收方接收结构体并解析打印，所以这个结构体保证了数据能够正常被解析，所以 这个结构体就是我们所说的"协议"。 所以协议就是要保证一样的，所以写了一个proto.h文件，让发送方共同引用，就保证是相同的结构体了。

## 信号量(semget) 按部就班，步骤与消息队列差不多

```c
semget - get a semaphore set identifier
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semget(key_t key,int nsems,int semflg);
// 用于创建信号量，成功返回sem ID，失败返回-1并设置errnp
```

* key: 具有亲缘关系的进程之间可以使用一个匿名的key值，key使用宏IPC_PRIVATE即可
* nsems:表示你到底有多少个sem。信号量其实是一个计数器，如果设置为1可以用来模拟互斥量
* semflg:IPC_CREAT表示创建sem，同时需要按位或一个权限，如果是匿名IPC则无需执行这个宏，直接给权限就好

```c
semctl - semaphore control operations

#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semctl(int semid, int semnum, int cmd, ...);
// 用来控制或销毁信号量
```

* semnum:信号量数组下标
* cmd: 可选的宏。常用的由IPC_RMID，表示从系统中删除该信号量集合，SETVAL可以为第几个成员设置值。
* ...: 根据不同命令设置不同的参数，后面的参数是变长的

```c
semop - semaphore operations

#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semop(int semid, struct sembuf *sops, unsigned nsops);
// 操作信号量。由于多个信号量可以组成数组。sops参数是数组的起始位置，nspos是指定数组的长度
// 成功返回0，失败返回-1并设置errno
struct sembuf {
    unsigned short sem_num; /* 对第几个资源（数组下标）操作 */
    short sem_op; /* 取几个资源写负数几(不要写减等于)，归还几个资源就写正数几 */
    short sem_flg; /* 特殊要求 */
};
```

信号量实际上是一个计数器，所以每次在使用资源之前，我们需要扣减信号量，当信号量被减到0时会阻塞等待。每次使用完成资源后，归还信号量，也就是增加信号量的数值

通过操作信号量的函数实现一个通过信号量实现互斥量的例子

```c
#include "../include/apue.h"
#include <string.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <errno.h>

#define PROCNUM 20
#define FNAME "./test.txt"
#define BUFSIZE 1024

// 多个函数都要使用这个信号量ID，所以定义为全局变量
static int semid;

// P操作
static void P(void){
    struct sembuf op;
    op.sem_num = 0; // 只有一个资源所以数组下标是0
    op.sem_op = -1; // 取一个资源就减1
    op.sem_flg = 0; // 没有特殊要求
    while(semop(semid,&op,1)<0){
        if(errno != EINTR && errno!=EAGAIN) err_sys("semop()");
    }
}
// V操作
static void V(void){
    struct sembuf op;
    op.sem_num = 0;
    op.sem_op = 1;
    op.sem_flg = 0;
    while(semop(semid,&op,1)<0){
        if(errno != EINTR && errno != EAGAIN) err_sys("semop()");
    }
}

static void func_add(){
    FILE *fp;
    char buf[BUFSIZE];
    fp = fopen(FNAME,"r+");
    if(fp == NULL) err_sys("fopen()");
    // 先取得信号量再操作文件，取不到就阻塞等待，避免发生竞争
    P();
    fgets(buf,BUFSIZE,fp); // 获取一行字符串
    rewind(fp); // 从头指针开始
    sleep(1); // 放大竞争
    fprintf(fp,"%d\n",atoi(buf)+1);
    fflush(fp);
    // 操作结束，归还信号量，让其他进程可以取得信号量
    V();
    fclose(fp);
    return;
}

int main(){
    int i;
    pid_t pid;

    // 在具有亲缘关系的进程之间使用，所以设置为IPC_PRIVATE
    // 另外想要实现互斥量的效果，所以信号量数量设置为1个即可
    semid = semget(IPC_PRIVATE,1,0600);
    if(semid<0) err_sys("semget()");
    // 将union semun.val的值设置为1
    if(semctl(semid,0,SETVAL,1)<0) err_sys("semctl()");

    // 创建20个子进程
    for(i = 0;i<PROCNUM;i++){
        pid = fork();
        if(pid<0) err_sys("fork()");
        if(pid == 0){
            func_add();
            exit(0);
        }
    }

    for(i = 0;i<PROCNUM;i++) wait(NULL);

    // 销毁信号量
    semctl(semid,0,IPC_RMID);
    exit(0);
}
```

## 共享存储 shmget

XSI的共享内存，一样按命名规则来

```c
shmget - allocates a shared memory segment
#include <sys/ipc.h>
#include <sys/shm.h>
int shmget(key_t key,size_t size,int shmflg);
// 成功返回shm ID；失败，返回-1
```

* key: 共享内存的唯一标识，具有亲缘关系的进程之间使用共享内存可以使用IPC_PRIVATE宏代替
* size: 是共享内存大小
* shmflg: IPC_CREAT表示创建shm，同时需要按位或一个权限，如果是=匿名IPC就无须指定这个宏，直接给权限就好

```c
shmat - shared memory operations

#include <sys/types.h>
#include <sys/shm.h>

void *shmat(int shmid, const void *shmaddr, int shmflg);
int shmdt(const void *shmaddr);
```

shmat使进程与共享内存关联起来。shmat函数中的shmaddr参数是共享内存的起始地址，传入NULL由内核帮我们寻找合适的地址。一般情况我们都是传入NULL值。

shmdt函数用于使进程分离共享内存，共享内存使用完毕之后需要用这个函数分离。分离不代表释放了这块空间，使用共享内存的双方依然要遵守"谁申请，谁释放“的原则。没有申请的一方是不需要释放的，但是双方都需要分离。

```c
shmctl - shared memory control
#include <sys/ipc.h>
#include <sys/shm.h>
int shmctl(int shmid,int cmd,struct shmid_ds *buf);
// 控制或删除共享内存
```

cmd设置IPC_RMID并且buf参数设置为NULL，就可以删除共享内存

```c
#include "../include/apue.h"
#include <sys/mman.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/wait.h>

#define MEMSIZE 1024

int main(){
    char *str;
    pid_t pid;
    int shmid;
    // 有亲缘关系的进程key参数可以使用IPC_PRIVATE宏，并且创建共享内存
    // shmflg参数不需要使用IPC_CREAT宏
    shmid = shmget(IPC_PRIVATE,MEMSIZE,0600);
    if(shmid < 0) err_sys("shmget()");
    if((pid = fork())<0) err_sys("fork()");
    if(pid==0){// 子进程
        // 关联共享内存
        str = shmat(shmid,NULL,0);
        if(str == (void*)-1) err_sys("shmat()");
        // 向共享内存写入数据
        strcpy(str,"hello!");
        // 分离共享内存
        shmdt(str);
        // 无需释放
        exit(0);
    }else{
        // 等待子进程结束后再运行，需要读取子进程写入的共享内存的数据
        wait(NULL);
        // 关联共享内存
        str = shmat(shmid,NULL,0);
        if(str == (void *)-1) err_sys("shmat()");
        // 打印读出来的数据
        puts(str);
        // 分离共享内存
        shmdt(str);
        // 释放共享内存 回收寺院
        shmctl(shmid,IPC_RMID,NULL);
        exit(0);
    }
    exit(0);

}
```




