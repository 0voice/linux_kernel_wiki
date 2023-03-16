## 1，什么是Netlink通信机制

Netlink是linux提供的用于内核和用户态进程之间的通信方式。但是注意虽然Netlink主要用于用户空间和内核空间的通信，但是也能用于用户空间的两个进程通信。只是进程间通信有其他很多方式，一般不用Netlink。除非需要用到Netlink的广播特性时。

**那么Netlink有什么优势呢？**

一般来说用户空间和内核空间的通信方式有三种：/proc、ioctl、Netlink。而前两种都是单向的，但是Netlink可以实现双工通信。Netlink协议基于BSD socket和AF_NETLINK地址簇(address family)，使用32位的端口号寻址(以前称作PID)，每个Netlink协议(或称作总线，man手册中则称之为netlink family)，通常与一个或一组内核服务/组件相关联，如NETLINK_ROUTE用于获取和设置路由与链路信息、NETLINK_KOBJECT_UEVENT用于内核向用户空间的udev进程发送通知等。

**netlink具有以下特点：**

- ① 支持全双工、异步通信(当然同步也支持)
- ② 用户空间可使用标准的BSD socket接口(但netlink并没有屏蔽掉协议包的构造与解析过程，推荐使用libnl等第三方库)
- ③ 在内核空间使用专用的内核API接口
- ④ 支持多播(因此支持“总线”式通信，可实现消息订阅)
- ⑤ 在内核端可用于进程上下文与中断上下文

## 2，用户态数据结构

**首先看一下几个重要的数据结构的关系：**

![img](https://pic4.zhimg.com/80/v2-96d22312a35be5bec7c90d6455c1e8a7_720w.webp)

**1.struct msghdr**

msghdr这个结构在socket变成中就会用到，并不算Netlink专有的，这里不在过多说明。只说明一下如何更好理解这个结构的功能。我们知道socket消息的发送和接收函数一般有这几对：recv／send、readv／writev、recvfrom／sendto。当然还有recvmsg／sendmsg，前面三对函数各有各的特点功能，而recvmsg／sendmsg就是要囊括前面三对的所有功能，当然还有自己特殊的用途。msghdr的前两个成员就是为了满足recvfrom／sendto的功能，中间两个成员msg_iov和msg_iovlen则是为了满足readv／writev的功能，而最后的msg_flags则是为了满足recv／send中flag的功能，剩下的msg_control和msg_controllen则是满足recvmsg／sendmsg特有的功能。

**2.struct sockaddr_ln**

struct sockaddr_ln为Netlink的地址，和我们通常socket编程中的sockaddr_in作用一样，他们的结构对比如下：

![img](https://pic4.zhimg.com/80/v2-05e9137ab14f9208bdbcc18909e3e7b7_720w.webp)

**struct sockaddr_nl的详细定义和描述如下：**

```cpp
struct sockaddr_nl
{
    sa_family_t nl_family; /*该字段总是为AF_NETLINK */
    unsigned short nl_pad; /* 目前未用到，填充为0*/
    __u32 nl_pid; /* process pid */
    __u32 nl_groups; /* multicast groups mask */
};
```

(1) nl_pid：在Netlink规范里，PID全称是Port-ID(32bits)，其主要作用是用于唯一的标识一个基于netlink的socket通道。通常情况下nl_pid都设置为当前进程的进程号。前面我们也说过，Netlink不仅可以实现用户-内核空间的通信还可使现实用户空间两个进程之间，或内核空间两个进程之间的通信。该属性为0时一般指内核。

(2) nl_groups：如果用户空间的进程希望加入某个多播组，则必须执行bind()系统调用。该字段指明了调用者希望加入的多播组号的掩码(注意不是组号，后面我们会详细讲解这个字段)。如果该字段为0则表示调用者不希望加入任何多播组。对于每个隶属于Netlink协议域的协议，最多可支持32个多播组(因为nl_groups的长度为32比特)，每个多播组用一个比特来表示。

**3.struct nlmsghdr**

Netlink的报文由消息头和消息体构成，struct nlmsghdr即为消息头。消息头定义在文件里，由结构体nlmsghdr表示：

```cpp
struct nlmsghdr
{
    __u32 nlmsg_len; /* Length of message including header */
    __u16 nlmsg_type; /* Message content */
    __u16 nlmsg_flags; /* Additional flags */
    __u32 nlmsg_seq; /* Sequence number */
    __u32 nlmsg_pid; /* Sending process PID */
};
```

**消息头中各成员属性的解释及说明：**

(1) nlmsg_len：整个消息的长度，按字节计算。包括了Netlink消息头本身。

(2) nlmsg_type：消息的类型，即是数据还是控制消息。目前(内核版本2.6.21)Netlink仅支持四种类型的控制消息，如下：

- a) NLMSG_NOOP-空消息，什么也不做；
- b) NLMSG_ERROR-指明该消息中包含一个错误；
- c) NLMSG_DONE-如果内核通过Netlink队列返回了多个消息，那么队列的最后一条消息的类型为NLMSG_DONE，其余所有消息的nlmsg_flags属性都被设置NLM_F_MULTI位有效。
- d) NLMSG_OVERRUN-暂时没用到。

(3) nlmsg_flags：附加在消息上的额外说明信息，如上面提到的NLM_F_MULTI。

## 3，用户空间Netlink socket API

**1.创建socket**

int socket(int domain, int type, int protocol)

domain指代地址族,即AF_NETLINK;

套接字类型为SOCK_RAW或SOCK_DGRAM,因为netlink是一个面向数据报的服务;

protocol选择该套接字使用哪种netlink特征。

**以下是几种预定义的协议类型:**

- NETLINK_ROUTE,
- NETLINK_FIREWALL,
- NETLINK_APRD,
- NETLINK_ROUTE6_FW。

可以非常容易的添加自己的netlink协议。为每一个协议类型最多可以定义32个多播组。每一个多播组用一个bitmask来表示,1<<i(0<=i<= 31),这在一组进程和内核进程协同完成一项任务时非常有用。发送多播netlink消息可以减少系统调用的数量,同时减少用来维护多播组成员信息的负担。

**2.地址绑定bind()**

bind(fd, (struct sockaddr*)&, nladdr, sizeof(nladdr));

**3.发送netlink消息**

为了发送一条netlink消息到内核或者其他的用户空间进程,另外一个struct sockaddr_nl nladdr需要作为目的地址,这和使用sendmsg()发送一个UDP包是一样的。

- 如果该消息是发送至内核的,那么nl_pid和nl_groups都置为0.
- 如果消息是发送给另一个进程的单播消息,nl_pid是另外一个进程的pid值而nl_groups为零。
- 如果消息是发送给一个或多个多播组的多播消息,所有的目的多播组必须bitmask必须or起来从而形成nl_groups域。sendmsg(fd, &, msg, 0);

**4.接收netlink消息**

一个接收程序必须分配一个足够大的内存用于保存netlink消息头和消息负载。然后其填充struct msghdr msg,再使用标准的recvmsg()函数来接收netlink消息。

当消息被正确的接收之后,nlh应该指向刚刚接收到的netlink消息的头。nladdr应该包含接收消息的目的地址,其中包括了消息发送者的pid和多播组。同时,宏NLMSG_DATA(nlh),定义在netlink.h中,返回一个指向netlink消息负载的指针。调用close(fd)关闭fd描述符所标识的socket；recvmsg(fd, &, msg, 0);

## 4、内核空间Netlink socket API

**1.创建 netlink socket**

```cpp
struct sock *netlink_kernel_create(struct net *net,
                                   int unit,unsigned int groups,
                                   void (*input)(struct sk_buff *skb),
                                   struct mutex *cb_mutex,struct module *module);
```

**参数说明：**

- (1) net：是一个网络名字空间namespace，在不同的名字空间里面可以有自己的转发信息库，有自己的一套net_device等等。默认情况下都是使用 init_net这个全局变量。
- (2) unit：表示netlink协议类型，如NETLINK_TEST、NETLINK_SELINUX。
- (3) groups：多播地址。
- (4) input：为内核模块定义的netlink消息处理函数，当有消 息到达这个netlink socket时，该input函数指针就会被引用，且只有此函数返回时，调用者的sendmsg才能返回。
- (5) cb_mutex：为访问数据时的互斥信号量。
- (6) module： 一般为THIS_MODULE。

**2.发送单播消息 netlink_unicast**

```cpp
int netlink_unicast(struct sock *ssk, struct sk_buff *skb, u32 pid, int nonblock)
```

**参数说明：**

- (1) ssk：为函数 netlink_kernel_create()返回的socket。
- (2) skb：存放消息，它的data字段指向要发送的netlink消息结构，而 skb的控制块保存了消息的地址信息，宏NETLINK_CB(skb)就用于方便设置该控制块。
- (3) pid：为接收此消息进程的pid，即目标地址，如果目标为组或内核，它设置为 0。
- (4) nonblock：表示该函数是否为非阻塞，如果为1，该函数将在没有接收缓存可利用时立即返回；而如果为0，该函数在没有接收缓存可利用定时睡眠。

**3.发送广播消息 netlink_broadcast**

```cpp
int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, u32 pid, u32 group, gfp_t allocation)
```

前面的三个参数与 netlink_unicast相同，参数group为接收消息的多播组，该参数的每一个位代表一个多播组，因此如果发送给多个多播组，就把该参数设置为多个多播组组ID的位或。参数allocation为内核内存分配类型，一般地为GFP_ATOMIC或GFP_KERNEL，GFP_ATOMIC用于原子的上下文（即不可以睡眠），而GFP_KERNEL用于非原子上下文。

**4.释放 netlink socket**

```cpp
int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, u32 pid, u32 group, gfp_t allocation)
```

## 5、用户态

**范例一**

```cpp
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <string.h>
#include <asm/types.h>
#include <linux/netlink.h>
#include <linux/socket.h>
#include <errno.h>
#define MAX_PAYLOAD 1024 // maximum payload size
#define NETLINK_TEST 25 //自定义的协议
int main(int argc, char* argv[])
{
    int state;
    struct sockaddr_nl src_addr, dest_addr;
    struct nlmsghdr *nlh = NULL; //Netlink数据包头
    struct iovec iov;
    struct msghdr msg;
    int sock_fd, retval;
    int state_smg = 0;
    // Create a socket
    sock_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_TEST);
    if(sock_fd == -1){
        printf("error getting socket: %s", strerror(errno));
        return -1;
    }
    // To prepare binding
    memset(&src_addr, 0, sizeof(src_addr));
    src_addr.nl_family = AF_NETLINK;
    src_addr.nl_pid = 100; //A：设置源端端口号
    src_addr.nl_groups = 0;
    //Bind
    retval = bind(sock_fd, (struct sockaddr*)&src_addr, sizeof(src_addr));
    if(retval < 0){
        printf("bind failed: %s", strerror(errno));
        close(sock_fd);
        return -1;
    }
    // To orepare create mssage
    nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
    if(!nlh){
        printf("malloc nlmsghdr error!\n");
        close(sock_fd);
        return -1;
}
    memset(&dest_addr,0,sizeof(dest_addr));
    dest_addr.nl_family = AF_NETLINK;
    dest_addr.nl_pid = 0; //B：设置目的端口号
    dest_addr.nl_groups = 0;
    nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
    nlh->nlmsg_pid = 100; //C：设置源端口
    nlh->nlmsg_flags = 0;
    strcpy(NLMSG_DATA(nlh),"Hello you!"); //设置消息体
    iov.iov_base = (void *)nlh;
    iov.iov_len = NLMSG_SPACE(MAX_PAYLOAD);
    //Create mssage
    memset(&msg, 0, sizeof(msg));
    msg.msg_name = (void *)&dest_addr;
    msg.msg_namelen = sizeof(dest_addr);
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    //send message
    printf("state_smg\n");
    state_smg = sendmsg(sock_fd,&msg,0);
    if(state_smg == -1)
    {
        printf("get error sendmsg = %s\n",strerror(errno));
    }
    memset(nlh,0,NLMSG_SPACE(MAX_PAYLOAD));
    //receive message
    printf("waiting received!\n");
    while(1){
        printf("In while recvmsg\n");
        state = recvmsg(sock_fd, &msg, 0);
        if(state<0)
        {
            printf("state<1");
        }
        printf("Received message: %s\n",(char *) NLMSG_DATA(nlh));
    }
    close(sock_fd);
    return 0;
}
```

上面程序首先向内核发送一条消息；“Hello you”，然后进入循环一直等待读取内核的回复，并将收到的回复打印出来。如果看上面程序感觉很吃力，那么应该首先复习一下UDP中使用sendmsg的用法，特别时struct msghdr的结构要清楚，这里再赘述。

**下面主要分析与UDP发送数据包的不同点：**

- \1. socket地址结构不同，UDP为sockaddr_in，Netlink为struct sockaddr_nl；
- \2. 与UDP发送数据相比，Netlink多了一个消息头结构struct nlmsghdr需要我们构造。

注意代码注释中的A、B、C三处分别设置了pid。首先解释一下什么是pid，网上很多文章把这个字段说成是进程的pid，其实这完全是望文生义。这里的pid和进程pid没有什么关系，仅仅相当于UDP的port。对于UDP来说port和ip标示一个地址，那对我们的NETLINK_TEST协议（注意Netlink本身不是一个协议）来说，pid就唯一标示了一个地址。所以你如果用进程pid做为标示当然也是可以的。当然同样的pid对于NETLINK_TEST协议和内核定义的其他使用Netlink的协议是不冲突的（就像TCP的80端口和UDP的80端口）。

下面分析这三处设置pid分别有什么作用，首先A和B位置的比较好理解，这是在地址（sockaddr_nl）上进行的设置，就是相当于设置源地址和目的地址（其实是端口），只是注意B处设置pid为0，0就代表是内核，可以理解为内核专用的pid，那么用户进程就不能用0做为自己的pid吗？这个只能说如果你非要用也是可以的，只是会产生一些问题，后面在分析。

接下来看为什么C处的消息头仍然需要设置pid呢？这里首先要知道一个前提：内核不会像UDP一样根据我们设置的原、目的地址为我们构造消息头，所以我们不在包头写入我们自己的地址（pid），那内核怎么知道是谁发来的报文呢？当然如果内核只是处理消息不需要回复进程的话舍不设置这个消息头pid都可以。

所以每个pid的设置功能不同：A处的设置是要设置发送者的源地址，有人会说既然源地址又不会自动填充到报文中，我们为什么还要设置这个，因为你还可能要接收回复啊。就像寄信，你连“门牌号”都没有，即使你在写信时候写上你的地址是100号，对方回信目的地址也是100号，但是邮局发现根本没有这个地址怎么可能把信送到你手里呢？所以A的主要作用是注册源地址，保证可以收到回复，如果不需要回复当然可以简单将pid设置为0；B处自然就是收信人的地址，pid为0代表内核的地址，假如有一个进程在101号上注册了地址，并调用了recvmsg，如果你将B处的pid设置为101，那数据包就发给了另一个进程，这就实现了使用Netlink进行进程间通信；C相当于你在信封上写的源地址，通常情况下这个应该和你的真实地址（A）处注册的源地址相同，当然你要是不想收到回信，又想恶搞一下或者有特殊需求，你可以写成其他进程注册的pid（比如101）。这和我们现实中寄信是一样的，你给你朋友写封情书，把写信人写成你的另一个好基友，然后后果你懂得……

好了，有了这个例子我们就大概知道用户态怎么使用Netlink了。

## 6、内核态程序

范例一

```cpp
#include <linux/init.h>
#include <linux/module.h>
#include <linux/timer.h>
#include <linux/time.h>
#include <linux/types.h>
#include <net/sock.h>
#include <net/netlink.h>
#define NETLINK_TEST 25
#define MAX_MSGSIZE 1024
int stringlength(char *s);
int err;
struct sock *nl_sk = NULL;
int flag = 0;
//向用户态进程回发消息
void sendnlmsg(char *message, int pid)
{
    struct sk_buff *skb_1;
    struct nlmsghdr *nlh;
    int len = NLMSG_SPACE(MAX_MSGSIZE);
    int slen = 0;
    if(!message || !nl_sk)
    {
        return ;
    }
    printk(KERN_ERR "pid:%d\n",pid);
    skb_1 = alloc_skb(len,GFP_KERNEL);
    if(!skb_1)
    {
        printk(KERN_ERR "my_net_link:alloc_skb error\n");
    }
    slen = stringlength(message);
    nlh = nlmsg_put(skb_1,0,0,0,MAX_MSGSIZE,0);
    NETLINK_CB(skb_1).pid = 0;
    NETLINK_CB(skb_1).dst_group = 0;
    message[slen]= '\0';
    memcpy(NLMSG_DATA(nlh),message,slen+1);
    printk("my_net_link:send message '%s'.\n",(char *)NLMSG_DATA(nlh));
    netlink_unicast(nl_sk,skb_1,pid,MSG_DONTWAIT);
}
int stringlength(char *s)
{
    int slen = 0;
    for(; *s; s++)
    {
        slen++;
    }
    return slen;
}
//接收用户态发来的消息
void nl_data_ready(struct sk_buff *__skb)
 {
     struct sk_buff *skb;
     struct nlmsghdr *nlh;
     char str[100];
     struct completion cmpl;
     printk("begin data_ready\n");
     int i=10;
     int pid;
     skb = skb_get (__skb);
     if(skb->len >= NLMSG_SPACE(0))
     {
         nlh = nlmsg_hdr(skb);
         memcpy(str, NLMSG_DATA(nlh), sizeof(str));
         printk("Message received:%s\n",str) ;
         pid = nlh->nlmsg_pid;
         while(i--)
        {//我们使用completion做延时，每3秒钟向用户态回发一个消息
            init_completion(&cmpl);
            wait_for_completion_timeout(&cmpl,3 * HZ);
            sendnlmsg("I am from kernel!",pid);
        }
         flag = 1;
         kfree_skb(skb);
    }
 }
// Initialize netlink
int netlink_init(void)
{
    nl_sk = netlink_kernel_create(&init_net, NETLINK_TEST, 1,
                                 nl_data_ready, NULL, THIS_MODULE);
    if(!nl_sk){
        printk(KERN_ERR "my_net_link: create netlink socket error.\n");
        return 1;
    }
    printk("my_net_link_4: create netlink socket ok.\n");
    return 0;
}
static void netlink_exit(void)
{
    if(nl_sk != NULL){
        sock_release(nl_sk->sk_socket);
    }
    printk("my_net_link: self module exited\n");
}
module_init(netlink_init);
module_exit(netlink_exit);
MODULE_AUTHOR("zhao_h");
MODULE_LICENSE("GPL");
```

附上内核代码的Makefile文件：

```text
ifneq ($(KERNELRELEASE),)
obj-m :=netl.o
else
KERNELDIR ?=/lib/modules/$(shell uname -r)/build
PWD :=$(shell pwd)
default:
$(MAKE) -C $(KERNELDIR) M=$(PWD) modules
endif
```

我们将内核模块insmod后，运行用户态程序，结果如下：



![img](https://pic4.zhimg.com/80/v2-6437cb1fc76f77b7a302356ce2003603_720w.webp)

这个结果复合我们的预期，但是运行过程中打印出“state_smg”卡了好久才输出了后面的结果。这时候查看客户进程是处于D状态的（不了解D状态的同学可以google一下）。这是为什么呢？因为进程使用Netlink向内核发数据是同步，内核向进程发数据是异步。什么意思呢？也就是用户进程调用sendmsg发送消息后，内核会调用相应的接收函数，但是一定到这个接收函数执行完用户态的sendmsg才能够返回。我们在内核态的接收函数中调用了10次回发函数，每次都等待3秒钟，所以内核接收函数30秒后才返回，所以我们用户态程序的sendmsg也要等30秒后才返回。相反，内核回发的数据不用等待用户程序接收，这是因为内核所发的数据会暂时存放在一个队列中。

再来回到之前的一个问题，用户态程序的源地址（pid）可以用0吗？我把上面的用户程序的A和C处pid都改为了0，结果一运行就死机了。为什么呢？我们看一下内核代码的逻辑，收到用户消息后，根据消息中的pid发送回去，而pid为0，内核并不认为这是用户程序，认为是自身，所有又将回发的10个消息发给了自己（内核），这样就陷入了一个死循环，而用户态这时候进程一直处于D。

另外一个问题，如果同时启动两个用户进程会是什么情况？答案是再调用bind时出错：“Address already in use”，这个同UDP一样，同一个地址同一个port如果没有设置SO_REUSEADDR两次bind就会出错，之后我用同样的方式再Netlink的socket上设置了SO_REUSEADDR，但是并没有什么效果。

## 7、用户态

**范例二**

之前我们说过UDP可以使用sendmsg／recvmsg也可以使用sendto／recvfrom，那么Netlink同样也可以使用sendto／recvfrom。**具体实现如下：**

```cpp
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <string.h>
#include <asm/types.h>
#include <linux/netlink.h>
#include <linux/socket.h>
#include <errno.h>
#define MAX_PAYLOAD 1024 // maximum payload size
#define NETLINK_TEST 25
int main(int argc, char* argv[])
{
    struct sockaddr_nl src_addr, dest_addr;
    struct nlmsghdr *nlh = NULL;
    int sock_fd, retval;
    int state,state_smg = 0;
    // Create a socket
    sock_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_TEST);
    if(sock_fd == -1){
        printf("error getting socket: %s", strerror(errno));
        return -1;
    }
    // To prepare binding
    memset(&src_addr, 0, sizeof(src_addr));
    src_addr.nl_family = AF_NETLINK;
    src_addr.nl_pid = 100;
    src_addr.nl_groups = 0;
    //Bind
    retval = bind(sock_fd, (struct sockaddr*)&src_addr, sizeof(src_addr));
    if(retval < 0){
        printf("bind failed: %s", strerror(errno));
        close(sock_fd);
        return -1;
    }
    // To orepare create mssage head
    nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
    if(!nlh){
        printf("malloc nlmsghdr error!\n");
        close(sock_fd);
        return -1;
    }
    memset(&dest_addr,0,sizeof(dest_addr));
    dest_addr.nl_family = AF_NETLINK;
    dest_addr.nl_pid = 0;
    dest_addr.nl_groups = 0;
    nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
    nlh->nlmsg_pid = 100;
    nlh->nlmsg_flags = 0;
    strcpy(NLMSG_DATA(nlh),"Hello you!");
    //send message
    printf("state_smg\n");
    sendto(sock_fd,nlh,NLMSG_LENGTH(MAX_PAYLOAD),0,(struct sockaddr*)(&dest_addr),sizeof(dest_addr));
    if(state_smg == -1)
    {
        printf("get error sendmsg = %s\n",strerror(errno));
    }
    memset(nlh,0,NLMSG_SPACE(MAX_PAYLOAD));
    //receive message
    printf("waiting received!\n");
while(1){
        printf("In while recvmsg\n");
state=recvfrom(sock_fd,nlh,NLMSG_LENGTH(MAX_PAYLOAD),0,NULL,NULL);
        if(state<0)
        {
            printf("state<1");
        }
        printf("Received message: %s\n",(char *) NLMSG_DATA(nlh));
        memset(nlh,0,NLMSG_SPACE(MAX_PAYLOAD));
    }
    close(sock_fd);
    return 0;
}
```

熟悉UDP编程的同学看到这个程序一定很熟悉，除了多了一个Netlink消息头的设置。但是我们发现程序中调用了bind函数，这个函数再UDP编程中的客户端不是必须的，因为我们不需要把UDP socket与某个地址关联，同时再发送UDP数据包时内核会为我们分配一个随即的端口。但是对于Netlink必须要有这一步bind，因为Netlink内核可不会为我们分配一个pid。再强调一遍消息头（nlmsghdr）中的pid是告诉内核接收端要回复的地址，但是这个地址存不存在内核并不关心，这个地址只有用户端调用了bind后才存在。

我们看到这两个例子都是用户态首先发起的，那Netlink是否支持内核态主动发起的情况呢？

当然是可以的，只是内核一般需要事件触发，这里，只要和用户态约定号一个地址（pid），内核直接调用netlink_unicast就可以了。

------

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/458996875