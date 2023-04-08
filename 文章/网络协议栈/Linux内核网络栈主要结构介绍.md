## 1、socket

（include\linux\Socket.h）该结构体socket 主要使用在BSD socket 层，是最上层的结构，在INET socket 层也会有涉及，但很少。

```text
/* 
 * Internal representation of a socket. not all the fields are used by 
 * all configurations: 
 * 
 *            server                client 
 * conn     client connected to server connected to 
 * iconn        list of clients     -unused- 
 *       awaiting connections 
 * wait     sleep for clients,  sleep for connection, 
 *          sleep for i/o               sleep for i/o 
 */  
 //该结构表示一个网络套接字  
struct socket {  
  short         type;       /* 套接字所用的流类型*/  
  socket_state      state;//套接字所处状态  
  long          flags;//标识字段，目前尚无明确作用  
  struct proto_ops  *ops;       /* 操作函数集指针  */  
    /* data保存指向‘私有'数据结构指针，在不同的域指向不同的数据结构        */  
  //在INET域，指向sock结构，UNIX域指向unix_proto_data结构  
  void          *data;    
  //下面两个字段只用于UNIX域  
  struct socket     *conn;      /* 指向连接的对端套接字   */  
  struct socket     *iconn;     /* 指向正等待连接的客户端(服务器端)    */  
  struct socket     *next;//链表  
  struct wait_queue **wait;     /* 等待队列 */  
  struct inode      *inode;//inode结构指针  
  struct fasync_struct  *fasync_list;   /* 异步唤醒链表结构 */  
};  
```

## 2、sock

(include\linux\Net.h) sock 的使用范围比socket 要大得多，sock结构的使用基本贯穿硬件层、设备接口层、ip层、INET socket 层，而且是作为各层之间的一个联系，主要是因为无论是发送还是接收的数据包都要被缓存到sock 结构中的缓冲队列中。

sock 结构与其对应的 socket 会相互绑定。

```text
/* 
 * This structure really needs to be cleaned up. 
 * Most of it is for TCP, and not used by any of 
 * the other protocols. 
 * 大部分功能是为TCP准备的 
 */  
struct sock {  
  struct options        *opt;//IP选项缓冲于此处  
  volatile unsigned long    wmem_alloc;//发送缓冲队列中存放的数据的大小，这两个与后面的rcvbuf和sndbuf一起使用  
  volatile unsigned long    rmem_alloc;//接收缓冲队列中存放的数据的大小  
  /* 下面三个seq用于TCP协议中为保证可靠数据传输而使用的序列号 */  
  unsigned long         write_seq;//  
  unsigned long         sent_seq;//  
  unsigned long         acked_seq;//  
  unsigned long         copied_seq;//应用程序有待读取(但尚未读取)数据的第一个序列号  
  unsigned long         rcv_ack_seq;//目前本地接收到的对本地发送数据的应答序列号  
  unsigned long         window_seq;//窗口大小  
  unsigned long         fin_seq;//应答序列号  
  //下面两个字段用于紧急数据处理  
  unsigned long         urg_seq;//紧急数据最大序列号  
  unsigned long         urg_data;//标志位，1表示收到紧急数据  
  
  /* 
   * Not all are volatile, but some are, so we 
   * might as well say they all are. 
   */  
  volatile char                 inuse,//表示其他进程正在使用该sock结构，本进程需等待  
                dead,//表示该sock结构已处于释放状态  
                urginline,//=1，表示紧急数据将被当做普通数据处理  
                intr,//  
                blog,  
                done,  
                reuse,  
                keepopen,//=1，使用保活定时器  
                linger,//=1，表示在关闭套接字时需要等待一段时间以确认其已关闭  
                delay_acks,//=1，表示延迟应答  
                destroy,//=1，表示该sock结构等待销毁  
                ack_timed,  
                no_check,  
                zapped, /* In ax25 & ipx means not linked */  
                broadcast,  
                nonagle;//=1，表示不使用NAGLE算法  
                //NAGLE算法:在前一个发送的数据包被应答之前，不可再继续发送其它数据包  
  unsigned long             lingertime;//等待关闭操作的时间  
  int               proc;//该sock结构所属的进程的进程号  
  struct sock           *next;  
  struct sock           *prev; /* Doubly linked chain.. */  
  struct sock           *pair;  
  //下面两个字段用于TCP协议重发队列  
  struct sk_buff        * volatile send_head;//这个队列中的数据均已经发送出去，但尚未接收到应答  
  struct sk_buff        * volatile send_tail;  
  struct sk_buff_head       back_log;//接收的数据包缓存队列，当套接字正忙时，数据包暂存在这里  
  struct sk_buff        *partial;//用于创建最大长度的待发送数据包  
  struct timer_list     partial_timer;//定时器，用于按时发送partial指针指向的数据包  
  long              retransmits;//重发次数  
  struct sk_buff_head       write_queue,//指向待发送数据包  
                receive_queue;//读队列，表示数据报已被正式接收，该队列中的数据可被应用程序读取?  
  struct proto          *prot;//传输层处理函数集  
  struct wait_queue     **sleep;  
  unsigned long         daddr;//sock结构所代表套接字的远端地址  
  unsigned long         saddr;//本地地址  
  unsigned short        max_unacked;//最大未处理请求连接数  
  unsigned short        window;//远端窗口大小  
  unsigned short        bytes_rcv;//已接收字节总数  
/* mss is min(mtu, max_window) */  
  unsigned short        mtu; //和链路层协议密切相关      /* 最大传输单元 */  
  volatile unsigned short   mss; //最大报文长度 =mtu-ip首部长度-tcp首部长度，也就是tcp数据包每次能够传输的最大数据分段  
  volatile unsigned short   user_mss;  /* mss requested by user in ioctl */  
  volatile unsigned short   max_window;//最大窗口大小  
  unsigned long         window_clamp;//窗口大小钳制值  
  unsigned short        num;//本地端口号  
  //下面三个字段用于拥塞算法  
  volatile unsigned short   cong_window;  
  volatile unsigned short   cong_count;  
  volatile unsigned short   ssthresh;  
  volatile unsigned short   packets_out;//本地已发送出去但尚未得到应答的数据包数目  
  volatile unsigned short   shutdown;//本地关闭标志位，用于半关闭操作  
  volatile unsigned long    rtt;//往返时间估计值  
  volatile unsigned long    mdev;//绝对偏差  
  volatile unsigned long    rto;//用rtt和mdev 用算法计算出的延迟时间值  
/* currently backoff isn't used, but I'm maintaining it in case 
 * we want to go back to a backoff formula that needs it 
 */  
  volatile unsigned short   backoff;//退避算法度量值  
  volatile short        err;//错误标志值  
  unsigned char         protocol;//传输层协议值  
  volatile unsigned char    state;//套接字状态值  
  volatile unsigned char    ack_backlog;//缓存的未应答数据包个数  
  unsigned char         max_ack_backlog;//最大缓存的未应答数据包个数  
  unsigned char         priority;//该套接字优先级  
  unsigned char         debug;  
  unsigned short        rcvbuf;//最大接收缓冲区大小  
  unsigned short        sndbuf;//最大发送缓冲区大小  
  unsigned short        type;//类型值如 SOCK_STREAM  
  unsigned char         localroute;//=1,表示只使用本地路由 /* Route locally only */  
#ifdef CONFIG_IPX  
  ipx_address           ipx_dest_addr;  
  ipx_interface         *ipx_intrfc;  
  unsigned short        ipx_port;  
  unsigned short        ipx_type;  
#endif  
#ifdef CONFIG_AX25  
/* Really we want to add a per protocol private area */  
  ax25_address          ax25_source_addr,ax25_dest_addr;  
  struct sk_buff *volatile  ax25_retxq[8];  
  char              ax25_state,ax25_vs,ax25_vr,ax25_lastrxnr,ax25_lasttxnr;  
  char              ax25_condition;  
  char              ax25_retxcnt;  
  char              ax25_xx;  
  char              ax25_retxqi;  
  char              ax25_rrtimer;  
  char              ax25_timer;  
  unsigned char         ax25_n2;  
  unsigned short        ax25_t1,ax25_t2,ax25_t3;  
  ax25_digi         *ax25_digipeat;  
#endif    
#ifdef CONFIG_ATALK  
  struct atalk_sock     at;  
#endif  
  
/* IP 'private area' or will be eventually */  
  int               ip_ttl;//ip首部ttl字段值，实际上表示路由器跳数      /* TTL setting */  
  int               ip_tos;//ip首部tos字段值，服务类型值       /* TOS */  
  struct tcphdr         dummy_th;//缓存的tcp首部，在tcp协议中创建一个发送数据包时可以利用此字段快速创建tcp首部  
  struct timer_list     keepalive_timer;//保活定时器，用于探测对方窗口大小，防止对方通报窗口大小的数据包丢弃 /* TCP keepalive hack */  
  struct timer_list     retransmit_timer;//重发定时器，用于数据包超时重发  /* TCP retransmit timer */  
  struct timer_list     ack_timer;//延迟应答定时器     /* TCP delayed ack timer */  
  int               ip_xmit_timeout;//表示定时器超时原因 /* Why the timeout is running */  
  
//用于ip多播  
#ifdef CONFIG_IP_MULTICAST    
  int               ip_mc_ttl;          /* Multicasting TTL */  
  int               ip_mc_loop;         /* Loopback (not implemented yet) */  
  char              ip_mc_name[MAX_ADDR_LEN];   /* Multicast device name */  
  struct ip_mc_socklist     *ip_mc_list;            /* Group array */  
#endif    
  
  /* This part is used for the timeout functions (timer.c). */  
  int               timeout;    /* What are we waiting for? */  
  struct timer_list     timer;      /* This is the TIME_WAIT/receive timer when we are doing IP */  
  struct timeval        stamp;  
  
  /* identd */  
  //一个套接在在不同的层次上分别由socket结构和sock结构表示  
  struct socket         *socket;  
    
  /* Callbacks *///回调函数  
  void              (*state_change)(struct sock *sk);  
  void              (*data_ready)(struct sock *sk,int bytes);  
  void              (*write_space)(struct sock *sk);  
  void              (*error_report)(struct sock *sk);  
    
};  
```

## 3、sk_buff

(include\linux\Skbuff.h) sk_buff 是网络数据报在内核中的表现形式，通过源码可以看出，数据包在内核协议栈中是通过这个数据结构来变现的。

从其中的 union 字段可以看出，该结构是贯穿在各个层的，可以说这个结构是用来为网络数据包服务的。其中的字段表明了数据包隶属的套接字、当前所处的协议层、所搭载的数据负载长度（data指针指向）、源端，目的端地址以及相关字段等。

主要重要的一个字段是 data[0]，这是一个指针，它指向对应层的数据报（首部+数据负载）内容的首地址。怎么解释呢？

如果在传输层，那么data指向的数据部分的首地址，其数据部分为 TCP 首部 + 有效数据负载。

如果在网络层，data指向的数据部分的首地址，其数据部分为 IP 首部 + TCP 首部 + 有效数据负载。

如果在链路层，data指向的首地址，其数据布局为 MAC 首部 + IP 首部 + TCP 首部 + 有效数据负载。

所以在该skb_buff结构传递时，获取某一层的首部，都是通过拷贝 data 指向地址对应首部大小的数据。

```text
  //sk_buff 结构用来封装网络数据  
  //网络栈代码对数据的处理都是以sk_buff 结构为单元进行的  
struct sk_buff {  
  struct sk_buff        * volatile next;  
  struct sk_buff        * volatile prev;//构成队列  
#if CONFIG_SKB_CHECK  
  int               magic_debug_cookie; //调试用  
#endif  
  struct sk_buff        * volatile link3; //构成数据包重发队列  
  struct sock           *sk; //数据包所属的套接字  
  volatile unsigned long    when;    //数据包的发送时间，用于计算往返时间RTT/* used to compute rtt's */  
  struct timeval        stamp; //记录时间  
  struct device         *dev; //接收该数据包的接口设备  
  struct sk_buff        *mem_addr; //该sk_buff在内存中的基地址，用于释放该sk_buff结构  
  //联合类型，表示数据报在不同处理层次上所到达的处理位置  
  union {  
    struct tcphdr   *th; //传输层tcp，指向首部第一个字节位置  
    struct ethhdr   *eth; //链路层上，指向以太网首部第一个字节位置  
    struct iphdr    *iph; //网络层上，指向ip首部第一个字节位置  
    struct udphdr   *uh; //传输层udp协议，  
    unsigned char   *raw; //随层次变化而变化，链路层=eth，网络层=iph  
    unsigned long   seq; //针对tcp协议的待发送数据包而言，表示该数据包的ACK值  
  } h;  
  struct iphdr      *ip_hdr; //指向ip首部的指针        /* For IPPROTO_RAW */  
  unsigned long         mem_len; //表示sk_buff结构大小加上数据部分的总长度  
  unsigned long         len; //只表示数据部分长度，len = mem_len - sizeof(sk_buff)  
  unsigned long         fraglen; //分片数据包个数  
  struct sk_buff        *fraglist;  /* Fragment list */  
  unsigned long         truesize; //同men_len  
  unsigned long         saddr; //源端ip地址  
  unsigned long         daddr; //目的端ip地址  
  unsigned long         raddr; //数据包下一站ip地址     /* next hop addr */  
   //标识字段  
  volatile char         acked, //=1，表示数据报已得到确认，可以从重发队列中删除  
                used, //=1，表示该数据包的数据已被应用程序读完，可以进行释放  
                free, //用于数据包发送，=1表示再进行发送操作后立即释放，无需缓存  
                arp; //用于待发送数据包，=1表示已完成MAC首部的建立，=0表示还不知道目的端MAC地址  
  //已进行tries试发送，该数据包正在被其余部分使用，路由类型，数据包类型  
  unsigned char         tries,lock,localroute,pkt_type;  
   //下面是数据包的类型，即pkt_type的取值  
#define PACKET_HOST     0     //发往本机    /* To us */  
#define PACKET_BROADCAST    1 //广播  
#define PACKET_MULTICAST    2 //多播  
#define PACKET_OTHERHOST    3 //其他机器        /* Unmatched promiscuous */  
  unsigned short        users; //使用该数据包的模块数     /* User count - see datagram.c (and soon seqpacket.c/stream.c) */  
  unsigned short        pkt_class;  /* For drivers that need to cache the packet type with the skbuff (new PPP) */  
#ifdef CONFIG_SLAVE_BALANCING  
  unsigned short        in_dev_queue; //该字段是否正在缓存于设备缓存队列中  
#endif    
  unsigned long         padding[0]; //填充字节  
  unsigned char         data[0]; //指向该层数据部分  
  //data指向的数据负载首地址，在各个层对应不同的数据部分  
//从侧面看出sk_buff结构基本上是贯穿整个网络栈的非常重要的一个数据结构  
};  
```

## 4、device

（include\linux\Netdevice.h）该结构表明了一个网络设备需要的字段信息。

```text
/* 
 * The DEVICE structure. 
 * Actually, this whole structure is a big mistake.  It mixes I/O 
 * data with strictly "high-level" data, and it has to know about 
 * almost every data structure used in the INET module.   
 */  
  //网络设备结构  
struct device   
{  
  
  /* 
   * This is the first field of the "visible" part of this structure 
   * (i.e. as seen by users in the "Space.c" file).  It is the name 
   * the interface. 
   */  
  char            *name;//设备名称  
  
  /* I/O specific fields - FIXME: Merge these and struct ifmap into one */  
  unsigned long       rmem_end;//设备读缓冲区空间       /* shmem "recv" end */  
  unsigned long       rmem_start;       /* shmem "recv" start   */  
  unsigned long       mem_end;//设备总缓冲区首地址和尾地址       /* sahared mem end  */  
  unsigned long       mem_start;        /* shared mem start */  
  unsigned long       base_addr;//设备寄存器读写IO基地址      /* device I/O address   */  
  unsigned char       irq;  //设备所使用中断号      /* device IRQ number    */  
  
  /* Low-level status flags. */  
  volatile unsigned char  start,//=1，表示设备已处于工作状态        /* start an operation   */  
                          tbusy,//=1，表示设备正忙于数据包发送       /* transmitter busy */  
                          interrupt;//=1，软件正在进行设备中断处理       /* interrupt arrived    */  
  
  struct device       *next;//构成设备队列  
  
  /* The device initialization function. Called only once. */  
  int             (*init)(struct device *dev);//设备初始化指针(函数指针)  
  
  /* Some hardware also needs these fields, but they are not part of the 
     usual set specified in Space.c. */  
  unsigned char       if_port;//指定使用的设备端口号      /* Selectable AUI, TP,..*/  
  unsigned char       dma;//设备所用的dma通道号         /* DMA channel      */  
  
  struct enet_statistics* (*get_stats)(struct device *dev);//设备信息获取函数指针  
  
  /* 
   * This marks the end of the "visible" part of the structure. All 
   * fields hereafter are internal to the system, and may change at 
   * will (read: may be cleaned up at will). 
   */  
  
  /* These may be needed for future network-power-down code. */  
  unsigned long       trans_start;//用于传输超时计算    /* Time (in jiffies) of last Tx */  
  unsigned long       last_rx;//上次接收一个数据包的时间    /* Time of last Rx      */  
  
  unsigned short      flags;//标志位   /* interface flags (a la BSD)   */  
  unsigned short      family;//设备所属的域协议 /* address family ID (AF_INET)  */  
  unsigned short      metric;   /* routing metric (not used)    */  
  unsigned short      mtu;//该接口设备的最大传输单元，ip首部+tcp首部+有效数据负载，去掉了以太网帧的帧头 /* interface MTU value*/  
  unsigned short      type;//该设备所属硬件类型      /* interface hardware type  */  
  unsigned short      hard_header_len;//硬件首部长度  /* hardware hdr length  */  
  void            *priv;//私有数据指针    /* pointer to private data  */  
  
  /* Interface address info. */  
  unsigned char       broadcast[MAX_ADDR_LEN];//链路层硬件广播地址   /* hw bcast add */  
  unsigned char       dev_addr[MAX_ADDR_LEN];//本设备硬件地址  /* hw address   */  
  unsigned char       addr_len;//硬件地址长度 /* hardware address length  */  
  unsigned long       pa_addr;//本地ip地址  /* protocol address     */  
  unsigned long       pa_brdaddr;//网络层广播ip地址    /* protocol broadcast addr  */  
  unsigned long       pa_dstaddr;//点对点网络中对点的ip地址    /* protocol P-P other side addr */  
  unsigned long       pa_mask;//ip地址网络掩码    /* protocol netmask     */  
  unsigned short      pa_alen;//ip地址长度  /* protocol address length  */  
  
  struct dev_mc_list     *mc_list;//多播地址链表  /* Multicast mac addresses  */  
  int            mc_count;//多播地址数目  /* Number of installed mcasts   */  
    
  struct ip_mc_list  *ip_mc_list;//网络层ip多播地址链表  /* IP multicast filter chain    */  
      
  /* For load balancing driver pair support */  
    
  unsigned long        pkt_queue;//该设备缓存的待发送的数据包个数  /* Packets queued */  
  struct device       *slave;//从设备  /* Slave device */  
    
  
  /* Pointer to the interface buffers. */  
  struct sk_buff_head     buffs[DEV_NUMBUFFS];//设备缓存的待发送的数据包  
  
 //函数指针  
  /* Pointers to interface service routines. */  
  int             (*open)(struct device *dev);  
  int             (*stop)(struct device *dev);  
  int             (*hard_start_xmit) (struct sk_buff *skb,  
                          struct device *dev);  
  int             (*hard_header) (unsigned char *buff,  
                      struct device *dev,  
                      unsigned short type,  
                      void *daddr,  
                      void *saddr,  
                      unsigned len,  
                      struct sk_buff *skb);  
  int             (*rebuild_header)(void *eth, struct device *dev,  
                unsigned long raddr, struct sk_buff *skb);  
  
  //用于从接收到的数据包提取MAC首部中类型字符值，从而将数据包传送给适当的协议处理函数进行处理  
  unsigned short      (*type_trans) (struct sk_buff *skb,  
                     struct device *dev);  
#define HAVE_MULTICAST             
  void            (*set_multicast_list)(struct device *dev,  
                     int num_addrs, void *addrs);  
#define HAVE_SET_MAC_ADDR          
  int             (*set_mac_address)(struct device *dev, void *addr);  
#define HAVE_PRIVATE_IOCTL  
  int             (*do_ioctl)(struct device *dev, struct ifreq *ifr, int cmd);  
#define HAVE_SET_CONFIG  
  int             (*set_config)(struct device *dev, struct ifmap *map);  
    
};  
```

## 5、tcp 首部格式

```text
 //tcp首部格式  
 //http://blog.csdn.net/wenqian1991/article/details/44598537  
struct tcphdr {  
    __u16   source;//源端口号  
    __u16   dest;//目的端口号  
    __u32   seq;//32位序列号  
    __u32   ack_seq;//32位确认号  
#if defined(LITTLE_ENDIAN_BITFIELD)  
    __u16   res1:4,//4位首部长度  
        doff:4,//保留  
        //下面为各个控制位  
        fin:1,//最后控制位，表示数据已全部传输完成  
        syn:1,//同步控制位  
        rst:1,//重置控制位  
        psh:1,//推控制位  
        ack:1,//确认控制位  
        urg:1,//紧急控制位  
        res2:2;//  
#elif defined(BIG_ENDIAN_BITFIELD)  
    __u16   doff:4,  
        res1:4,  
        res2:2,  
        urg:1,  
        ack:1,  
        psh:1,  
        rst:1,  
        syn:1,  
        fin:1;  
#else  
#error  "Adjust your <asm/byteorder.h> defines"  
#endif    
    __u16   window;//16位窗口大小  
    __u16   check;//16位校验和  
    __u16   urg_ptr;//16位紧急指针  
};  
```

## 6、ip 首部格式

```text
 //ip数据报，首部格式  
struct iphdr {  
#if defined(LITTLE_ENDIAN_BITFIELD)//如果是小端模式  
    __u8    ihl:4,//首部长度  
        version:4;//版本  
#elif defined (BIG_ENDIAN_BITFIELD)//大端  
    __u8    version:4,  
        ihl:4;  
#else  
#error  "Please fix <asm/byteorder.h>"  
#endif  
    __u8    tos;//区分服务，用语表示数据报的优先级和服务类型  
    __u16   tot_len;//总长度，标识整个ip数据报的总长度 = 报头+数据部分  
    __u16   id;//表示ip数据报的标识符  
    __u16   frag_off;//片偏移  
    __u8    ttl;//生存时间，即ip数据报在网络中传输的有效期  
    __u8    protocol;//协议，标识此ip数据报在传输层所采用的协议类型  
    __u16   check;//首部校验和  
    __u32   saddr;//源地址  
    __u32   daddr;//目的地址  
    /*The options start here. */  
};  
```

## 7、以太网帧帧头格式

```text
/* This is an Ethernet frame header. */ 
struct ethhdr {  
  unsigned char     h_dest[ETH_ALEN];//目的地址 /* destination eth addr */ 
  unsigned char     h_source[ETH_ALEN];//源地址    /* source ether addr    */ 
  unsigned short    h_proto;//类型        /* packet type ID field */ 
};  
```

## 8、ARP报文报头

```text
/* 
 *  This structure defines an ethernet arp header. 
 */  
 //ARP报文格式(arp报头)  
struct arphdr  
{  
    unsigned short  ar_hrd;//硬件类型       /* format of hardware address   */  
    unsigned short  ar_pro;//上层协议类型     /* format of protocol address   */  
    unsigned char   ar_hln;//MAC地址长度        /* length of hardware address   */  
    unsigned char   ar_pln;//协议地址长度     /* length of protocol address   */  
    unsigned short  ar_op;//操作类型        /* ARP opcode (command)     */  
  
#if 0  
     /* 
      *  Ethernet looks like this : This bit is variable sized however... 
      */  
    unsigned char       ar_sha[ETH_ALEN];//源MAC地址   /* sender hardware address  */  
    unsigned char       ar_sip[4];//源IP地址       /* sender IP address        */  
    unsigned char       ar_tha[ETH_ALEN];//目的MAC地址  /* target hardware address  */  
    unsigned char       ar_tip[4];//目的IP地址      /* target IP address        */  
#endif  
  
};  
```

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549140581