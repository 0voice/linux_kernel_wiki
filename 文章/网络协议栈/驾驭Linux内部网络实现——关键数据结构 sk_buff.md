## 背景

在 Linux 的网络栈实现代码中，引用到了一些数据结构。要理解 Linux 内部的网络实现，需要先理清这些数据结构的作用。关键数据结构主要有两个: sk_buff 和 net_device。

struct sk_buff: 是整个网络数据包存储的地方。这个数据结构会被网络协议栈中的各层用来储存它们的协议头、用户数据和其他它们完成工作需要的数据。

struct net_device: 在 Linux 内核中，这个数据结构将用来代表网络设备。它会包含设备的硬件和软件配置信息。

在 Linux 的网络实现中，核心数据结构还有struct sock, 它被用来储存 socket 的信息。但是 Socket 其实是内核为用户态程序提供的一组 Api, 用来访问内核的网络栈实现，所以它不属于内核内部的网络实现，也就不再这里介绍了。

本文将先着重理解 sk_buff 数据结构。

## Socket Buffer: sk_buff

> sk_buff: 在本文中后面部分也会被称为缓冲区

在 Linux 内核的网络代码中，这或许是最重要的数据结构，用来表示已接收或将要传输的数据。定义在 <include/linux/skbuff.h> 中，它由许多变量组成，目标就是满足所有网络协议的需要。

sk_buff 的结构随着内核的迭代已经被添加了许多新的选项，已经存在的字段也被重新整理了很多遍。可将内部的字段分为以下几类：

* Layout 负责内存布局的字段
* General 通用的字段
* Feature-specific 对应特别功能字段
* Management functions 一些用来管理 sk_buff 的函数

sk_buff 在不同的网络层被使用（MAC 或其他在 L2 的协议，在 L3 的 IP 协议，在 L4 的 TCP 或 UDP 等），当它从一层传递到另一层时，各个字段也会发生变化。在被传递到 L3 之前，L4 会追加头信息，然后在被传递到 L2 之前，L3 会追加头信息。从一层传递到另一层时，通过追加头信息的方式比将数据在层之间拷贝会更有效率。由于要在 buff 的开头增加空间（与平时常见的在尾部追加空间相比）是一项复杂的操作，内核便提供了 skb_reserve 函数执行这个操作。因此，随着 buffer 从上层到下层的传递，每层协议做的第一件事就是调用 skb_reserve 去为它们的协议头在 buffer 的头部分配空间。在后面，我们将通过一个例子去了解内核如何在当 buffer 在各个层间传递时，确保为每一层保留了足够的空间让它们添加它们自己的协议头。

在接收数据时，buffer 会被从下层到上层传递，在从下到上的过程中，前一层的协议头对于当前层来说已经没有用了。比如：L2 的协议头只会被处理 L2 协议的设备驱动程序使用，L3 并不关心 L2 的头。那么内核怎么做的呢? 内核的实现是：** sk_buff 中有一个指针会指向当前位于的层次的协议的协议头的内存开始地址，于是从 L2 到 L3 时，只需将指向 L2 头部的指针移动到 L3 的头部即可**（又是一步追求效率的操作）。

## 网络选项和内核结构

内核的网络代码提供了大量有用但不是必须的选项，例如防火墙，多播等功能。这些选项都需要内核数据结构中的其他字段。因此，sk_buff 使用 C 语言的预处理命令 #ifdef 来做条件编译。例如在 sk_buff 定义的后面部分:

```c
struct sk_buff {
//......
    #ifdef CONFIG_NET_SCHED 
        __u32 tc_index;
    #ifdef CONFIG_NET_CLS_ACT 
        __u32 tc_verd;
        __u32 tc_classid;
    #endif
    #endif
}
```

通过在编译 Linux 时配置不同的编译选项，能够让编译出来的内核支持不同的功能。

## Layout Fields

在 sk_buff 中存在一些字段，它们存在的意义只是为了搜索的方便和数据结构的组织。这类字段称为 Layout Fileds。Linux 内核把系统中所有的 sk_buff 实例维护在一个双向链表中。但是组织这个链表比传统的双向链表要复杂一点。

和任何双向链表类似，sk_buff 链表的每个节点也通过 next 和 prev 分别指向后继和前驱节点。但是 sk_buff 链表还要求：每个节点必须能够很快的找到整个链表的头节点。为了实现这个要求，一个额外的数据结构(sk_buff_head)被添加到链表的头部，作为一个空节点：

```c
struct sk_buff_head {
	/* These two members must be first. */
	struct sk_buff	*next;
	struct sk_buff	*prev;

	__u32		qlen;
	spinlock_t	lock;
};
```
* qlen: 表示链表中的节点数
* lock: 用作多线程同步

sk_buff 和 sk_buff_head 开始的两个节点(next prev)是相同的。即使 sk_buff_head 比 sk_buff 更轻量化，也允许这两种结构在链表中共存。另外，可以使用相同函数来操作 sk_buff 和 sk_buff_head。

为了实现通过每个节点都能快速找到链表头，每个节点都会包含一个指向链表中唯一的 sk_buff_head 的指针（list）。

![image](https://user-images.githubusercontent.com/87457873/127461275-2d349c99-38c1-44b6-a381-6a487add8c0f.png)

## sk_buff 其他字段

### struct sock sk

一个指向 sock 数据结构的指针，表示 sock 对应 socket 拥有这个 sk_buff。当数据是由本地进程生成或接收时需要这个指针，因为数据和 socket相关的信息会被 L4（TCP或UDP）和用户态的程序使用。当一个 sk_buff 仅仅是被转发时（也就是说，源和目标地址不在本地计算机上），这个指针是不需要的，因此将会是 NULL。

### unsigned int len

表示在 buffer 中数据区域的大小。该长度既包括主缓冲区的数据长度，也包括片段中的数据。因为协议头在向上传递中会被丢弃，在向下传递中会被添加，所以它的值会随着 buffer 在各层间传递而改变。

### unsiged int data_len

和 len 不同的是，data_len 只记录分段中的数据大小。

### unsigned int mac_len

MAC 头部的长度

### atomic_t users

sk_buff 的引用计数，或引用了此 sk_buff 缓冲区的对象数。 主要用途是避免在有人还在使用时就释放了 sk_buff。users的值可以通过 atomic_inc 和 atomic_dec直接增加和减少，但更多的时候是通过`skb_get和kfree_skb` 函数进行。

### unsigned int truesize

这个字段表示 buffer 的总大小，包括了 sk_buff 自己的占用。在执行 alloc_skb 函数时该字段被初始化。

    #define SKB_TRUESIZE(X) ((X) +						\
       SKB_DATA_ALIGN(sizeof(struct sk_buff)) +	\
       SKB_DATA_ALIGN(sizeof(struct skb_shared_info)))

* unsigned char head
* unsigned char end
* unsigned char data
* unsigned char tail

上面4个指针用来表示 buffer 中数据域的边界。当每一层为了任务而准备 buffer 时，为了协议头或数据，可能会分配更多的空间。 head 和 end 指向了 buffer 被分配的内存区域的开始和结束， data 和 tail 指向其中实际数据域的开始和结束。

![image](https://user-images.githubusercontent.com/87457873/127461623-6ab77e77-099c-4a58-b2ca-bafadafe8116.png)

每一层能够在 head 和 data 之间的区域填充协议头，或者在 tail 和 end 之间的区域填充新的数据。

### void (destructor)(…)

这个函数指针能够在运行时被赋值，从而在一个 buffer 被移除时，执行一些操作。当一个 buffer 不属于一个 socket 时，这个函数指针通常为空。当一个 buffer 属于一个 socket 时，这个函数指针通常被设置为 sock_rfree 或 sock_wfree。两个 sock_xxx 函数用于更新 socket 在它的队列中持有的大量内存。

### General Fields
在 sk_buff 中存在一些通用目的的字段，这些字段没有与特定的内核功能绑定：

### struct timeval stamp
这个字段通常仅对接收到的数据包有意义。这是一个时间戳，表示何时接收到一个数据包，或者何时计划发送一个数据包。它由接收 net_timestamp 参数的函数 netif_rx 设置，在接收每个数据包后由设备驱动程序调用。

### struct net_device dev
该字段描述了网络设备。 dev代表的设备的作用取决于缓冲区中存储的数据包是要发送还是刚刚被接收。<br>
当收到数据包后，设备驱动程序会使用指向接收数据接口的指针来更新此字段。<br>
当要发送数据包时，此参数表示将通过其发送出去的设备。<br>

### struct net_device input_dev
表示数据包到来的设备。当数据包是在本地生成的，其值是 NULL。

### struct net_device real_dev
该字段仅对虚拟设备有意义，代表与虚拟设备关联的实际设备。 例如，Bonding（将两个或多个网络接口组合或合并为一个接口） 和 VLAN(virtual local area network) 接口使用它来记住从何处接收到真实设备入口流量。

    union {…} h
    union {…} nh
    union {…} mac

上面的3个指针表示了 TCP/IP 协议栈中的协议头: h 代表 L4, nh 代表 L3, mac 代表 L2。每个字段都指向的是各种共用体结构，某个结构只能被内核中对应的层的协议理解。比如：h 共用体就包含了L4上的每个协议能理解的头信息。每个共用体的一个成员称为raw，用于初始化。 所有以后的访问都通过协议特定的成员进行。

当接收到数据包时，负责处理第 n 层协议头的函数从第 n-1 层接收一个 buffer，其中skb->data 指向第 n 层协议头的开头。处理第n层的函数会为此层初始化适当的指针（例如，L3 的处理函数会为 skb->nh 赋值）以保留 skb->data 字段，因为当 skb->data 被赋值为 buffer 内的其他偏移量时，该指针的内容将在下一层的处理过程中丢失。然后，该函数完成第 n 层的处理，并在将数据包传递到第 n+1 层处理程序之前，更新 skb->data 使其指向第 n 层协议头的末尾（即第n+1 层协议头的开始位置）。

![image](https://user-images.githubusercontent.com/87457873/127462034-7946a510-ed17-44ce-93b7-b9a23e142f68.png)

### struct dst_entry dst
由路由子系统使用。 因为数据结构非常复杂，并且需要了解其他子系统的工作原理，所以留在以后在详解。

### char cb[40]
control buffer 的简称，或存储一些私有信息，由各层维护以供内部使用。它是在 sk_buff 结构中静态分配的（当前大小为40个字节），并且足够大以容纳每一层所需的任何私有数据。在每一层的代码中，访问都是通过宏进行的，以使代码更具可读性。例如，TCP使用该空间存储 tcp_skb_cb 数据结构，该数据结构在 include/net/tcp.h 中定义：

```c
struct tcp_skb_cb {
    //...
	__u32		seq;		/* Starting sequence number	*/
	__u32		end_seq;	/* SEQ + FIN + SYN + datalen	*/
	__u8		tcp_flags;	/* TCP header flags. (tcp[13])	*/
	__u32		ack_seq;	/* Sequence number ACK'd	*/
    //...
};
```
这是 TCP 代码访问结构的宏。宏仅由一个指针转换组成：

    #define TCP_SKB_CB(__skb)	((struct tcp_skb_cb *)&((__skb)->cb[0]))

这是一个示例，其中 TCP 模块在收到分段后填写 cb 结构：

```c
static void tcp_v4_fill_cb(struct sk_buff *skb, const struct iphdr *iph, const struct tcphdr *th) {
    TCP_SKB_CB(skb)->seq = ntohl(th->seq);
    TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin + skb->len - th->doff * 4);
    TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);
    TCP_SKB_CB(skb)->tcp_flags = tcp_flag_byte(th);
    TCP_SKB_CB(skb)->tcp_tw_isn = 0;
    TCP_SKB_CB(skb)->ip_dsfield = ipv4_get_dsfield(iph);
    TCP_SKB_CB(skb)->sacked	 = 0;
}
```

### unsigned int csum
### unsigned char ip_summed

上面两个字段代表校验和和状态相关的标志。

### unsigned char cloned
一个布尔标志，设置后表示此结构是另一个sk_buff缓冲区的克隆。

### unsigned char pkt_type
该字段根据 L2 目的地址对帧的类型进行分类。可能的值在 include/linux/if_packet.h 中列出。对于以太网设备，此参数由函数 eth_type_trans 初始化。

主要的一些值：<br>
* PACKET_HOST：接收到的帧的目的地址就是当前接收接口。也就是说，数据包已到达其目的地
* PACKET_MULTICAST：接收到的帧的目标地址是当前接收接口已注册过的的多播地址之一
* PACKET_BROADCAST：接收帧的目的地址是接收接口的广播地址
* PACKET_OTHERHOST：接收帧的目的地址不属于与接口关联的目的地址（单播，组播和广播）；因此，如果启用了转发，则必须转发该帧，否则将其丢弃
* PACKET_OUTGOING：表示数据包正在发送
* PACKET_LOOPBACK：数据包被发送到回环设备。多亏了此标志，内核在处理回环设备时，可以跳过某些实际设备所需的操作
* PACKET_FASTROUTE：数据包正在使用快速路由功能进行路由

### unsigned short protocol
从 L2 处的网卡设备驱动程序的角度来看，这是在更高层次上使用的协议。典型协议有 IP，IPv6和ARP。完整列表可在 include/linux/if_ether.h 中找到。由于每个协议都有其自己的处理传入数据包的处理函数，因此驱动程序使用此字段来通知其上一层使用什么处理函数。每个驱动程序都调用 netif_rx 来调用上层网络层的处理函数，因此必须在调用该函数之前初始化协议字段。

### unsigned short security
表示数据包的安全级别。 该字段最初是为与 IPsec 一起使用而引入的。

## Feature-Specific Fields

Linux内核是模块化的，允许你选择要包括的内容和要忽略的内容。因此，只有在编译内核时开启支持像 Netfilter 或 QoS 之类的特定功能的情况下，某些字段才会包含在 sk_buff 数据结构中：

unsigned long nfmark<br>
__u32 nfcache<br>
__u32 nfctinfo<br>
struct nf_conntrack *nfct unsigned int nfdebug<br>
struct nf_bridge_info *nf_bridge<br>

这些参数由Netfilter使用。

union {…} private

高性能并行接口HIPPI使用此共用体。

__u32 tc_index<br>
__u32 tc_verd<br>
__u32 tc_classid<br>

这些参数由流量控制使用。

## Management Functions

内核提供了许多很简短的简单函数来操纵 sk_buff 节点或链表。

如果查看文件 include/linux/skbuff.h 和 net/core/skbuff.c，你会发现几乎所有功能都有两个版本，名称分别为 do_ something 和 __do_something。通常，第一个是包装函数，它在对第二个调用的调用周围添加了额外的健全性检查或锁定机制。内部 __do_something 函数通常不直接调用。该规则的例外通常是编码不良的函数，这些函数最终将被修复。

下图为分别对 sk_buff 执行 skb_put(a)，skb_push(b)，skb_pull(c)，skb_reserve(d) 的前后对比：

![image](https://user-images.githubusercontent.com/87457873/127462588-5dd5787d-2bc4-4482-99fb-a0a0dc27d0a0.png)

* skb_put：在数据域尾部追加一段空间
* skb_push：在数据域的头部追加一段空间
* skb_pull：将 skb->data 指针在数据域下移指定字节
* skb_reserve：在 sk_buff 中 skb->data 之前的空间追加一段空间（在每层追加自己的协议头时常用到）

## 内存分配

* alloc_skb
* dev_alloc_skb

alloc_skb 是分配缓冲区的主要函数，在 net/core/skbuff.c 中定义。由于数据缓冲区（由 sk_buff 的 head end data tail 指针维护的内存区域）和链表（sk_buff 自身）是两个不同的结构，所以创建单个缓冲区涉及两个内存分配（一个分配给数据缓冲区，另一个分配给 sk_buff 自身结构）。

alloc_skb 通过调用函数 kmem_cache_alloc 从缓存中获取 sk_buff 数据结构，并通过调用 kmalloc 获取数据缓冲区，而 kmalloc 也会使用缓存的内存（如果可用）：

    struct sk_buff *skb;
    u8 *data

    skb = kmem_cache_alloc_node(cache, gfp_mask & ~__GFP_DMA, node);
    size = SKB_DATA_ALIGN(size);
    size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
    data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);

在调用 kmalloc 之前，使用宏 SKB_DATA_ALIGN 调整了大小参数以强制对齐。返回之前，该函数将初始化结构体中的一些参数，从而产生下图所示的最终结果：

![image](https://user-images.githubusercontent.com/87457873/127462806-b8830459-e68c-4b61-bd96-66427bdbbda2.png)

在图右侧存储块的底部，可以看到为了强制对齐而引入的 Padding 区域。 skb_shared_info 块主要用于处理 IP 的分片（IP协议根据 MTU 和 MSS 对数据包进行的分片传输）。

### dev_alloc_skb
dev_alloc_skb 是供设备驱动程序使用的缓冲区分配函数。这类驱动程序预计将在中断模式下被执行。它就是简单的包装了下 alloc_skb, 相比 alloc_skb 多分配了一些字节的空间，并请求了原子操作（GFP_ATOMIC），因为设备驱动程序将在中断处理程序中调用。

## 内存释放

* kfree_skb
* dev_kfree_skb

这两个函数释放一个缓冲区，会让释放的缓冲区返回到缓冲池（缓存）中。dev_kfree_skb 只是通过宏定义的一个标示，内部还是调用的 kfree_skb。

kfree_skb 仅在 skb->users 计数器为1时（没有缓冲区的用户时）才释放缓冲区。 否则，该函数只会使该计数器递减。因此，如果一个缓冲区有三个用户，则只有当调用第三次 dev_kfree_skb 或 kfree_skb 时才会真正释放内存。

![image](https://user-images.githubusercontent.com/87457873/127462960-6fcf0432-7420-4ac2-84ca-008f8faec543.png)

## 数据保留和对齐

* skb_reserve
* skb_put
* skb_push
* skb_pull
* 
skb_reserve 在缓冲区的头部保留一些空间，通常用于允许插入协议头或强制将数据在某个边界上对齐。它通过移动标记数据域开始和结束的 data 和 tail 指针来完成操作。

查看以太网网卡驱动程序的代码（比如: drivers/net/ethernet/3com/3c59x.c vortex_rx 函数），你能看到它们在将任何数据存储在他们刚刚分配的缓冲区中之前都会使用以下命令：

    skb_reserve(skb, 2);	/* Align IP on 16 byte boundaries */
    
因为他们知道他们将要把协议头为 14 个字节的以太网帧复制到缓冲区中，所以参数2将缓冲区的 head 指针下移了 2 个字节。这将让紧跟在以太网头之后的 IP 头，从缓冲区的开头在 16 字节边界上对齐。

![image](https://user-images.githubusercontent.com/87457873/127463077-d110085e-493a-40d2-9130-af014e7c1acb.png)

下图展示了 skb_reserve 在数据从上到下传递（发送数据）时的作用（为下层协议在数据区的头部分配空间）：

![image](https://user-images.githubusercontent.com/87457873/127463098-7f6f9bf7-64aa-49e6-8c19-b8d68891d9ba.png)

1、当要求 TCP 传输某些数据时，它会按照某些条件（TCP Max Segment Size(mss)，对分散收集 I/O 支持等）分配一个缓冲区。<br>
2、TCP 在缓冲区的头部保留（通过调用 skb_reserve）足够的空间，以容纳所有层（TCP，IP，Link 层）的所有协议头。参数 MAX_TCP_HEADER 是所有级别的所有协议头的总和，并考虑到最坏的情况：因为 TCP 层不知道将使用哪种类型的接口进行传输，因此它为每个层保留最大的标头。它甚至考虑到多个 IP 协议头的可能性（因为当内核编译为支持 IP in IP 时，你可以拥有多个IP 协议头）。<br>
3、TCP 的 payload （应用层传输的数据）被复制到缓冲区中。请注意上图只是个例子。TCP 的 payload 可以被不同地组织；例如，可以将其存储为片段。<br>
4、TCP 层添加它的协议头。<br>
5、TCP 层将缓冲区移交给 IP 层，IP层也添加协议头。<br>
6、IP 层将缓冲区移交给下一层，下一层也添加它的协议头。<br>

请注意，当缓冲区在网络栈中向下移动时，每个协议会将 skb->data 指针向下移动，在其协议头中复制，并更新 skb->len。

注意，skb_reserve 函数实际上并没有将任何内容移入或移出数据缓冲区。 它只是更新两个指针：

```c
static inline void skb_reserve(struct sk_buff *skb, int len)
{
	skb->data += len;
	skb->tail += len;
}

```

> 感觉这里是用空间来换了时间，在一开始就分配需要用到的全部空间，然后就可以通过只移动指针来提高效率了。

skb_push 将一个数据块添加到缓冲区的开头，而 skb_put 将一个数据块添加到末尾。像 skb_reserve 一样，这些函数实际上并不会向缓冲区添加任何数据。他们只是将指针移到它的头或尾。数据填充应该由其他功能显式操作。skb_pull 通过将 head 指针向前移动来从缓冲区的头中删除数据块。

### skb_shared_info 结构体 & skb_shinfo 函数

在上面网卡驱动拷贝帧到缓冲区的例子中出现过 skb_shared_info。它是用来保留与数据域有关的其他信息。这个数据结构紧跟在标记数据域结束的 end 指针后面。

```c
struct skb_shared_info {
    atomic_t        dataref;
    __u8            nr_frags;
    struct sk_buff	*frag_list;
    skb_frag_t	    frags[MAX_SKB_FRAGS];
    //...
};
```
* dataref：代表数据域的「用户」数（数据域被引用的次数）
* nr_frags, frag_list, frags：用于处理 IP 片段

skb_is_nonlinear 函数可用于检查缓冲区是否已分段，而 skb_linearize 函数可用于将多个片段合为单个缓冲区。

sk_buff 中没有专门的指针指向 skb_shared_info 区域，skb_shinfo 函数就是方便得到指向 skb_shared_info 区域指针的函数:

```c
#define skb_shinfo(SKB)	((struct skb_shared_info *)(skb_end_pointer(SKB)))

static inline unsigned char *skb_end_pointer(const struct sk_buff *skb)
{
	return skb->end;
}
```

### 克隆和拷贝

当相同的缓冲区需要由不同的消费者处理，并且他们可能更改 sk_buff 结构中的内容（协议头的h和 nh 指针等 Layout 字段）时，为了提高效率，内核并没有克隆缓冲区的结构和数据域，而是仅复制 sk_buff 的结构，并使用引用计数进行操作，以避免过早释放共享数据块。skb_clone 函数负责拷贝一个 buffer。

使用克隆的一种情况是，需要将入口数据包分发给多个接收者，例如协议处理程序和一个或多个网络分接头（Network taps）。

sk_buff 克隆不会链接到任何链表，也没有引用套接字所有者。克隆和原始缓冲区中的 skb->cloned 字段均设置为1。在克隆中将 skb->users 设置为1，以便第一次尝试删除它（被克隆的 sk_buff）时会成功，并且数据域的引用数（dataref）递增（因为现在有一个新的 sk_buff 指向了）。

skb_clone 会调用 __skb_clone:

```c
static struct sk_buff *__skb_clone(struct sk_buff *n, struct sk_buff *skb) 
{
#define C(x) n->x = skb->x 
// 定义的宏，如果 x 是普通变量则是值赋值
// 如果 x 是指针，则是指向同一块区域

	n->next = n->prev = NULL;
	n->sk = NULL;
	__copy_skb_header(n, skb);
    //...
	n->destructor = NULL;
	C(tail);
	C(end);
	C(head);
	C(head_frag);
	C(data);    // data 是一个指针, 所以没有克隆数据域，只是指向了数据域的内存地址
	C(truesize);
	refcount_set(&n->users, 1); //设置克隆的 sk_buff 的用户数为1
	atomic_inc(&(skb_shinfo(skb)->dataref)); //增加数据域的引用次数
	skb->cloned = 1;
	return n;
#undef C
}
```

下图为一个被分段（一个缓冲区，其中一些数据存储在与frags数组链接的数据片段中）了的缓冲区克隆的例子:

![image](https://user-images.githubusercontent.com/87457873/127463500-7ce7c75f-47ca-46f0-beba-68d57ffb9112.png)

当缓冲区被克隆时，无法修改数据块的内容。这意味着代码无需做同步保证即可访问数据。但是，当一个函数不仅需要修改 sk_buff 结构的内容，还需要修改数据域时，就必须要克隆数据域了。如果真要修改数据域，开发者也有两个选项可用。

1、当开发者知道自己仅仅需要修改的数据在 skb->start 和 skb->end 的区域时，开发者可以使用 pskb_copy 方法只克隆那个区域。<br>
2、当开发者认为自己或许也需要修改分段数据域时，就必须使用 skb_copy。<br>

pskb_copy 和 skb_copy 的不同如下图中的(a)和(b):

![image](https://user-images.githubusercontent.com/87457873/127463640-4c822c4a-b66e-4e0c-9dec-37fd52aff195.png)

在决定克隆或复制缓冲区时，每个子系统的程序员都无法预料其他内核组件（或其子系统的其他用户）是否需要该缓冲区中的原始信息。内核是非常模块化的，并且以非常动态和不可预测的方式进行更改，因此每个子系统都不知道其他子系统可以使用缓冲区做什么。因此，每个子系统的程序员只需跟踪他们对缓冲区所做的任何修改，并注意在修改任何内容之前先进行复制，以防内核的其他部分需要原始信息。

## 队列管理函数

有一些函数用来维护 sk_buff 双向链表（也可以称为队列 queue）中的节点。下面是一些常用的功能函数：

### skb_queue_head_init
使用空节点初始化 sk_buff_head。

### skb_queue_head, skb_queue_tail
将一个缓冲区添加到队列的头部或尾部。

### skb_dequeue, skb_dequeue_tail
从队列的头和尾取出一个节点。

### skb_queue_purge
清空队列。

### skb_queue_walk

使用 for 循环遍历队列，其实现如下：

```c
#define skb_queue_walk(queue, skb) 
       for (skb = (queue)->next;					
            skb != (struct sk_buff *)(queue);				
            skb = skb->next)
```
可以看到其实现是定义了一个宏，预处理编译之后 skb_queue_walk 就会被替换为上面的代码，因为 sk_buff 的队列是一个双向链表，去过遍历到了头节点，说明遍历完成了。

操作队列的所有函数都必须保证是原子操作。也就是说，它们必须获取 sk_buff_head 结构提供的队列自旋锁。否则，它们可能会被异步事件中断，这些异步事件会使队列中的元素入队或出队，例如到期计时器调用的函数会导致争用条件。


