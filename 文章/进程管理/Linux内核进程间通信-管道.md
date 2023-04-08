## 1、**管道的定义**

- 管道是第一个广泛应用的进程间通信手段。日常在终端执行shell命令时，会大量用到管道。但管道的缺陷在于只能在有亲缘关系（有共同的祖先）的进程之间使用。为了突破这个限制，后来引入了命名管道。

![img](https://pic2.zhimg.com/80/v2-13b4a46233ad254912d9bf9ba6bbdb95_720w.webp)

![img](https://pic1.zhimg.com/80/v2-011bb4b9ebed933a31b219d5599307d8_720w.webp)

## 2、**管道的用途**

- 管道是最早出现的进程间通信的手段。在shell中执行命令，经常会将上一个命令的输出作为下一个命令的输入，由多个命令配合完成一件事情。而这就是通过管道来实现的。
  在图9-3中，进程who的标准输出，通过管道传递给下游的wc进程作为标准输入，从而通过相互配合完成了一件任务。

![img](https://pic3.zhimg.com/80/v2-6026b2f409cd9c1afad043993753b3aa_720w.webp)

## 3、**管道的操作**

- 管道的作用是在具有亲缘关系的进程之间传递消息，所谓有亲缘关系，是指有同一个祖先。**所以管道并不是只可以用于父子进程通信，也可以在兄弟进程之间还可以用在祖孙之间等，反正只要共同的祖先调用了pipe函数，打开的管道文件就会在fork之后，被各个后代所共享**。
- 不过由于管道是字节流通信，没有消息边界，多个进程同时发送的字节流混在一起，则无法分辨消息，所有管道一般用于2个进程之间通信，另外管道的内容读完后不会保存，管道是单向的，一边要么读，一边要么写，不可以又读又写，想要一边读一边写，那就创建2个管道，如下图

![img](https://pic4.zhimg.com/80/v2-e551506e0ee9b917160021358b8b45b7_720w.webp)

- 管道是一种文件，可以调用read、write和close等操作文件的接口来操作管道。**另一方面管道又不是一种普通的文件，它属于一种独特的文件系统：pipefs。管道的本质是内核维护了一块缓冲区与管道文件相关联，对管道文件的操作，被内核转换成对这块缓冲区内存的操作**。下面我们来看一下如何使用管道。

```text
 #include<unistd.h>
 int pipe(int fd[2])
```

如果成功，则返回值是0，如果失败，则返回值是-1，并且设置errno。
成功调用pipe函数之后，会返回两个打开的文件描述符，**一个是管道的读取端描述符pipefd[0]，另一个是管道的写入端描述符pipefd[1]。管道没有文件名与之关联，因此程序没有选择，只能通过文件描述符来访问管道，只有那些能看到这两个文件描述符的进程才能够使用管道**。*那么谁能看到进程打开的文件描述符呢？只有该进程及该进程的子孙进程才能看到。这就限制了管道的使用范围*。

- 成功调用pipe函数之后，可以对写入端描述符pipefd[1]调用write，向管道里面写入数据，代码如下所示：

```text
write(pipefd[1],wbuf,count);
```

一旦向管道的写入端写入数据后，就可以对读取端描述符pipefd[0]调用read，读出管道里面的内容。如下所示，管道上的read调用返回的字节数等于请求字节数和管道中当前存在的字节数的最小值。如果当前管道为空，那么read调用会阻塞（如果没有设置O_NONBLOCK标志位的话）。

## 4、**管道非法read与write内核实现解析**

调用pipe函数返回的两个文件描述符中，读取端pipefd[0]支持的文件操作定义在read_pipefifo_fops，写入端pipefd[1]支持的文件操作定义在write_pipefifo_fops，其定义如下：

```text
const struct file_operations read_pipefifo_fops = { //读端相关操作
 .llseek = no_llseek,
 .read = do_sync_read,
 .aio_read = pipe_read,
 .write = bad_pipe_w, //一旦写，将调用bad_pipe_w
 .poll = pipe_poll,
 .unlocked_ioctl = pipe_ioctl,
 .open = pipe_read_open,
 .release = pipe_read_release,
 .fasync = pipe_read_fasync,
};
const struct file_operations write_pipefifo_fops = {//写端相关操作
 .llseek = no_llseek,
 .read = bad_pipe_r, //一旦读，将调用bad_pipe_r
 .write = do_sync_write,
 .aio_write = pipe_write,
 .poll = pipe_poll,
 .unlocked_ioctl = pipe_ioctl,
 .open = pipe_write_open,
 .release = pipe_write_release,
 .fasync = pipe_write_fasync,
};
```

我们可以看到，**对读取端描述符执行write操作，内核就会执行bad_pipe_w函数；对写入端描述符执行read操作，内核就会执行bad_pipe_r函数。这两个函数比较简单，都是直接返回-EBADF。因此对应的read和write调用都会失败，返回-1，并置errno为EBADF**。

```text
static ssize_t 
bad_pipe_r(struct file filp, char __user buf, size_t count, loff_t ppos) 
{ 
return -EBADF; //返回错误 
} 
static ssize_t 
bad_pipe_w(struct file filp, const char __user buf, size_t count,loff_t ppos) 
{ 
return -EBADF; 
}
```

## 5、**管道通信原理及其亲戚通信解析**

### 5.1**父子进程通信解析**

**我们只介绍了pipe函数接口，至今尚看不出来该如何使用pipe函数进行进程间通信。调用pipe之后，进程发生了什么呢？请看图9-5**。

![img](https://pic3.zhimg.com/80/v2-96b1020e08c5ee4afd1da7dd51bf43b6_720w.webp)

可以看到，调用pipe函数之后，系统给进程分配了两个文件描述符，即pipe函数返回的两个描述符。该进程既可以往写入端描述符写入信息，也可以从读取端描述符读出信息。可是一个进程管道，起不到任何通信的作用。这不是通信，而是自言自语。
如果调用pipe函数的进程随后调用fork函数，创建了子进程，情况就不一样了。fork以后，子进程复制了父进程打开的文件描述符（如图9-6所示），两条通信的通道就建立起来了。此时，可以是父进程往管道里写，子进程从管道里面读；也可以是子进程往管道里写，父进程从管道里面读。这两条通路都是可选的，但是不能都选。**原因前面介绍过，管道里面是字节流，父子进程都写、都读，就会导致内容混在一起，对于读管道的一方，解析起来就比较困难**。常规的使用方法是父子进程一方只能写入，另一方只能读出，管道变成一个单向的通道，以方便使用。如图9-7所示，父进程放弃读，子进程放弃写，变成父进程写入，子进程读出，成为一个通信的通道…

![img](https://pic3.zhimg.com/80/v2-0395cc21e8cf14f00477245e3e5b638e_720w.webp)

- **父进程如何放弃读，子进程又如何放弃写**？其实很简单，父进程把读端口pipefd[0]这个文件描述符关闭掉，子进程把写端口pipefd[1]这个文件描述符关闭掉就可以了，示例代码如下：

```text
int pipefd[2]; 
pipe(pipefd); 
switch(fork()) 
{ 
case -1: 
/fork failed, error handler here/ 
case 0: /子进程/ 
close(pipefd[1]) ; /关闭掉写入端对应的文件描述符/ 
/子进程可以对pipefd[0]调用read/ 
break； 
default: /父进程/ 
close(pipefd[0]); /父进程关闭掉读取端对应的文件描述符/ 
/父进程可以对pipefd[1]调用write, 写入想告知子进程的内容/ 
break 
}
```

![img](https://pic4.zhimg.com/80/v2-c773670f4666bf0aab36d630d88562eb_720w.webp)

### 5.2**亲缘关系的进程管道通信解析**

- 图9-8也讲述了如何在兄弟进程之间通过管道通信。如图9-8所示，父进程再次创建一个子进程B，子进程B就持有管道写入端，这时候两个子进程之间就可以通过管道通信了。父进程为了不干扰两个子进程通信，很自觉地关闭了自己的写入端。从此管道成为了两个子进程之间的单向的通信通道。在shell中执行管道命令就是这种情景，只是略有特殊之处，其特殊的地方是管道描述符占用了标准输入和标准输出两个文件描述符

![img](https://pic2.zhimg.com/80/v2-c5b7f0c17357dc449edd3596133e2c49_720w.webp)

## 6、**管道的注意事项及其性质**

### 6.1**管道有以下三条性质**

- 只有当所有的写入端描述符都已经关闭了，而且管道中的数据都被读出，对读取描述符调用read函数才返回0（及读到EOF标志）。
- 如果所有的读取端描述符都已经关闭了，此时进程再次往管道里面写入数据，写操作将会失败，并且内核会像进程发送一个SIGPIPE信号(默认杀死进程)。
- 当所有的读端与写端都已经关闭时，管道才会关闭.
- **就因为有这些特性，我们要即使关闭没用的管道文件描述符**

### 6.2**shell管道的实现**

- shell编程会大量使用管道，我们经常看到前一个命令的标准输出作为后一个命令的标准输入，来协作完成任务，如图9-9所示。管道是如何做到的呢？
  兄弟进程可以通过管道来传递消息，这并不稀奇，前面已经图示了做法。**关键是如何使得一个程序的标准输出被重定向到管道中，而另一个程序的标准输入从管道中读取呢？**

![img](https://pic4.zhimg.com/80/v2-afc1997ad3bb948e98ec7e6c38d8793f_720w.webp)

**答案就是复制文件描述符。**
对于第一个子进程，执行dup2之后，标准输出对应的文件描述符1，也成为了管道的写入端。这时候，管道就有了两个写入端，按照前面的建议，需要关闭不相干的写入端，使读取端可以顺利地读到EOF，所以应将刚开始分配的管道写入端的文件描述符pipefd[1]关闭掉。

```text
if(pipefd[1] != STDOUT_FILENO)
{
dup2(pipefd[1],STDOUT_FILENO);
close(pipefd[1]);
}
```

同样的道理,对于第二个子进程,如法炮制:

```text
if(pipefd[0] != STDIN_FILENO)
{
dup2(pipefd[0],STDIN_FILENO);
close(pipefd[0]);
}
```

- 简单来说，就是第一个子进程的标准输出被绑定到了管道的写入端，于是第一个命令的输出，写入了管道，而第二个子进程管道将其标准输入绑定到管道的读取端，只要管道里面有了内容，这些内容就成了标准输入。
- 两个示例代码，为什么要判断管道的文件描述符是否等于标准输入和标准输出呢？原因是，在调用pipe时，进程很可能已经关闭了标准输入和标准输出，调用pipe函数时，内核会分配最小的文件描述符，所以pipe的文件描述符可能等于0或1。在这种情况下，如果没有if判断加以保护，代码就变成了：

```text
dup2(1,1);
close(1);
```

这样的话，第一行代码什么也没做，第二行代码就把管道的写入端给关闭了，于是便无法传递信息了

### 6.3**与shell命令进行通信**

道的一个重要作用是和外部命令进行通信。在日常编程中，经常会需要调用一个外部命令，并且要获取命令的输出。而有些时候，需要给外部命令提供一些内容，让外部命令处理这些输入。Linux提供了popen接口来帮助程序员做这些事情。
就像system函数，即使没有system函数，我们通过fork、exec及wait家族函数一样也可以实现system的功能。但终归是不方便，system函数为我们提供了一些便利。同样的道理，只用pipe函数及dup2等函数，也能完成popen要完成的工作，但popen接口给我们提供了便利。
popen接口定义如下：

```text
#include <stdio.h>
FILE *popen(const char *command, const char *type);
int pclose(FILE *stream);
```

popen函数会创建一个管道，并且创建一个子进程来执行shell，shell会创建一个子进程来执行command。根据type值的不同，分成以下两种情况。
如果type是r：command执行的标准输出，就会写入管道，从而被调用popen的进程读到。通过对popen返回的FILE类型指针执行read或fgets等操作，就可以读取到command的标准输出，如图9-10所示。

![img](https://pic1.zhimg.com/80/v2-c08b21beae3b080597b1eb9847e3a5f0_720w.webp)

如果type是w：调用popen的进程，可以通过对FILE类型的指针fp执行write、fputs等操作，负责往管道里面写入，写入的内容经过管道传给执行command的进程，作为命令的输入，如图9-11所示

![img](https://pic3.zhimg.com/80/v2-0c6714d4c36f7f4a9c14dcea2915fce2_720w.webp)

popen函数成功时，会返回stdio库封装的FILE类型的指针，失败时会返回NULL，并且设置errno。常见的失败有fork失败，pipe失败，或者分配内存失败。

I/O结束了以后，可以调用pclose函数来关闭管道，并且等待子进程的退出。尽管popen函数返回的是FILE类型的指针，也不应调用fclose函数来关闭popen函数打开的文件流指针，因为fclose不会等待子进程的退出。pclose函数成功时会返回子进程中shell的终止状态。popen函数和system函数类似，如果command对应的命令无法执行，就如同执行了exit（127）一样。如果发生其他错误，pclose函数则返回-1。可以从errno中获取到失败的原因。

下面给出一个简单的例子，来示范下popen的用法：

```text
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<errno.h>
#include<sys/wait.h>
#include<signal.h>
#define MAX_LINE_SIZE 8192
void print_wait_exit(int status)
{
    printf("status = %d\n",status);
    if(WIFEXITED(status))
    {
        printf("normal termination,exit status = %d\n",WEXITSTATUS(status));
    }
    else if(WIFSIGNALED(status))
    {
        printf("abnormal termination,signal number =%d%s\n",
                WTERMSIG(status),
#ifdef WCOREDUMP
                WCOREDUMP(status)?"core file generated" : "");
#else
        "");
#endif
    }
}
int main(int argc ,char* argv[])
{
    FILE *fp = NULL ;
    char command[MAX_LINE_SIZE],buffer[MAX_LINE_SIZE];
    if(argc != 2 )
    {
        fprintf(stderr,"Usage: %s filename \n",argv[0]);
        exit(1);
    }
       snprintf(command,sizeof(command),"cat %s",argv[1]);
    fp = popen(command,"r");
    if(fp == NULL)
    {
        fprintf(stderr,"popen failed (%s)",strerror(errno));
        exit(2);
    }
    while(fgets(buffer,MAX_LINE_SIZE,fp) != NULL)
    {
        fprintf(stdout,"%s",buffer);
    }
    int ret = pclose(fp);
    if(ret == 127 )
    {
        fprintf(stderr,"bad command : %s\n",command);
        exit(3);
    }
    else if(ret == -1)
    {
        fprintf(stderr,"failed to get child status (%s)\n",
strerror(errno));
        exit(4);
    }
    else
    {
        print_wait_exit(ret);
    }
    exit(0);
} 
```

- 将文件名作为参数传递给程序，执行cat filename的命令。popen创建子进程来负责执行cat filename的命令，子进程的标准输出通过管道传给父进程，父进程可以通过fgets来读取command的标准输出。

### 6.4**system函数与popen函数区别**

- popen函数和system有很多相似的地方，但是也有显著的不同。调用system函数时，shell命令的执行被封装在了函数内部，所以若system函数不返回，调用system的进程就不再继续执行。但是popen函数不同，一旦调用popen函数，调用进程和执行command的进程便处于并行状态。然后pclose函数才会关闭管道，等待执行command的进程退出。换句话说，在popen之后，pclose之前，调用popen的进程和执行command的进程是并行的，这种差异带来了两种显著的不同：
- 在并行期间，调用popen的进程可能会创建其他子进程，所以标准规定popen不能阻塞SIGCHLD信号.这也意味着，popen创建的子进程可能被提前执行的等待操作所捕获。若发生这种情况，调用pclose函数时，已经无法等待command子进程的退出，这种情况下，将返回-1，并且errno为ECHILD。
- 调用进程和command子进程是并行的，所以标准要求popen不能忽略SIGINT和SIGQUIT信号。如果是从键盘产生的上述信号，那么，调用进程和command子进程都会收到信号。

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/548003903