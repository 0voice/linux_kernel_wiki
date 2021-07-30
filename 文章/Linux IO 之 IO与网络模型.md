## Linux内核针对不同并发场景的工具实现

![image](https://user-images.githubusercontent.com/87457873/127649604-761ead2e-8c72-4876-a52f-8310f3c3b49c.png)

### atomic 原子变量

x86在多核环境下，多核竞争数据总线时，提供Lock指令进行锁总线操作。保证“读-修改-写”的操作在芯片级的原子性。

### spinlock 自旋锁

自旋锁将当前线程不停地执行循环体，而不改变线程的运行状态，在CPU上实现忙等，以此保证响应速度更快。这种类型的线程数不断增加时，性能明显下降。所以自旋锁保护的临界区必须小，操作过程必须短。

### semaphore 信号量

信号量用于保护有限数量的临界资源，信号量在获取和释放时，通过自旋锁保护，当有中断会把中断保存到eflags寄存器，最后再恢复中断。

### mutex 互斥锁

为了控制同一时刻只有一个线程进入临界区，让无法进入临界区的线程休眠。

### rw-lock 读写锁

读写锁，把读操作和写操作分别进行加锁处理，减小了加锁粒度，优化了读大于写的场景。

### preempt 抢占

* 时间片用完后调用schedule函数。
* 由于IO等原因自己主动调用schedule。
* 其他情况，当前进程被其他进程替换的时候。

### per-cpu 变量

linux为解决cpu 各自使用的L2 cache 数据与内存中的不一致的问题。

### RCU机制 (Read, Copy, Update)

用于解决多个CPU同时读写共享数据的场景。它允许多个CPU同时进行写操作，不使用锁，并且实现垃圾回收来处理旧数据。

![image](https://user-images.githubusercontent.com/87457873/127649772-3121520b-8989-4176-a472-43719b64eb10.png)

### 内存屏障 memory-barrier

程序运行过程中，对内存访问不一定按照代码编写的顺序来进行。

* 编译器对代码进行优化。
* 多cpu架构存在指令乱序访问内存的可能。

## I/O 与网络模型

介绍各种各样的I/O模型，包括以下场景：

* 阻塞 & 非阻塞
* 多路复用
* Signal IO
* 异步 IO
* libevent

现实生活中的场景复杂，Linux CPU和IO行为，他们之间互相等待。例如，阻塞的IO可能会让CPU暂停。

I/O模型很难说好与坏，只能说在某些场景下，更适合某些IO模型。其中，1、4 更适合块设备，2、3 更适用于字符设备。

为什么硬盘没有所谓的 多路复用，libevent，signal IO？

> 因为select(串口), epoll（socket） 这些都是在监听事件，所以各种各样的IO模型，更多是描述字符设备和网络socket的问题。但硬盘的文件，只有读写，没有 epoll这些。
> 这些IO模型更多是在字符设备，网络socket的场景。

### 为什么程序要选择正确的IO模型？

蓝色代表：cpu，红色代表：io

![image](https://user-images.githubusercontent.com/87457873/127649947-6c89dd89-5076-4aca-af0a-69f154cf26b4.png)

如上图，某个应用打开一个图片文件，先需要100ms初始化，接下来100ms读这个图片。那打开这个图片就需要200ms。

但是 是否可以开两个线程，同时做这两件事？

![image](https://user-images.githubusercontent.com/87457873/127649983-79730c12-91e9-462f-8abf-5fbf9634bd08.png)

如上图，网络收发程序，如果串行执行，CPU和IO会需要互相等待。<br>
为什么CPU和IO可以并行？因为一般硬件，IO通过DMA，cpu消耗比较小，在硬件上操作的时间更长。CPU和硬盘是两个不同的硬件。

再比如开机加速中systemd使用的readahead功能:<br>
第一次启动过程，读的文件，会通过Linux inotify监控linux内核文件被操作的情况，记录下来。第二次启动，后台有进程直接读这些文件，而不是等到需要的时候再读。

![image](https://user-images.githubusercontent.com/87457873/127650021-0bed9b00-4448-4c7a-b9a4-ccff573a50ea.png)

I/O模型会深刻影响应用的最终性能，阻塞 & 非阻塞 、异步 IO 是针对硬盘， 多路复用、signal io、libevent 是针对字符设备和 socket。

### 简单的IO模型

![image](https://user-images.githubusercontent.com/87457873/127650047-8bb138e2-f652-4115-8ea6-0498b3e1be1a.png)

当一个进程需要读 键盘、触屏、鼠标时，进程会阻塞。但对于大量并发的场景，阻塞IO无法搞定，也可能会被信号打断。

内核程序等待IO，gobal fifo read不到

一般情况select返回，会调用 if signal_pending，进程会返回 ERESTARTSYS；此时，进程的read 返回由singal决定。有可能返回（EINTR），也有可能不返回。

#### demo:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <errno.h>
#include <string.h>

static void sig_handler(int signum)
{
	printf("int handler %d\n", signum);
}

int main(int argc, char **argv)
{
	char buf[100];
	ssize_t ret;
	struct sigaction oldact;
	struct sigaction act;

	act.sa_handler = sig_handler;
	act.sa_flags = 0;
	//  act.sa_flags |= SA_RESTART;
	sigemptyset(&act.sa_mask);
	if (-1 == sigaction(SIGUSR1, &act, &oldact)) {
		printf("sigaction failed!/n");
		return -1;
	}

	bzero(buf, 100);
	do {
		ret = read(STDIN_FILENO, buf, 10);
		if ((ret == -1) && (errno == EINTR))
			printf("retry after eintr\n");
	} while((ret == -1) && (errno == EINTR));

	if (ret > 0)
		printf("read %d bytes, content is %s\n", ret, buf);
	return 0;
}
```

![image](https://user-images.githubusercontent.com/87457873/127650094-8544cd30-d8de-4773-8a5c-b5d865115dce.png)

一个阻塞的IO，在睡眠等IO时Ready，但中途被信号打断，linux响应信号，read/write请求阻塞。<br>
配置信号时，在SA_FLAG是不是加“自动”，SA_RESTART指定 被阻塞的IO请求是否重发，并且应用中可以捕捉。加了SA_RESTART重发，就不会返回出错码EINTR。<br>
没有加SA_RESTART重发，就会返回出错码（EINTR），这样可以检测read被信号打断时的返回。<br>

但Linux中有一些系统调用，即便你加了自动重发，也不能自动重发。man signal.

![image](https://user-images.githubusercontent.com/87457873/127650145-bcad6c25-0a99-4158-af47-7ead22941fb2.png)

当使用阻塞IO时，要小心这部分。

![image](https://user-images.githubusercontent.com/87457873/127650167-86e622e8-4e04-437d-9e29-aa3cacc78968.png)

### 多进程、多线程模型

当有多个socket消息需要处理，阻塞IO搞不定，有一种可能是多个进程/线程，每当有一个连接建立（accept socket)，都会启动一个线程去处理新建立的连接。但是，这种模型性能不太好，创建多进程、多线程时会有开销。

经典的C10K问题，意思是 在一台服务器上维护1w个连接，需要建立1w个进程或者线程。那么如果维护1亿用户在线，则需要1w台服务器。

IO多路复用，则是解决以上问题的场景。

总结：多进程、多线程模型企图把每一个fd放到不同的线程/进程处理，避免阻塞的问题，从而引入了进程创建\撤销，调度的开销。能否在一个线程内搞定所有IO? -- 这就是多路复用的作用。

### 多路复用

![image](https://user-images.githubusercontent.com/87457873/127650193-d91a7751-d9be-43e4-ace1-d153f18d82e7.png)

#### select

select：效率低，性能不太好。不能解决大量并发请求的问题。

它把1000个fd加入到fd_set（文件描述符集合），通过select监控fd_set里的fd是否有变化。如果有一个fd满足读写事件，就会依次查看每个文件描述符，那些发生变化的描述符在fd_set对应位设为1，表示socket可读或者可写。

Select通过轮询的方式监听，对监听的FD数量 t通过FD_SETSIZE限制。

两个问题：

1、select初始化时，要告诉内核，关注1000个fd， 每次初始化都需要重新关注1000个fd。前期准备阶段长。<br>
2、select返回之后，要扫描1000个fd。 后期扫描维护成本大，CPU开销大。

#### epoll

epoll ：在内核中的实现不是通过轮询的方式，而是通过注册callback函数的方式。当某个文件描述符发现变化，就主动通知。成功解决了select的两个问题，“epoll 被称为解决 C10K 问题的利器。”

1、select的“健忘症”，一返回就不记得关注了多少fd。api 把告诉内核等哪些文件，和最终监听哪些文件，都是同一个api。而epoll，告诉内核等哪些文件 和具体等哪些文件分开成两个api，epoll的“等”返回后，还是知道关注了哪些fd。<br>
2、select在返回后的维护开销很大，而epoll就可以直接知道需要等fd。

![image](https://user-images.githubusercontent.com/87457873/127650306-b4419ff5-164e-4678-80f4-6f839ad44245.png)

![image](https://user-images.githubusercontent.com/87457873/127650321-094aa0ad-c49b-46d9-8100-4b7c3cc60f53.png)

epoll获取事件的时候，无须遍历整个被侦听的描述符集，只要遍历那些被内核I/O事件异步唤醒而加入就绪队列的描述符集合。

epoll_create: 创建epoll池子。<br>
epoll_ctl：向epoll注册事件。告诉内核epoll关心哪些文件，让内核没有健忘症。<br>
epoll_wait：等待就绪事件的到来。专门等哪些文件，第2个参数 是输出参数，包含满足的fd，不需要再遍历所有的fd文件。<br>

![image](https://user-images.githubusercontent.com/87457873/127650368-39c111d9-a074-4f0b-befd-3c0e9a29867b.png)

如上图，epoll在CPU的消耗上，远低于select，这样就可以在一个线程内监控更多的IO。

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/epoll.h>
#include <sys/stat.h>

static void call_epoll(void)
{
   int epfd, fifofd, pipefd;
   struct epoll_event ev, events[2];
   int ret;

   epfd = epoll_create(2);
   if (epfd < 0) {
   	perror("epoll_create()");
   	return;
   }

   ev.events = EPOLLIN|EPOLLET;

   fifofd = open("/dev/globalfifo", O_RDONLY, S_IRUSR);
   printf("fifo fd:%d\n", fifofd);
   ev.data.fd = fifofd;
   ret = epoll_ctl(epfd, EPOLL_CTL_ADD, fifofd, &ev);

   pipefd = open("pipe", O_RDONLY|O_NONBLOCK, S_IRUSR);
   printf("pipe fd:%d\n", pipefd);
   ev.data.fd = pipefd;
   ret = epoll_ctl(epfd, EPOLL_CTL_ADD, pipefd, &ev);

   while(1) {
   	ret = epoll_wait(epfd, events, 2, 50000);
   	if (ret < 0) {
   		perror("epoll_wait()");
   	} else if (ret == 0) {
   		printf("No data within 50 seconds.\n");
   	} else {
   		int i;
   		for(i=0;i<ret;i++) {
   			char buf[100];
   			read(events[i].data.fd, buf, 100);
   			printf("%s is available now:, %s\n",
   					events[i].data.fd==fifofd? "fifo":"pipe", buf);
   		}
   	}
   }
_out:
   close(epfd);
}

int main()
{
   call_epoll();
   return 0;
}
```
总结：epoll是几乎是大规模并行网络程序设计的代名词，一个线程里可以处理大量的tcp连接，cpu消耗也比较低。很多框架模型，nginx, nodejs, 底层均使用epoll实现。

#### signal IO

目前在linux中很少被用到，Linux内核某个IO事件ready，通过kill出一个signal，应用程序在signal IO上绑定处理函数。

![image](https://user-images.githubusercontent.com/87457873/127650438-dafaa967-0eb9-4886-9a90-4200fd3d2708.png)

kernel发现设备读写事件变化，调用一个 kill fa_sync ，应用程序绑定signal_io上的事件。

![image](https://user-images.githubusercontent.com/87457873/127650461-1e5fc79c-496e-4c7e-a1d4-dfa0c922644e.png)

#### 异步IO

![image](https://user-images.githubusercontent.com/87457873/127650494-6b63c04c-7b7e-4afa-9f8e-d37f36046651.png)

Linux中

不要把aio串起来，

基于epoll等api进行上层的封装，再基于事件编程。某个事件成立了，就开始去做某件事。

#### libevent

![image](https://user-images.githubusercontent.com/87457873/127650540-a0005da6-290f-464a-a14b-f05088b9c9d8.png)

就像MFC一样，界面上的按钮，VC会产生一个on_button，调对应的函数。是一种典型的事件循环。

本质上还是用了epoll，只是基于事件编程。

![image](https://user-images.githubusercontent.com/87457873/127650560-697e2b4e-61ab-4e43-960f-67d6f2aa2b1a.png)

