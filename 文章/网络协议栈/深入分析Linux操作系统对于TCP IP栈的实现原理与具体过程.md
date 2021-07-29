## 一、Linux内核与网络体系结构

在我们了解整个linux系统的网络体系结构之前，我们需要对整个网络体系调用，初始化和交互的位置，同时也是Linux操作系统中最为关键的一部分代码-------内核，有一个初步的认知。

### 1、Linux内核的结构

首先，从功能上，我们将linux内核划分为五个不同的部分，分别是

（1）进程管理：主要负载CPU的访问控制，对CPU进行调度管理；<br>
（2）内存管理：主要提供对内存资源的访问控制；<br>
（3）文件系统：将硬盘的扇区组织成文件系统，实现文件读写等操作；<br>
（4）设备管理：用于控制所有的外部设备及控制器；<br>
（5）网洛：主要负责管理各种网络设备，并实现各种网络协议栈，最终实现通过网络连接其它系统的功能；<br>

每个部分分别处理一项明确的功能，又向其它各个部分提供自己所完成的功能，相互协调，共同完成操作系统的任务。

Linux内核架构如下图所示：

![image](https://user-images.githubusercontent.com/87457873/127457010-628a0bb6-0541-4ea4-966e-9e0c6b2873ca.png)

### 2、Linux网络子系统

内核的基本架构我们已经了解清楚了，接下来我们重点关注到内核中的网络模块，观察在linux内核中，我们是如何实现及运用TCP/IP协议，并完成网络的初始化及各个模块调用调度。我们将内核中的网络部分抽出，通过对比TCP/IP分层协议，与Linux网络实现体系相对比，深入的了解学习linux内核是怎样具体的实现TCP/IP协议栈的。

Linux网络体系与TCP/IP协议栈如下图所示。       

![image](https://user-images.githubusercontent.com/87457873/127457083-f9239de8-6358-4909-9fc4-605a81024009.png)

![image](https://user-images.githubusercontent.com/87457873/127457095-b19e6c20-5843-47e4-938f-2a6785aa83ff.png)

可以看到，在图中，linux为了抽象与实现相分离，将内核中的网络部分划分为五层：

* 系统调用接口：系统调用接口是用户空间的应用程序正常访问内核的唯一途径，系统调用一般以sys开头。
* 协议无关接口：协议无关接口是由socket来实现的，它提供一组通用函数来支持各种不同的协议。Linux中socket结构是struct sock，这个结构定义了socket所需要的所  有状态信息，包括socke所使用的协议以及可以在socket上执行的操作。
* 网络协议：Linux支持多种协议，每一个协议都对应net_family[]数组中的一项，net_family[]的元素为一个结构体指针，指向一个包含注册协议信息的结构体                      net_proto_family;
* 设备无关接口：设备无关接口net_device实现的，任何设备与上层通信都是通过net_device设备无关接口。它将设备与具有很多功能的不同硬件连接在一起，这一层提供一组通用函数供底层网络设备驱动程序使用，让它们可以对高层协议栈进行操作。
* 设备驱动程序：网络体系结构的最底部是负责管理物理网络设备的设备驱动程序层。

Linux网络子系统通过这五层结构的相互交互，共同完成TCP/IP协议栈的运行。

### 3、TCP/IP协议栈

#### 3.1 网络架构

Linux网络协议栈的架构如下图所示。该图展示了如何实现Internet模型,在最上面的是用户空间中实现的应用层，而中间为内核空间中实现的网络子系统，底部为物理设备，提供了对网络的连接能力。在网络协议栈内部流动的是套接口缓冲区(SKB)，用于在协议栈的底层、上层以及应用层之间传递报文数据。

网络协议栈顶部是系统调用接口，为用户空间中的应用程序提供一种访问内核网络子系统的接口。下面是一个协议无关层，它提供了一种通用方法来使用传输层协议。然后是传输层的具体协议，包括TCP、UDP。在传输层下面是网络层，之后是邻居子系统，再下面是网络设备接口，提供了与各个设备驱动通信的通用接口。最底层是设备驱动程序。                  

![image](https://user-images.githubusercontent.com/87457873/127457257-693815e3-3443-4d21-a79f-06d93234303f.png)

#### 3.2 协议无关接口

通过网络协议栈通信需要对套接口进行操作，套接口是一个与协议无关的接口，它提供了一组接口来支持各种协议，套接口层不但可以支持典型的TCP和UDP协议，还可以支持RAW套接口、RAW以太网以及其他传输协议。

Linux中使用socket结构描述套接口，代表一条通信链路的一端，用来存储与该链路有关的所有信息，这些信息中包括：

* 所使用的协议
* 协议的状态信息(包括源地址和目标地址)
* 到达的连接队列
* 数据缓存和可选标志等等

其示意图如下所示：

![image](https://user-images.githubusercontent.com/87457873/127457377-6d4fe3e9-139c-4b43-9e84-c8b9dd2e92e2.png)

其中最关键的成员是sk和ops，sk指向与该套接口相关的传输控制块，ops指向特定的传输协议的操作集。

下图详细展示了socket结构体中的sk和ops字段，以TCP为例。

![image](https://user-images.githubusercontent.com/87457873/127457404-384b4a7b-6f26-4cef-a345-c5b6c673e400.png)

sk字段指向与该套接口相关的传输控制块，传输层使用传输控制块来存放套接口所需的信息，在上图中即为TCP传输控制块，即tcp_sock结构。

ops字段指向特定传输协议的操作集接口，proto_pos结构中定义的接口函数是从套接口系统调用到传输层调用的入口，因此其成员与socket系统调用基本上是一一对应的。整个proto_ops结构就是一张套接口系统调用的跳转表，TCP、UDP、RAW套接口的传输层操作集分别为inet_stream_ops、inet_dgram_ops、inet_sockraw_ops。

#### 3.3 套接口缓存

如下图所示，网络子系统中用来存储数据的缓冲区叫做套接口缓存，简称为SKB，该缓存区能够处理可变长数据，即能够很容易地在数据区头尾部添加和移除数据，且尽量避免数据的复制，通常每个报文使用一个SKB表示，各协议栈报文头通过一组指针进行定位，由于SKB是网络子系统中数据管理的核心，因此有很多管理函数是对它进行操作的。

SKB主要用于在网络驱动程序和应用程序之间传递、复制数据包。当应用程序要发送一个数据包时：

* 数据通过系统调用提交到内核中
* 系统会分配一个SKB来存储数据
* 之后向下层传递
* 再传递给网络驱动后才将其释放

当网络设备接收到数据包也要分配一个SKB来对数据进行存储，之后再向上传递，最终将数据复制到应用程序后进行释放。

![image](https://user-images.githubusercontent.com/87457873/127457515-485fa24f-613c-4c78-b661-2e02a47968d8.png)

#### 3.4 重要的数据结构

##### 3.4.1 sk_buf

sk_buf是Linux网络协议栈最重要的数据结构之一，该数据结构贯穿于整个数据包处理的流程。由于协议采用分层结构，上层向下层传递数据时需要增加包头，下层向上层数据时又需要去掉包头。sk_buff中保存了L2，L3，L4层的头指针，这样在层传递时只需要对数据缓冲区改变头部信息，并调整sk_buff中的指针，而不需要拷贝数据，这样大大减少了内存拷贝的需要。

sk_buf的示意图如下：

![image](https://user-images.githubusercontent.com/87457873/127457634-36a897d5-c130-4581-bc9b-96ae226e4feb.png)

各字段含义如下：

* head：指向分配给的线性数据内存首地址。
* data：指向保存数据内容的首地址。
* tail：指向数据的结尾。 
* end：指向分配的内存块的结尾。
* len：数据的长度。
* head room: 位于head至data之间的空间，用于存储：protocol header，例如：TCP header, IP header, Ethernet header等。
* user data: 位于data至tail之间的空间，用于存储：应用层数据，一般系统调用时会使用到。 
* tail room: 位于tail至end之间的空间，用于填充用户数据未使用完的空间。
* skb_shared_info: 位于end之后，用于存储特殊数据结构skb_shared_info，该结构用于描述分片信息。 

sk_buf的常用操作函数如下：

* alloc_skb：分配sk_buf。
* skb_reserve：为sk_buff设置header空间。
* skb_put：添加用户层数据。
* skb_push：向header空间添加协议头。
* skb_pull：复位data至数据区。

操作sk_buf的简单示意图如下：

![image](https://user-images.githubusercontent.com/87457873/127457802-b2293277-90b3-4135-9f70-4f03675fba53.png)

##### 3.4.2 net_device

在网络适配器硬件和软件协议栈之间需要一个接口，共同完成操作系统内核中协议栈数据处理与异步收发的功能。在Linux网络体系结构中，这个接口要满足以下要求:

（1）抽象出网络适配器的硬件特性。<br>
（2）为协议栈提供统一的调用接口。<br>

以上两个要求在Linux内核的网络体系结构中分别由两个软件（设备独立接口文件dev.c和网络设备驱动程序）和一个主要的数据结构net_device实现。

设备独立接口文件dev.c中实现了对上层协议的统一调用接口，dev.c文件中的函数实现了以下主要功能。

* 协议调用与驱动程序函数对应：dev.c文件中的函数查看数据包由哪个网络设备(由sk_buff结构中*dev数据域指明该数据包由哪个网络设备net_device实例接收/发送)传送，根据系统中注册的设备实例,调用网络设备驱动程序函数，实现硬件的收发。
* 对net_device数据结构的数据域统一初始化：dev.c提供了一些常规函数，来初始化net_device结构中的这样一些数据域:它们的值对所有类型的设备都一样，驱动程序可以调用这些函数来设置其设备实例的默认值，也可以重写由内核初始化的值。

每一个网络设备都必须有一个驱动程序，并提供一个初始化函数供内核启动时调用，或在装载网络驱动程序模块时调用。不管网络设备内部有什么不同，有一件事是所有网络设备驱动程序必须首先完成的任务:初始化一个net_device数据结构的实例作为网络设备在内核中的实体，并将net_device数据结构实例的各数据域初始化为可工作的状态，然后将设备实例注册到内核中，为协议栈提供传送服务。

net_device数据结构从以下两个方面描述了网络设备的硬件特性在内核中的表示。

* 描述设备属性

net_device数据结构实例是网络设备在内核中的表示，它是每个网络设备在内核中的基础数据结构，它包含的信息不仅仅是网络设备的硬件属性（中断、端口地址、驱动程序函数等)，还包括网络中与设备有关的上层协议栈的配置信息（如IP地址、子网掩码等)。它跟踪连接到 TCP/IP协议栈上的所有设备的状态信息。

* 实现设备驱动程序接口

net_device数据结构代表了上层的网络协议和硬件之间的一个通用接口，使我们可以将网络协议层的实现从具体的网络硬件部件中抽象出来，独立于硬件设备。为了有效地实现这种抽象，net_device中使用了大量函数指针，这样相对于上层的协议栈，它们在做数据收发操作时调用的函数的名字是相同的，但具体的函数实现细节可以根据不同的网络适配器而不同，由设备驱动程序提供，对网络协议栈透明。

![image](https://user-images.githubusercontent.com/87457873/127458062-fc93e822-b1d7-43c5-9f2e-213b66168293.png)

##### 3.4.3 socket

内核中的进程可以通过socket结构体来访问linux内核中的网络系统中的传输层、网络层以及数据链路层，也可以说socket是内核中的进程与内核中的网络系统的桥梁。

我们知道在TCP层中使用两个协议：tcp协议和udp协议。而在将TCP层中的数据往下传输时，要使用网络层的协议，而网络层的协议很多，不同的网络使用不同的网络层协议。我们常用的因特网中，网络层使用的是IPV4和IPV6协议。所以在内核中的进程在使用struct socket提取内核网络系统中的数据时，不光要指明struct socket的类型(用于说明是提取TCP层中tcp协议负载的数据，还是udp层负载的数据)，还要指明网络层的协议类型(网络层的协议用于负载TCP层中的数据)。

linux内核中的网络系统中的网络层的协议，在linux中被称为address family(地址簇，通常以AF_XXX表示）或protocol family(协议簇，通常以PF_XXX表示)。

## 二、网络信息处理流程

### 1、硬中断处理

首先当数据帧从网线到达网卡上的时候，第一站是网卡的接收队列。网卡在分配给自己的RingBuffer中寻找可用的内存位置，找到后DMA引擎会把数据DMA到网卡之前关联的内存里，这个时候CPU都是无感的。当DMA操作完成以后，网卡会像CPU发起一个硬中断，通知CPU有数据到达。

![image](https://user-images.githubusercontent.com/87457873/127458176-e1d436d9-5ddf-4e59-9ef8-f23ac16a725e.png)

注意，当RingBuffer满的时候，新来的数据包将给丢弃。ifconfig查看网卡的时候，可以里面有个overruns，表示因为环形队列满被丢弃的包。如果发现有丢包，可能需要通过ethtool命令来加大环形队列的长度。

网卡的硬中断注册的处理函数是igb_msix_ring。

```c
//file: drivers/net/ethernet/intel/igb/igb_main.c
static irqreturn_t igb_msix_ring(int irq, void *data)
{
    struct igb_q_vector *q_vector = data;

    /* Write the ITR value calculated from the previous interrupt. */
    igb_write_itr(q_vector);

    napi_schedule(&q_vector->napi);

    return IRQ_HANDLED;
}
```

igb_write_itr只是记录一下硬件中断频率（据说目的是在减少对CPU的中断频率时用到）。顺着napi_schedule调用一路跟踪下去，__napi_schedule=>____napi_schedule。

```c
/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
                     struct napi_struct *napi)
{
    list_add_tail(&napi->poll_list, &sd->poll_list);
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```

这里我们看到，list_add_tail修改了CPU变量softnet_data里的poll_list，将驱动napi_struct传过来的poll_list添加了进来。 其中softnet_data中的poll_list是一个双向列表，其中的设备都带有输入帧等着被处理。紧接着__raise_softirq_irqoff触发了一个软中断NET_RX_SOFTIRQ， 这个所谓的触发过程只是对一个变量进行了一次或运算而已。

```c
void __raise_softirq_irqoff(unsigned int nr)
{
    trace_softirq_raise(nr);
    or_softirq_pending(1UL << nr);
}
//file: include/linux/irq_cpustat.h
#define or_softirq_pending(x)  (local_softirq_pending() |= (x))
```

Linux在硬中断里只完成简单必要的工作，剩下的大部分的处理都是转交给软中断的。通过上面代码可以看到，硬中断处理过程真的是非常短。只是记录了一个寄存器，修改了一下下CPU的poll_list，然后发出个软中断。就这么简单，硬中断工作就算是完成了。

### 2、ksoftirqd内核线程处理软中断

![image](https://user-images.githubusercontent.com/87457873/127458460-dede7009-1ba5-427e-98e2-345f798d6724.png)

ksoftirqd_should_run代码如下：
```c
static int ksoftirqd_should_run(unsigned int cpu)
{
    return local_softirq_pending();
}

#define local_softirq_pending() \
    __IRQ_STAT(smp_processor_id(), __softirq_pending)
这里看到和硬中断中调用了同一个函数local_softirq_pending。使用方式不同的是硬中断位置是为了写入标记，这里仅仅只是读取。如果硬中断中设置了NET_RX_SOFTIRQ,这里自然能读取的到。接下来会真正进入线程函数中run_ksoftirqd处理：
static void run_ksoftirqd(unsigned int cpu)
{
    local_irq_disable();
    if (local_softirq_pending()) {
        __do_softirq();
        rcu_note_context_switch(cpu);
        local_irq_enable();
        cond_resched();
        return;
    }
    local_irq_enable();
}
```

在__do_softirq中，判断根据当前CPU的软中断类型，调用其注册的action方法。

```c
asmlinkage void __do_softirq(void)
{
    do {
        if (pending & 1) {
            unsigned int vec_nr = h - softirq_vec;
            int prev_count = preempt_count();

            ...
            trace_softirq_entry(vec_nr);
            h->action(h);
            trace_softirq_exit(vec_nr);
            ...
        }
        h++;
        pending >>= 1;
    } while (pending);
}　　

```

在网络子系统初始化小节， 我们看到我们为NET_RX_SOFTIRQ注册了处理函数net_rx_action。所以net_rx_action函数就会被执行到了。

这里需要注意一个细节，硬中断中设置软中断标记，和ksoftirq的判断是否有软中断到达，都是基于smp_processor_id()的。这意味着只要硬中断在哪个CPU上被响应，那么软中断也是在这个CPU上处理的。所以说，如果你发现你的Linux软中断CPU消耗都集中在一个核上的话，做法是要把调整硬中断的CPU亲和性，来将硬中断打散到不通的CPU核上去。

我们再来把精力集中到这个核心函数net_rx_action上来。

```c
static void net_rx_action(struct softirq_action *h)
{
    struct softnet_data *sd = &__get_cpu_var(softnet_data);
    unsigned long time_limit = jiffies + 2;
    int budget = netdev_budget;
    void *have;

    local_irq_disable();

    while (!list_empty(&sd->poll_list)) {
        ......
        n = list_first_entry(&sd->poll_list, struct napi_struct, poll_list);

        work = 0;
        if (test_bit(NAPI_STATE_SCHED, &n->state)) {
            work = n->poll(n, weight);
            trace_napi_poll(n);
        }

        budget -= work;
    }
}
```

函数开头的time_limit和budget是用来控制net_rx_action函数主动退出的，目的是保证网络包的接收不霸占CPU不放。 等下次网卡再有硬中断过来的时候再处理剩下的接收数据包。其中budget可以通过内核参数调整。 这个函数中剩下的核心逻辑是获取到当前CPU变量softnet_data，对其poll_list进行遍历, 然后执行到网卡驱动注册到的poll函数。对于igb网卡来说，就是igb驱动力的igb_poll函数了。

```c
/**
 *  igb_poll - NAPI Rx polling callback
 *  @napi: napi polling structure
 *  @budget: count of how many packets we should handle
 **/
static int igb_poll(struct napi_struct *napi, int budget)
{
    ...
    if (q_vector->tx.ring)
        clean_complete = igb_clean_tx_irq(q_vector);

    if (q_vector->rx.ring)
        clean_complete &= igb_clean_rx_irq(q_vector, budget);
    ...
}
```
在读取操作中，igb_poll的重点工作是对igb_clean_rx_irq的调用。

```c
static bool igb_clean_rx_irq(struct igb_q_vector *q_vector, const int budget)
{
    ...

    do {

        /* retrieve a buffer from the ring */
        skb = igb_fetch_rx_buffer(rx_ring, rx_desc, skb);

        /* fetch next buffer in frame if non-eop */
        if (igb_is_non_eop(rx_ring, rx_desc))
            continue;
        }

        /* verify the packet layout is correct */
        if (igb_cleanup_headers(rx_ring, rx_desc, skb)) {
            skb = NULL;
            continue;
        }

        /* populate checksum, timestamp, VLAN, and protocol */
        igb_process_skb_fields(rx_ring, rx_desc, skb);

        napi_gro_receive(&q_vector->napi, skb);
}
igb_fetch_rx_buffer和igb_is_non_eop的作用就是把数据帧从RingBuffer上取下来。为什么需要两个函数呢？因为有可能帧要占多多个RingBuffer，所以是在一个循环中获取的，直到帧尾部。获取下来的一个数据帧用一个sk_buff来表示。收取完数据以后，对其进行一些校验，然后开始设置sbk变量的timestamp, VLAN id, protocol等字段。接下来进入到napi_gro_receive中:
//file: net/core/dev.c
gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
{
    skb_gro_reset_offset(skb);

    return napi_skb_finish(dev_gro_receive(napi, skb), skb);
}
dev_gro_receive这个函数代表的是网卡GRO特性，可以简单理解成把相关的小包合并成一个大包就行，目的是减少传送给网络栈的包数，这有助于减少 CPU 的使用量。我们暂且忽略，直接看napi_skb_finish, 这个函数主要就是调用了netif_receive_skb。
//file: net/core/dev.c
static gro_result_t napi_skb_finish(gro_result_t ret, struct sk_buff *skb)
{
    switch (ret) {
    case GRO_NORMAL:
        if (netif_receive_skb(skb))
            ret = GRO_DROP;
        break;
    ......
}
```

在netif_receive_skb中，数据包将被送到协议栈中。声明，以下的3.3, 3.4, 3.5也都属于软中断的处理过程，只不过由于篇幅太长，单独拿出来成小节。

### 3、网络协议栈处理

netif_receive_skb函数会根据包的协议，假如是udp包，会将包依次送到ip_rcv(),udp_rcv()协议处理函数中进行处理。

![image](https://user-images.githubusercontent.com/87457873/127458725-e83ce43a-9dc2-4794-b828-e9f7bac3b168.png)

```c
//file: net/core/dev.c
int netif_receive_skb(struct sk_buff *skb)
{
    //RPS处理逻辑，先忽略
    ......

    return __netif_receive_skb(skb);
}

static int __netif_receive_skb(struct sk_buff *skb)
{
    ......   
    ret = __netif_receive_skb_core(skb, false);
}

static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
    ......

    //pcap逻辑，这里会将数据送入抓包点。tcpdump就是从这个入口获取包的
    list_for_each_entry_rcu(ptype, &ptype_all, list) {
        if (!ptype->dev || ptype->dev == skb->dev) {
            if (pt_prev)
                ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = ptype;
        }
    }

    ......

    list_for_each_entry_rcu(ptype,
            &ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
        if (ptype->type == type &&
            (ptype->dev == null_or_dev || ptype->dev == skb->dev ||
             ptype->dev == orig_dev)) {
            if (pt_prev)
                ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = ptype;
        }
    }
}
```

在__netif_receive_skb_core中，我看着原来经常使用的tcpdump的抓包点，很是激动，看来读一遍源代码时间真的没白浪费。接着__netif_receive_skb_core取出protocol，它会从数据包中取出协议信息，然后遍历注册在这个协议上的回调函数列表。ptype_base 是一个 hash table，在协议注册小节我们提到过。ip_rcv 函数地址就是存在这个 hash table中的。

```c
//file: net/core/dev.c
static inline int deliver_skb(struct sk_buff *skb,
                  struct packet_type *pt_prev,
                  struct net_device *orig_dev)
{
    ......
    return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

pt_prev->func这一行就调用到了协议层注册的处理函数了。对于ip包来讲，就会进入到ip_rcv（如果是arp包的话，会进入到arp_rcv）。

### 4、IP协议层处理

我们再来大致看一下linux在ip协议层都做了什么，包又是怎么样进一步被送到udp或tcp协议处理函数中的。

```c
//file: net/ipv4/ip_input.c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    ......

    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
               ip_rcv_finish);
}
```

这里NF_HOOK是一个钩子函数，当执行完注册的钩子后就会执行到最后一个参数指向的函数ip_rcv_finish。

```c
static int ip_rcv_finish(struct sk_buff *skb)
{
    ......

    if (!skb_dst(skb)) {
        int err = ip_route_input_noref(skb, iph->daddr, iph->saddr,
                           iph->tos, skb->dev);
        ...
    }

    ......

    return dst_input(skb);
}
```

跟踪ip_route_input_noref 后看到它又调用了 ip_route_input_mc。 在ip_route_input_mc中，函数ip_local_deliver被赋值给了dst.input, 如下：

```c
//file: net/ipv4/route.c
static int ip_route_input_mc(struct sk_buff *skb, __be32 daddr, __be32 saddr,
                u8 tos, struct net_device *dev, int our)
{
    if (our) {
        rth->dst.input= ip_local_deliver;
        rth->rt_flags |= RTCF_LOCAL;
    }
}
```

所以回到ip_rcv_finish中的return dst_input(skb)。

```c
/* Input packet from network to transport.  */
static inline int dst_input(struct sk_buff *skb)
{
    return skb_dst(skb)->input(skb);
}
```
skb_dst(skb)->input调用的input方法就是路由子系统赋的ip_local_deliver。

```c
//file: net/ipv4/ip_input.c
int ip_local_deliver(struct sk_buff *skb)
{
    /*
     *  Reassemble IP fragments.
     */

    if (ip_is_fragment(ip_hdr(skb))) {
        if (ip_defrag(skb, IP_DEFRAG_LOCAL_DELIVER))
            return 0;
    }

    return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, skb, skb->dev, NULL,
               ip_local_deliver_finish);
}
static int ip_local_deliver_finish(struct sk_buff *skb)
{
    ......

    int protocol = ip_hdr(skb)->protocol;
    const struct net_protocol *ipprot;

    ipprot = rcu_dereference(inet_protos[protocol]);
    if (ipprot != NULL) {
        ret = ipprot->handler(skb);
    }
}
```
如协议注册小节看到inet_protos中保存着tcp_rcv()和udp_rcv()的函数地址。这里将会根据包中的协议类型选择进行分发,在这里skb包将会进一步被派送到更上层的协议中，udp和tcp。

### 5、UDP协议层处理

udp协议的处理函数是udp_rcv。

```c
//file: net/ipv4/udp.c
int udp_rcv(struct sk_buff *skb)
{
    return __udp4_lib_rcv(skb, &udp_table, IPPROTO_UDP);
}


int __udp4_lib_rcv(struct sk_buff *skb, struct udp_table *udptable,
           int proto)
{
    sk = __udp4_lib_lookup_skb(skb, uh->source, uh->dest, udptable);

    if (sk != NULL) {
        int ret = udp_queue_rcv_skb(sk, skb
    }

    icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);
}
```
__udp4_lib_lookup_skb是根据skb来寻找对应的socket，当找到以后将数据包放到socket的缓存队列里。如果没有找到，则发送一个目标不可达的icmp包。

```c
//file: net/ipv4/udp.c
int udp_queue_rcv_skb(struct sock *sk, struct sk_buff *skb)
{   
    ......

    if (sk_rcvqueues_full(sk, skb, sk->sk_rcvbuf))
        goto drop;

        rc = 0;

    ipv4_pktinfo_prepare(skb);
    bh_lock_sock(sk);
    if (!sock_owned_by_user(sk))
        rc = __udp_queue_rcv_skb(sk, skb);
    else if (sk_add_backlog(sk, skb, sk->sk_rcvbuf)) {
        bh_unlock_sock(sk);
        goto drop;
    }
    bh_unlock_sock(sk);

    return rc;
}
```

sock_owned_by_user判断的是用户是不是正在这个socker上进行系统调用（socket被占用），如果没有，那就可以直接放到socket的接收队列中。如果有，那就通过sk_add_backlog把数据包添加到backlog队列。 当用户释放的socket的时候，内核会检查backlog队列，如果有数据再移动到接收队列中。

sk_rcvqueues_full接收队列如果满了的话，将直接把包丢弃。接收队列大小受内核参数net.core.rmem_max和net.core.rmem_default影响。

## 三、send分析

### 1、传输层分析

send的定义如下所示：
```
ssize_t send(int sockfd, const void *buf, size_t len, int flags)
```

当在调用send函数的时候，内核封装send()为sendto()，然后发起系统调用。其实也很好理解，send()就是sendto()的一种特殊情况，而sendto()在内核的系统调用服务程序为sys_sendto，sys_sendto的代码如下所示：

```c
int __sys_sendto(int fd, void __user *buff, size_t len, unsigned int flags,
         struct sockaddr __user *addr,  int addr_len)
{
    struct socket *sock;
    struct sockaddr_storage address;
    int err;
    struct msghdr msg; //用来表示要发送的数据的一些属性
    struct iovec iov;
    int fput_needed;
    err = import_single_range(WRITE, buff, len, &iov, &msg.msg_iter);
    if (unlikely(err))
        return err;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    msg.msg_name = NULL;
    msg.msg_control = NULL;
    msg.msg_controllen = 0;
    msg.msg_namelen = 0;
    if (addr) {
        err = move_addr_to_kernel(addr, addr_len, &address);
        if (err < 0)
            goto out_put;
        msg.msg_name = (struct sockaddr *)&address;
        msg.msg_namelen = addr_len;
    }
    if (sock->file->f_flags & O_NONBLOCK)
        flags |= MSG_DONTWAIT;
    msg.msg_flags = flags;
    err = sock_sendmsg(sock, &msg); //实际的发送函数

out_put:
    fput_light(sock->file, fput_needed);
out:
    return err;
}
```

__sys_sendto函数其实做了3件事：

* 通过fd获取了对应的struct socket
* 创建了用来描述要发送的数据的结构体struct msghdr
* 调用了sock_sendmsg来执行实际的发送

继续追踪sock_sendmsg，发现其最终调用的是sock->ops->sendmsg(sock, msg, msg_data_left(msg))，即socet在初始化时赋值给结构体struct proto tcp_prot的函数tcp_sendmsg，如下所示：

```c
struct proto tcp_prot = {
    .name            = "TCP",
    .owner            = THIS_MODULE,
    .close            = tcp_close,
    .pre_connect        = tcp_v4_pre_connect,
    .connect        = tcp_v4_connect,
    .disconnect        = tcp_disconnect,
    .accept            = inet_csk_accept,
    .ioctl            = tcp_ioctl,
    .init            = tcp_v4_init_sock,
    .destroy        = tcp_v4_destroy_sock,
    .shutdown        = tcp_shutdown,
    .setsockopt        = tcp_setsockopt,
    .getsockopt        = tcp_getsockopt,
    .keepalive        = tcp_set_keepalive,
    .recvmsg        = tcp_recvmsg,
    .sendmsg        = tcp_sendmsg,
  ...
 ```
而tcp_send函数实际调用的是tcp_sendmsg_locked函数，该函数的定义如下所示：

```c
int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
    struct tcp_sock *tp = tcp_sk(sk);/*进行了强制类型转换*/
    struct sk_buff *skb;
    flags = msg->msg_flags;
    ......
        if (copied)
            tcp_push(sk, flags & ~MSG_MORE, mss_now,
                 TCP_NAGLE_PUSH, size_goal);
}

```
在tcp_sendmsg_locked中，完成的是将所有的数据组织成发送队列，这个发送队列是struct sock结构中的一个域sk_write_queue，这个队列的每一个元素是一个skb，里面存放的就是待发送的数据。在该函数中通过调用tcp_push()函数将数据加入到发送队列中。

sock结构体的部分代码如下所示：
```c
struct sock{
    ...
    struct sk_buff_head    sk_write_queue;/*指向skb队列的第一个元素*/
    ...
    struct sk_buff    *sk_send_head;/*指向队列第一个还没有发送的元素*/
}
```
tcp_push的代码如下所示：
```c
static void tcp_push(struct sock *sk, int flags, int mss_now,
             int nonagle, int size_goal)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;

    skb = tcp_write_queue_tail(sk);
    if (!skb)
        return;
    if (!(flags & MSG_MORE) || forced_push(tp))
        tcp_mark_push(tp, skb);

    tcp_mark_urg(tp, flags);

    if (tcp_should_autocork(sk, skb, size_goal)) {

        /* avoid atomic op if TSQ_THROTTLED bit is already set */
        if (!test_bit(TSQ_THROTTLED, &sk->sk_tsq_flags)) {
            NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPAUTOCORKING);
            set_bit(TSQ_THROTTLED, &sk->sk_tsq_flags);
        }
        /* It is possible TX completion already happened
         * before we set TSQ_THROTTLED.
         */
        if (refcount_read(&sk->sk_wmem_alloc) > skb->truesize)
            return;
    }

    if (flags & MSG_MORE)
        nonagle = TCP_NAGLE_CORK;

    __tcp_push_pending_frames(sk, mss_now, nonagle); //最终通过调用该函数发送数据
}
```
在之后tcp_push调用了__tcp_push_pending_frames(sk, mss_now, nonagle);来发送数据

__tcp_push_pending_frames的代码如下所示：
```c
void __tcp_push_pending_frames(struct sock *sk, unsigned int cur_mss,
                   int nonagle)
{

    if (tcp_write_xmit(sk, cur_mss, nonagle, 0,
               sk_gfp_mask(sk, GFP_ATOMIC))) //调用该函数发送数据
        tcp_check_probe_timer(sk);
}
```
在__tcp_push_pending_frames又调用了tcp_write_xmit来发送数据，代码如下所示：

```c
static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now, int nonagle,
               int push_one, gfp_t gfp)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;
    unsigned int tso_segs, sent_pkts;
    int cwnd_quota;
    int result;
    bool is_cwnd_limited = false, is_rwnd_limited = false;
    u32 max_segs;
    /*统计已发送的报文总数*/
    sent_pkts = 0;
    ......

    /*若发送队列未满，则准备发送报文*/
    while ((skb = tcp_send_head(sk))) {
        unsigned int limit;

        if (unlikely(tp->repair) && tp->repair_queue == TCP_SEND_QUEUE) {
            /* "skb_mstamp_ns" is used as a start point for the retransmit timer */
            skb->skb_mstamp_ns = tp->tcp_wstamp_ns = tp->tcp_clock_cache;
            list_move_tail(&skb->tcp_tsorted_anchor, &tp->tsorted_sent_queue);
            tcp_init_tso_segs(skb, mss_now);
            goto repair; /* Skip network transmission */
        }

        if (tcp_pacing_check(sk))
            break;

        tso_segs = tcp_init_tso_segs(skb, mss_now);
        BUG_ON(!tso_segs);
        /*检查发送窗口的大小*/
        cwnd_quota = tcp_cwnd_test(tp, skb);
        if (!cwnd_quota) {
            if (push_one == 2)
                /* Force out a loss probe pkt. */
                cwnd_quota = 1;
            else
                break;
        }

        if (unlikely(!tcp_snd_wnd_test(tp, skb, mss_now))) {
            is_rwnd_limited = true;
            break;
        ......
        limit = mss_now;
        if (tso_segs > 1 && !tcp_urg_mode(tp))
            limit = tcp_mss_split_point(sk, skb, mss_now,
                            min_t(unsigned int,
                              cwnd_quota,
                              max_segs),
                            nonagle);

        if (skb->len > limit &&
            unlikely(tso_fragment(sk, TCP_FRAG_IN_WRITE_QUEUE,
                      skb, limit, mss_now, gfp)))
            break;

        if (tcp_small_queue_check(sk, skb, 0))
            break;

        if (unlikely(tcp_transmit_skb(sk, skb, 1, gfp))) //调用该函数发送数据
            break;
    ......
    
 ```
tcp_write_xmit位于tcpoutput.c中，它实现了tcp的拥塞控制，然后调用了tcp_transmit_skb(sk, skb, 1, gfp)传输数据，实际上调用的是__tcp_transmit_skb。

__tcp_transmit_skb的部分代码如下所示：

```c
static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
                  int clone_it, gfp_t gfp_mask, u32 rcv_nxt)
{
    
    skb_push(skb, tcp_header_size);
    skb_reset_transport_header(skb);
    ......
    /* 构建TCP头部和校验和 */
    th = (struct tcphdr *)skb->data;
    th->source        = inet->inet_sport;
    th->dest        = inet->inet_dport;
    th->seq            = htonl(tcb->seq);
    th->ack_seq        = htonl(rcv_nxt);

    tcp_options_write((__be32 *)(th + 1), tp, &opts);
    skb_shinfo(skb)->gso_type = sk->sk_gso_type;
    if (likely(!(tcb->tcp_flags & TCPHDR_SYN))) {
        th->window      = htons(tcp_select_window(sk));
        tcp_ecn_send(sk, skb, th, tcp_header_size);
    } else {
        /* RFC1323: The window in SYN & SYN/ACK segments
         * is never scaled.
         */
        th->window    = htons(min(tp->rcv_wnd, 65535U));
    }
    ......
    icsk->icsk_af_ops->send_check(sk, skb);

    if (likely(tcb->tcp_flags & TCPHDR_ACK))
        tcp_event_ack_sent(sk, tcp_skb_pcount(skb), rcv_nxt);

    if (skb->len != tcp_header_size) {
        tcp_event_data_sent(tp, sk);
        tp->data_segs_out += tcp_skb_pcount(skb);
        tp->bytes_sent += skb->len - tcp_header_size;
    }

    if (after(tcb->end_seq, tp->snd_nxt) || tcb->seq == tcb->end_seq)
        TCP_ADD_STATS(sock_net(sk), TCP_MIB_OUTSEGS,
                  tcp_skb_pcount(skb));

    tp->segs_out += tcp_skb_pcount(skb);
    /* OK, its time to fill skb_shinfo(skb)->gso_{segs|size} */
    skb_shinfo(skb)->gso_segs = tcp_skb_pcount(skb);
    skb_shinfo(skb)->gso_size = tcp_skb_mss(skb);

    /* Leave earliest departure time in skb->tstamp (skb->skb_mstamp_ns) */

    /* Cleanup our debris for IP stacks */
    memset(skb->cb, 0, max(sizeof(struct inet_skb_parm),
                   sizeof(struct inet6_skb_parm)));

    err = icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl); //调用网络层的发送接口
    ......
}
```

__tcp_transmit_skb是位于传输层发送tcp数据的最后一步，这里首先对TCP数据段的头部进行了处理，然后调用了网络层提供的发送接口：

icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl);实现了数据的发送，自此，数据离开了传输层，传输层的任务也就结束了。

传输层时序图如下图所示：

![image](https://user-images.githubusercontent.com/87457873/127459705-58e89b43-bcfb-43a4-861e-9affcefed5c1.png)

GDB调试如下所示。

![image](https://user-images.githubusercontent.com/87457873/127459738-790d6b13-ecc0-4ba2-91f0-21b06f51b56c.png)

### 2、网络层分析

将TCP传输过来的数据包打包成IP数据报，将数据打包成IP数据包之后，通过调用ip_local_out函数，在该函数内部调用了__ip_local_out，该函数返回了一个nf_hook函数，在该函数内部调用了dst_output

```c
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl)
{
    struct inet_sock *inet = inet_sk(sk);
    struct net *net = sock_net(sk);
    struct ip_options_rcu *inet_opt;
    struct flowi4 *fl4;
    struct rtable *rt;
    struct iphdr *iph;
    int res;

    /* Skip all of this if the packet is already routed,
     * f.e. by something like SCTP.
     */
    rcu_read_lock();
    /*
     * 如果待输出的数据包已准备好路由缓存，
     * 则无需再查找路由，直接跳转到packet_routed
     * 处作处理。
     */
    inet_opt = rcu_dereference(inet->inet_opt);
    fl4 = &fl->u.ip4;
    rt = skb_rtable(skb);
    if (rt)
        goto packet_routed;

    /* Make sure we can route this packet. */
    /*
     * 如果输出该数据包的传输控制块中
     * 缓存了输出路由缓存项，则需检测
     * 该路由缓存项是否过期。
     * 如果过期，重新通过输出网络设备、
     * 目的地址、源地址等信息查找输出
     * 路由缓存项。如果查找到对应的路
     * 由缓存项，则将其缓存到传输控制
     * 块中，否则丢弃该数据包。
     * 如果未过期，则直接使用缓存在
     * 传输控制块中的路由缓存项。
     */
    rt = (struct rtable *)__sk_dst_check(sk, 0);
    if (!rt) { /* 缓存过期 */
        __be32 daddr;

        /* Use correct destination address if we have options. */
        daddr = inet->inet_daddr; /* 目的地址 */
        if (inet_opt && inet_opt->opt.srr)
            daddr = inet_opt->opt.faddr; /* 严格路由选项 */

        /* If this fails, retransmit mechanism of transport layer will
         * keep trying until route appears or the connection times
         * itself out.
         */ /* 查找路由缓存 */
        rt = ip_route_output_ports(net, fl4, sk,
                       daddr, inet->inet_saddr,
                       inet->inet_dport,
                       inet->inet_sport,
                       sk->sk_protocol,
                       RT_CONN_FLAGS(sk),
                       sk->sk_bound_dev_if);
        if (IS_ERR(rt))
            goto no_route;
        sk_setup_caps(sk, &rt->dst);  /* 设置控制块的路由缓存 */
    }
    skb_dst_set_noref(skb, &rt->dst);/* 将路由设置到skb中 */

packet_routed:
    if (inet_opt && inet_opt->opt.is_strictroute && rt->rt_uses_gateway)
        goto no_route;

    /* OK, we know where to send it, allocate and build IP header. */
    /*
     * 设置IP首部中各字段的值。如果存在IP选项，
     * 则在IP数据包首部中构建IP选项。
     */
    skb_push(skb, sizeof(struct iphdr) + (inet_opt ? inet_opt->opt.optlen : 0));
    skb_reset_network_header(skb);
    iph = ip_hdr(skb);/* 构造ip头 */
    *((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (inet->tos & 0xff));
    if (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df)
        iph->frag_off = htons(IP_DF);
    else
        iph->frag_off = 0;
    iph->ttl      = ip_select_ttl(inet, &rt->dst);
    iph->protocol = sk->sk_protocol;
    ip_copy_addrs(iph, fl4);

    /* Transport layer set skb->h.foo itself. */
     /* 构造ip选项 */
    if (inet_opt && inet_opt->opt.optlen) {
        iph->ihl += inet_opt->opt.optlen >> 2;
        ip_options_build(skb, &inet_opt->opt, inet->inet_daddr, rt, 0);
    }

    ip_select_ident_segs(net, skb, sk,
                 skb_shinfo(skb)->gso_segs ?: 1);

    /* TODO : should we use skb->sk here instead of sk ? */
    /*
     * 设置输出数据包的QoS类型。
     */
    skb->priority = sk->sk_priority;
    skb->mark = sk->sk_mark;

    res = ip_local_out(net, sk, skb);  /* 输出 */
    rcu_read_unlock();
    return res;

no_route:
    rcu_read_unlock();
    /*
     * 如果查找不到对应的路由缓存项，
     * 在此处理，将该数据包丢弃。
     */
    IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
    kfree_skb(skb);
    return -EHOSTUNREACH;
}
```

dst_output()实际调用skb_dst(skb)->output(skb)，skb_dst(skb)就是skb所对应的路由项。skb_dst(skb)指向的是路由项dst_entry，它的input在收到报文时赋值ip_local_deliver()，而output在发送报文时赋值ip_output()，该函数的作用是处理单播数据报，设置数据报的输出网络设备以及网络层协议类型参数。随后调用ip_finish_output，观察数据报长度是否大于MTU，若大于，则调用ip_fragment分片，否则调用ip_finish_output2输出。在ip_finish_output2函数中会检测skb的前部空间是否还能存储链路层首部。如果不够，就会申请更大的存储空间，最终会调用邻居子系统的输出函数neigh_output进行输出，输出分为有二层头缓存和没有两种情况，有缓存时调用neigh_hh_output进行快速输出，没有缓存时，则调用邻居子系统的输出回调函数进行慢速输出。

网络层时序图如下图所示。

![image](https://user-images.githubusercontent.com/87457873/127459859-c301360c-a540-497c-ac5d-26f4de0dbdb9.png)

GDB调试结果如下。

![image](https://user-images.githubusercontent.com/87457873/127459945-d8dcffe1-39b2-42de-a877-8cbed25b7f1e.png)

### 3、数据链路层分析

网络层最终会通过调用dev_queue_xmit来发送报文，在该函数中调用的是__dev_queue_xmit(skb, NULL);，如下所示：
```c
int dev_queue_xmit(struct sk_buff *skb)
{
    return __dev_queue_xmit(skb, NULL);
}
```
直接调用__dev_queue_xmit传入的参数是一个skb 数据包

__dev_queue_xmit函数会根据不同的情况会调用__dev_xmit_skb或者sch_direct_xmit函数，最终会调用dev_hard_start_xmit函数，该函数最终会调用xmit_one来发送一到多个数据包。

数据链路层时序图如下所示。

![image](https://user-images.githubusercontent.com/87457873/127460006-68b34ede-8413-4ab7-b8fc-8a260857151e.png)

GDB调试结果。

![image](https://user-images.githubusercontent.com/87457873/127460024-8e3fa78f-5695-4da9-8194-20258a22b2cc.png)

## 四、recv分析

### 1、数据链路层分析

在数据链路层接受数据并传递给上层的步骤如下所示：

1、一个 package 到达机器的物理网络适配器，当它接收到数据帧时，就会触发一个中断，并将通过 DMA 传送到位于 linux kernel 内存中的 rx_ring。<br>
2、网卡发出中断，通知 CPU 有个 package 需要它处理。中断处理程序主要进行以下一些操作，包括分配 skb_buff 数据结构，并将接收到的数据帧从网络适配器I/O端口拷贝到skb_buff 缓冲区中；从数据帧中提取出一些信息，并设置 skb_buff相应的参数，这些参数将被上层的网络协议使用，例如skb->protocol；<br>
3、终端处理程序经过简单处理后，发出一个软中断（NET_RX_SOFTIRQ），通知内核接收到新的数据帧。<br>
4、内核 2.5 中引入一组新的 API 来处理接收的数据帧，即 NAPI。所以，驱动有两种方式通知内核：(1) 通过以前的函数netif_rx；(2)通过NAPI机制。该中断处理程序调用 Network device的 netif_rx_schedule函数，进入软中断处理流程，再调用net_rx_action函数。<br>
5、该函数关闭中断，获取每个 Network device 的 rx_ring 中的所有 package，最终 pacakage 从 rx_ring 中被删除，进入netif _receive_skb处理流程。<br>
6、netif_receive_skb是链路层接收数据报的最后一站。它根据注册在全局数组 ptype_all 和 ptype_base 里的网络层数据报类型，把数据报递交给不同的网络层协议的接收函数(INET域中主要是ip_rcv和arp_rcv)。该函数主要就是调用第三层协议的接收函数处理该skb包，进入第三层网络层处理。<br>

数据链路层的时序图如下所示。

![image](https://user-images.githubusercontent.com/87457873/127460208-acaf449e-8a4e-4938-afa5-46fd3c7e31c3.png)

GDB调试如下。

![image](https://user-images.githubusercontent.com/87457873/127460233-529bc635-9488-4fc1-8f3d-383185560a4b.png)

### 2、网络层分析

ip层的入口在ip_rcv函数，该函数首先会做包括 package checksum 在内的各种检查，如果需要的话会做 IP defragment（将多个分片合并），然后 packet 调用已经注册的 Pre-routing netfilter hook ，完成后最终到达ip_rcv_finish函数。

```c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
       struct net_device *orig_dev)
{
    struct net *net = dev_net(dev);

    skb = ip_rcv_core(skb, net);
    if (skb == NULL)
        return NET_RX_DROP;

    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
               net, NULL, skb, dev, NULL,
               ip_rcv_finish);
}
```

ip_rcv_finish函数如下所示：

```c
static int ip_rcv_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct net_device *dev = skb->dev;
    int ret;

    /* if ingress device is enslaved to an L3 master device pass the
     * skb to its handler for processing
     */
    skb = l3mdev_ip_rcv(skb);
    if (!skb)
        return NET_RX_SUCCESS;

    ret = ip_rcv_finish_core(net, sk, skb, dev, NULL);
    if (ret != NET_RX_DROP)
        ret = dst_input(skb);
    return ret;
}
```
ip_rcv_finish 函数最终会调用ip_route_input函数，进入路由处理环节。它首先会调用 ip_route_input 来更新路由，然后查找 route，决定该 package 将会被发到本机还是会被转发还是丢弃：

1、如果是发到本机的话，调用ip_local_deliver 函数，可能会做 de-fragment（合并多个 IP packet），然后调用ip_local_deliver函数。该函数根据 package 的下一个处理层的 protocal number，调用下一层接口，包括 tcp_v4_rcv （TCP）, udp_rcv （UDP），icmp_rcv (ICMP)，igmp_rcv(IGMP)。对于 TCP 来说，函数 tcp_v4_rcv 函数会被调用，从而处理流程进入 TCP 栈。<br>
2、如果需要转发 （forward），则进入转发流程。该流程需要处理 TTL，再调用dst_input函数。该函数会 （1）处理 Netfilter Hook （2）执行 IP fragmentation （3）调用 dev_queue_xmit，进入链路层处理流程。

网络层时序图如下图所示。

![image](https://user-images.githubusercontent.com/87457873/127460441-3fe2e740-5f9e-44f1-b99b-872167a92176.png)

GDB调试如下图所示。

![image](https://user-images.githubusercontent.com/87457873/127460461-f71b6ea5-0573-4d32-9087-cf9a1c8d88a5.png)

### 3、传输层分析

对于recv函数，与send函数类似，调用的系统调用是__sys_recvfrom，其代码如下所示：

```c
int __sys_recvfrom(int fd, void __user *ubuf, size_t size, unsigned int flags,
           struct sockaddr __user *addr, int __user *addr_len)
{
    ......
    err = import_single_range(READ, ubuf, size, &iov, &msg.msg_iter);
    if (unlikely(err))
        return err;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    .....
    msg.msg_control = NULL;
    msg.msg_controllen = 0;
    /* Save some cycles and don't copy the address if not needed */
    msg.msg_name = addr ? (struct sockaddr *)&address : NULL;
    /* We assume all kernel code knows the size of sockaddr_storage */
    msg.msg_namelen = 0;
    msg.msg_iocb = NULL;
    msg.msg_flags = 0;
    if (sock->file->f_flags & O_NONBLOCK)
        flags |= MSG_DONTWAIT;
    err = sock_recvmsg(sock, &msg, flags); //调用该函数接受数据

    if (err >= 0 && addr != NULL) {
        err2 = move_addr_to_user(&address,
                     msg.msg_namelen, addr, addr_len);
    .....
}
```

__sys_recvfrom通过调用sock_recvmsg来对数据进行接收，该函数实际调用的是sock->ops->recvmsg(sock, msg, msg_data_left(msg), flags); ，同样类似send函数中，调用的实际上是socket在初始化时赋值给结构体struct proto tcp_prot的函数tcp_rcvmsg，如下所示：

```c
struct proto tcp_prot = {
    .name            = "TCP",
    .owner            = THIS_MODULE,
    .close            = tcp_close,
    .pre_connect        = tcp_v4_pre_connect,
    .connect        = tcp_v4_connect,
    .disconnect        = tcp_disconnect,
    .accept            = inet_csk_accept,
    .ioctl            = tcp_ioctl,
    .init            = tcp_v4_init_sock,
    .destroy        = tcp_v4_destroy_sock,
    .shutdown        = tcp_shutdown,
    .setsockopt        = tcp_setsockopt,
    .getsockopt        = tcp_getsockopt,
    .keepalive        = tcp_set_keepalive,
    .recvmsg        = tcp_recvmsg,
    .sendmsg        = tcp_sendmsg,
  ...
tcp_rcvmsg的代码如下所示：
int tcp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len, int nonblock,
        int flags, int *addr_len)
{
    ......
    if (sk_can_busy_loop(sk) && skb_queue_empty(&sk->sk_receive_queue) &&
        (sk->sk_state == TCP_ESTABLISHED))
        sk_busy_loop(sk, nonblock); //如果接收队列为空，则会在该函数内循环等待

    lock_sock(sk);
    .....
        if (unlikely(tp->repair)) {
        err = -EPERM;
        if (!(flags & MSG_PEEK))
            goto out;

        if (tp->repair_queue == TCP_SEND_QUEUE) 
            goto recv_sndq;

        err = -EINVAL;
        if (tp->repair_queue == TCP_NO_QUEUE)
            goto out;
    ......
        last = skb_peek_tail(&sk->sk_receive_queue); 
        skb_queue_walk(&sk->sk_receive_queue, skb) {
            last = skb;
    ......
            if (!(flags & MSG_TRUNC)) {
            err = skb_copy_datagram_msg(skb, offset, msg, used); //将接收到的数据拷贝到用户态
            if (err) {
                /* Exception. Bailout! */
                if (!copied)
                    copied = -EFAULT;
                break;
            }
        }

        *seq += used;
        copied += used;
        len -= used;

        tcp_rcv_space_adjust(sk);
```
在连接建立后，若没有数据到来，接收队列为空，进程会在sk_busy_loop函数内循环等待，知道接收队列不为空，并调用函数数skb_copy_datagram_msg将接收到的数据拷贝到用户态，该函数内部实际调用的是__skb_datagram_iter，其代码如下所示：

```c
int __skb_datagram_iter(const struct sk_buff *skb, int offset,
            struct iov_iter *to, int len, bool fault_short,
            size_t (*cb)(const void *, size_t, void *, struct iov_iter *),
            void *data)
{
    int start = skb_headlen(skb);
    int i, copy = start - offset, start_off = offset, n;
    struct sk_buff *frag_iter;

    /* 拷贝tcp头部 */
    if (copy > 0) {
        if (copy > len)
            copy = len;
        n = cb(skb->data + offset, copy, data, to);
        offset += n;
        if (n != copy)
            goto short_copy;
        if ((len -= copy) == 0)
            return 0;
    }

    /* 拷贝数据部分 */
    for (i = 0; i < skb_shinfo(skb)->nr_frags; i++) {
        int end;
        const skb_frag_t *frag = &skb_shinfo(skb)->frags[i];

        WARN_ON(start > offset + len);

        end = start + skb_frag_size(frag);
        if ((copy = end - offset) > 0) {
            struct page *page = skb_frag_page(frag);
            u8 *vaddr = kmap(page);

            if (copy > len)
                copy = len;
            n = cb(vaddr + frag->page_offset +
                offset - start, copy, data, to);
            kunmap(page);
            offset += n;
            if (n != copy)
                goto short_copy;
            if (!(len -= copy))
                return 0;
        }
        start = end;
    }
```
传输层时序图如下图所示。

![image](https://user-images.githubusercontent.com/87457873/127460642-9deb8274-b587-45bb-8e33-5eea8a69f247.png)

 GDB调试如下图所示。

![image](https://user-images.githubusercontent.com/87457873/127460687-9c34a4fe-db7b-4f61-a84c-d5aa894b96c2.png)

## 五、小结

从Linux操作系统实现入手，深入的分析了Linux操作系统对于TCP/IP栈的实现原理与具体过程，了解了Linux网络子系统的具体构成及流程，通过这次调研，使我对TCP/IP协议的原理及具体实现有了极其深入的理解。
