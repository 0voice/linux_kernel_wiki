## 1、应用层——acce**pt 函数**

该函数返回一个已建立连接的可用于数据通信的套接字。

```text
#include <sys/socket.h> 
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);  
//返回：非负描述子——成功，-1——出错 
/*参数sockfd是监听后的套接字，这个套接字用来监听一个端口，当有一个客户与服务器连接时，它使用一个与这个套接字关联的端口号， 
比较特别的是：参数cliaddr和addrlen是一个结果参数，用来返回已连接客户的协议地址。如果对客户的地址不感兴趣，那么可以把这个值设置为NULL*/ 
```

## 2、BSD Socket 层——sock_accept 函数

```text
/* 
 *  For accept, we attempt to create a new socket, set up the link 
 *  with the client, wake up the client, then return the new 
 *  connected fd. We collect the address of the connector in kernel 
 *  space and move it to user at the very end. This is buggy because 
 *  we open the socket then return an error. 
 */  
//用于服务器接收一个客户端的连接请求，这里是值-结果参数，之前有说到  
//fd 为监听后套接字。最后返回一个记录了本地与目的端信息的套接字  
//upeer_sockaddr用来返回已连接客户的协议地址，如果对协议地址不感兴趣就NULL  
static int sock_accept(int fd, struct sockaddr *upeer_sockaddr, int *upeer_addrlen)  
{  
    struct file *file;  
    struct socket *sock, *newsock;  
    int i;  
    char address[MAX_SOCK_ADDR];  
    int len;  
  
    if (fd < 0 || fd >= NR_OPEN || ((file = current->files->fd[fd]) == NULL))  
        return(-EBADF);  
    if (!(sock = sockfd_lookup(fd, &file)))   
        return(-ENOTSOCK);  
    if (sock->state != SS_UNCONNECTED)//socket各个状态的演变是一步一步来的   
    {  
        return(-EINVAL);  
    }  
    //这是tcp连接，得按步骤来  
    if (!(sock->flags & SO_ACCEPTCON))//没有listen  
    {  
        return(-EINVAL);  
    }  
    //分配一个新的套接字，用于表示后面可进行通信的套接字  
    if (!(newsock = sock_alloc()))   
    {  
        printk("NET: sock_accept: no more sockets\n");  
        return(-ENOSR); /* Was: EAGAIN, but we are out of system 
                   resources! */  
    }  
    newsock->type = sock->type;  
    newsock->ops = sock->ops;  
    //套接字重定向，目的是初始化新的用于数据传送的套接字  
    //继承了第一参数传来的服务器的IP和端口号信息  
    if ((i = sock->ops->dup(newsock, sock)) < 0)   
    {  
        sock_release(newsock);  
        return(i);  
    }  
    //转调用inet_accept  
    i = newsock->ops->accept(sock, newsock, file->f_flags);  
    if ( i < 0)   
    {  
        sock_release(newsock);  
        return(i);  
    }  
    //分配一个文件描述符，用于以后的数据传送  
    if ((fd = get_fd(SOCK_INODE(newsock))) < 0)   
    {  
        sock_release(newsock);  
        return(-EINVAL);  
    }  
    //返回通信远端的地址  
    if (upeer_sockaddr)  
    {//得到客户端地址，并复制到用户空间  
        newsock->ops->getname(newsock, (struct sockaddr *)address, &len, 1);  
        move_addr_to_user(address,len, upeer_sockaddr, upeer_addrlen);  
    }  
    return(fd);  
}  
```

## 3、INET Socket 层——inet_accept 函数

```text
/* 
 *  Accept a pending connection. The TCP layer now gives BSD semantics. 
 */  
//先去看看sock_accept，看看各个参数的意思，newsock是dup sock后的新sock  
//sock为监听套接字，newsock为连接成功后实际用于通信的sock  
static int inet_accept(struct socket *sock, struct socket *newsock, int flags)  
{  
    struct sock *sk1, *sk2;  
    int err;  
  
    sk1 = (struct sock *) sock->data;  
  
    /* 
     * We've been passed an extra socket. 
     * We need to free it up because the tcp module creates 
     * its own when it accepts one. 
     */  
     //如果sock->data 已经指向了对应的sock结构，则把它销毁  
     //销毁旧的，后面指向新的accept后的  
    if (newsock->data)  
    {  
        struct sock *sk=(struct sock *)newsock->data;  
        newsock->data=NULL;  
        sk->dead = 1;  
        destroy_sock(sk);//销毁旧的socket对应的sock结构  
    }  
    
    if (sk1->prot->accept == NULL) //没有对应的操作函数集，退出  
        return(-EOPNOTSUPP);  
  
    /* Restore the state if we have been interrupted, and then returned. */  
//如果套接字在等待连接的过程中被中断，则监听套接字与中断的套接字关联，下次优先处理该套接字  
    if (sk1->pair != NULL )   
    {  
        sk2 = sk1->pair;  
        sk1->pair = NULL;  
    }   
    else  
    {  
//这里调用下层处理函数tcp_accept，首次调用inet_accept，sk1->pair 肯定是为NULL的，所以一开始就会执行下面的代码  
        sk2 = sk1->prot->accept(sk1,flags);//交给下层处理函数  
        if (sk2 == NULL)   
        {  
            if (sk1->err <= 0)  
                printk("Warning sock.c:sk1->err <= 0.  Returning non-error.\n");  
            err=sk1->err;  
            sk1->err=0;  
            return(-err);  
        }  
    }  
    //socket sock建立关联  
    newsock->data = (void *)sk2;//指向新的，sk2为下层函数tcp_accept返回的套接字  
    sk2->sleep = newsock->wait;//等待队列  
    sk2->socket = newsock;//回绑，指向上层的socket结构  
    newsock->conn = NULL;//还没有连接客户端  
    if (flags & O_NONBLOCK)   
        return(0);  
  
    cli(); /* avoid the race. */  
    //三次握手中间过程，tcp SYN序列号接收  
    while(sk2->state == TCP_SYN_RECV)   
    {  
    //被中断了  
        interruptible_sleep_on(sk2->sleep);  
        if (current->signal & ~current->blocked)   
        {  
            sti();  
            sk1->pair = sk2;//存入pair，下次优先处理  
            sk2->sleep = NULL;  
            sk2->socket=NULL;  
            newsock->data = NULL;  
            return(-ERESTARTSYS);  
        }  
    }  
    sti();  
    //连接失败，三次握手失败  
    if (sk2->state != TCP_ESTABLISHED && sk2->err > 0)   
    {  
        err = -sk2->err;  
        sk2->err=0;  
        sk2->dead=1; /* ANK */  
        destroy_sock(sk2);//销毁新建的sock结构  
        newsock->data = NULL;  
        return(err);  
    }  
    newsock->state = SS_CONNECTED;//已经建立了连接  
    return(0);  
}  
```

## 4、传输层——tcp_accept 函数

```text
/* 
 *  This will accept the next outstanding connection.  
 */  
 //accept->sock_accpet->inet_accpet->tcp_accept(tcp)  
 //顶层accept传值进来的套接字sk是监听套接字，然后返回可以进行数据通信的套接字  
 //tcp_accept就是从监听套接字缓存队列里面找到一个完成连接的套接字  
static struct sock *tcp_accept(struct sock *sk, int flags)  
{  
    struct sock *newsk;  
    struct sk_buff *skb;  
    
  /* 
   * We need to make sure that this socket is listening, 
   * and that it has something pending. 
   */  
  
    if (sk->state != TCP_LISTEN) //如果当前不是出于监听状态就退出  
    {  
        sk->err = EINVAL;  
        return(NULL);   
    }  
    //套接字处于监听状态  
    /* Avoid the race. */  
    cli();  
    sk->inuse = 1;//表示当前进程正在使用该sock结构，其余进程不能使用，加锁  
      
  //从监听套接字缓存队列里找到已经建立连接的套接字,并返回  
    while((skb = tcp_dequeue_established(sk)) == NULL)  
    {  
    //如果没有完成连接的，就一直陷入循环，然后重发back_log中的数据包  
        if (flags & O_NONBLOCK) //不阻塞  
        {  
            sti();  
//如果当前套接字正忙，数据包将插入到sock结构的back_log队列中，back_log只是暂居之所  
//数据包必须插入到receive_queue中才算被接收  
            release_sock(sk);//从back_log中取数据包重新调用tcp_rcv函数对数据包进行接收  
            sk->err = EAGAIN;  
            return(NULL);  
        }  
       
        release_sock(sk);//从back_log中取数据包重新调用tcp_rcv函数对数据包进行接收  
        interruptible_sleep_on(sk->sleep);  
        if (current->signal & ~current->blocked)   
        {  
            sti();  
            sk->err = ERESTARTSYS;  
            return(NULL);  
        }  
        sk->inuse = 1;//加锁  
    }  
    sti();  
  
    /* 
     *  Now all we need to do is return skb->sk.  
     */  
    newsk = skb->sk;//返回的套接字(已完成连接)  
  
    kfree_skb(skb, FREE_READ);//释放sk_buff  
    sk->ack_backlog--;//未应答数据包个数-1  
    release_sock(sk);//原套接字继续监听  
    return(newsk);  
}  
```

好，定位到 tcp_dqueue_established 函数：

```text
/* 
 *  Remove a completed connection and return it. This is used by 
 *  tcp_accept() to get connections from the queue. 
 */  
//移除sock中一个已经建立连接的数据包，并返回该数据包  
//结合前面可以看出，对于receive_queue，套接字连接的第1次握手时在该链表尾部增加一个链表节点，  
//当第3次握手完成将此节点删除,所以对于监听套接字receive_queue中保存的是不完全建立连接的套接字的数据包  
static struct sk_buff *tcp_dequeue_established(struct sock *s)  
{  
    struct sk_buff *skb;  
    unsigned long flags;  
    save_flags(flags);//保存状态  
    cli();   
    skb=tcp_find_established(s);//找到已经建立连接的数据包  
    if(skb!=NULL)  
        skb_unlink(skb);    //从队列中移除，但该数据报实体还是存在的/* Take it off the queue */  
    restore_flags(flags);  
    return skb;  
}  
```

好，再定位到 tcp_find_established 函数：

```text
/* 
 *  Find someone to 'accept'. Must be called with 
 *  sk->inuse=1 or cli() 
 */   
 //sk_buff 表示接收或发送数据报的包头信息  
/*从监听套接字缓冲队列中检查是否存在已经完成连接的远端发送的数据包，该数据包的作用是完成连接。 
本地监听套接字在处理完该连接，设置相关状态后将该数据包缓存在receive_queue中 
*/  
/*对于监听套接字而言，其接收队列中的数据包是建立连接数据包，即SYN数据包，不含数据数据包 
*/  
static struct sk_buff *tcp_find_established(struct sock *s)  
{  
//获取receive_queue中的第一个链表元素，sk_buff 结构  
    //该队列中的数据报表示已被正式接收  
  
    struct sk_buff *p=skb_peek(&s->receive_queue);//返回指向链表第一个节点的指针  
    if(p==NULL)  
        return NULL;  
    do  
    {  
    //sk的状态state是枚举类  
    //返回完成了三次握手连接的套接字  
        if(p->sk->state == TCP_ESTABLISHED || p->sk->state >= TCP_FIN_WAIT1)  
            return p;  
        p=p->next;  
    }  
    while(p!=(struct sk_buff *)&s->receive_queue);//双向链表，即遍历整个队列  
    return NULL;  
}  
```

看到没，这里accept 函数返回的通信套接字是从监听套接字的 receive_queue 队列中获得的，那么通信套接字与监听套接字的receive_queue队列之间的关系，其中的细节又是什么呢？

其内部实现在 tcp_conn_request 函数中啦：而这个函数在connect 函数下层函数被调用，服务器是被动打开的，即是客户端主动连接。

```text
/* 
 *  This routine handles a connection request. 
 *  It should make sure we haven't already responded. 
 *  Because of the way BSD works, we have to send a syn/ack now. 
 *  This also means it will be harder to close a socket which is 
 *  listening. 
 */  
 /* 
 参数中daddr，saddr的理解应从远端角度出发。所以daddr表示本地地址；saddr表示远端地址 
 seq是函数调用tcp_init_seq()的返回值，表示本地初始化序列号； 
 dev表示接收该数据包的接口设备 
 */  
 //tcp_rcv接收一个syn连接请求数据包后，将调用tcp_con_request函数进行具体处理  
 //其内部逻辑很简单:创建一个新的套接字用于通信，其本身继续监听客户端的请求  
 //创建一个新的套接字得设置其各个字段，这里是复制监听套接字的已有信息，然后根据需要修改部分  
 //然后创建数据包，设置其TCP首部，创建MAC和IP首部，然后回送给客户端，  
 //并把该数据包插入到监听套接字sk的recive_queue队列中，该数据包已经关联了新套接字，  
 //在accept函数中，最后返回的通信套接字则是从这个队列中获得(参见tcp_accept函数)  
static void tcp_conn_request(struct sock *sk, struct sk_buff *skb,  
         unsigned long daddr, unsigned long saddr,  
         struct options *opt, struct device *dev, unsigned long seq)  
{  
    struct sk_buff *buff;  
    struct tcphdr *t1;  
    unsigned char *ptr;  
    struct sock *newsk;  
    struct tcphdr *th;  
    struct device *ndev=NULL;  
    int tmp;  
    struct rtable *rt;  
    
    th = skb->h.th;//获取tcp首部  
  
    /* If the socket is dead, don't accept the connection. */  
    //判断套接字合法性  
    if (!sk->dead)   
    {  
        sk->data_ready(sk,0);//通知睡眠进程，有数据到达  
    }  
    else //无效套接字，已经释放了的  
    {  
        if(sk->debug)  
            printk("Reset on %p: Connect on dead socket.\n",sk);  
        tcp_reset(daddr, saddr, th, sk->prot, opt, dev, sk->ip_tos,sk->ip_ttl);  
        tcp_statistics.TcpAttemptFails++;  
        kfree_skb(skb, FREE_READ);  
        return;  
    }  
  
    /* 
     * Make sure we can accept more.  This will prevent a 
     * flurry of syns from eating up all our memory. 
     */  
   //缓存的未应答数据包个数 >= 最大可缓存个数；表示满了，已经不能接收了  
    if (sk->ack_backlog >= sk->max_ack_backlog)   
    {  
        tcp_statistics.TcpAttemptFails++;  
        kfree_skb(skb, FREE_READ);  
        return;  
    }  
  
    /* 
     * We need to build a new sock struct. 
     * It is sort of bad to have a socket without an inode attached 
     * to it, but the wake_up's will just wake up the listening socket, 
     * and if the listening socket is destroyed before this is taken 
     * off of the queue, this will take care of it. 
     */  
  //创建一个新的套接字  
    newsk = (struct sock *) kmalloc(sizeof(struct sock), GFP_ATOMIC);  
    if (newsk == NULL)   
    {  
        /* just ignore the syn.  It will get retransmitted. */  
        tcp_statistics.TcpAttemptFails++;  
        kfree_skb(skb, FREE_READ);  
        return;  
    }  
   //复制一个套接字结构，即新的套接字中主要信息来源于监听套接字中的已有信息  
    memcpy(newsk, sk, sizeof(*newsk));  
   //下面两个就是把待发送和已接收队列初始化成数据包链表形式  
    skb_queue_head_init(&newsk->write_queue);  
    skb_queue_head_init(&newsk->receive_queue);  
    //下面是重发队列  
    newsk->send_head = NULL;  
    newsk->send_tail = NULL;  
    skb_queue_head_init(&newsk->back_log);//数据包暂存队列(中转站)  
    //新套接字的字段设置  
    newsk->rtt = 0;      /*TCP_CONNECT_TIME<<3*/  
    newsk->rto = TCP_TIMEOUT_INIT;  
    newsk->mdev = 0;  
    newsk->max_window = 0;  
    newsk->cong_window = 1;  
    newsk->cong_count = 0;  
    newsk->ssthresh = 0;  
    newsk->backoff = 0;  
    newsk->blog = 0;  
    newsk->intr = 0;  
    newsk->proc = 0;  
    newsk->done = 0;  
    newsk->partial = NULL;  
    newsk->pair = NULL;  
    newsk->wmem_alloc = 0;  
    newsk->rmem_alloc = 0;  
    newsk->localroute = sk->localroute;  
  
    newsk->max_unacked = MAX_WINDOW - TCP_WINDOW_DIFF;  
  
    newsk->err = 0;  
    newsk->shutdown = 0;  
    newsk->ack_backlog = 0;  
    newsk->acked_seq = skb->h.th->seq+1;  
    newsk->copied_seq = skb->h.th->seq+1;  
    newsk->fin_seq = skb->h.th->seq;  
    newsk->state = TCP_SYN_RECV;  
    newsk->timeout = 0;  
    newsk->ip_xmit_timeout = 0;  
    newsk->write_seq = seq;   
    newsk->window_seq = newsk->write_seq;  
    newsk->rcv_ack_seq = newsk->write_seq;  
    newsk->urg_data = 0;  
    newsk->retransmits = 0;  
    newsk->linger=0;  
    newsk->destroy = 0;  
    init_timer(&newsk->timer);  
    newsk->timer.data = (unsigned long)newsk;  
    newsk->timer.function = &net_timer;  
    init_timer(&newsk->retransmit_timer);  
    newsk->retransmit_timer.data = (unsigned long)newsk;  
    newsk->retransmit_timer.function=&retransmit_timer;  
    newsk->dummy_th.source = skb->h.th->dest;  
    newsk->dummy_th.dest = skb->h.th->source;  
      
    /* 
     *  Swap these two, they are from our point of view.  
     */  
       
    newsk->daddr = saddr;  
    newsk->saddr = daddr;  
  
    put_sock(newsk->num,newsk);//插入 array_sock 哈希表中  
    newsk->dummy_th.res1 = 0;  
    newsk->dummy_th.doff = 6;  
    newsk->dummy_th.fin = 0;  
    newsk->dummy_th.syn = 0;  
    newsk->dummy_th.rst = 0;   
    newsk->dummy_th.psh = 0;  
    newsk->dummy_th.ack = 0;  
    newsk->dummy_th.urg = 0;  
    newsk->dummy_th.res2 = 0;  
    newsk->acked_seq = skb->h.th->seq + 1;//序列号设置  
    newsk->copied_seq = skb->h.th->seq + 1;  
    newsk->socket = NULL;  
  
    /* 
     *  Grab the ttl and tos values and use them  
     */  
  
    newsk->ip_ttl=sk->ip_ttl;  
    newsk->ip_tos=skb->ip_hdr->tos;  
  
    /* 
     *  Use 512 or whatever user asked for  
     */  
  
    /* 
     *  Note use of sk->user_mss, since user has no direct access to newsk  
     */  
    //ip路由表查找表项  
    rt=ip_rt_route(saddr, NULL,NULL);  
      
    if(rt!=NULL && (rt->rt_flags&RTF_WINDOW))  
        newsk->window_clamp = rt->rt_window;  
    else  
        newsk->window_clamp = 0;  
          
    if (sk->user_mss)  
        newsk->mtu = sk->user_mss;  
    else if(rt!=NULL && (rt->rt_flags&RTF_MSS))  
        newsk->mtu = rt->rt_mss - HEADER_SIZE;  
    else   
    {  
#ifdef CONFIG_INET_SNARL    /* Sub Nets Are Local */  
        if ((saddr ^ daddr) & default_mask(saddr))  
#else  
        if ((saddr ^ daddr) & dev->pa_mask)  
#endif  
            newsk->mtu = 576 - HEADER_SIZE;  
        else  
            newsk->mtu = MAX_WINDOW;  
    }  
  
    /* 
     *  But not bigger than device MTU  
     */  
  
    newsk->mtu = min(newsk->mtu, dev->mtu - HEADER_SIZE);  
  
    /* 
     *  This will min with what arrived in the packet  
     */  
  
    tcp_options(newsk,skb->h.th);  
 //服务器端创建新的套接字后，接下来就是创建一个数据包(syn+ack)回送过去  
    buff = newsk->prot->wmalloc(newsk, MAX_SYN_SIZE, 1, GFP_ATOMIC);  
    if (buff == NULL)   
    {  
        sk->err = ENOMEM;  
        newsk->dead = 1;  
        newsk->state = TCP_CLOSE;  
        /* And this will destroy it */  
        release_sock(newsk);  
        kfree_skb(skb, FREE_READ);  
        tcp_statistics.TcpAttemptFails++;  
        return;  
    }  
  //字段设置  
    buff->len = sizeof(struct tcphdr)+4;  
    buff->sk = newsk;//与新套接字关联  
    buff->localroute = newsk->localroute;  
  
    t1 =(struct tcphdr *) buff->data;  
  
    /* 
     *  Put in the IP header and routing stuff.  
     */  
  //MAC 头+ ip 头创建  
    tmp = sk->prot->build_header(buff, newsk->saddr, newsk->daddr, &ndev,  
                   IPPROTO_TCP, NULL, MAX_SYN_SIZE,sk->ip_tos,sk->ip_ttl);  
  
    /* 
     *  Something went wrong.  
     */  
  
    if (tmp < 0)   
    {  
        sk->err = tmp;  
        buff->free = 1;  
        kfree_skb(buff,FREE_WRITE);  
        newsk->dead = 1;  
        newsk->state = TCP_CLOSE;  
        release_sock(newsk);  
        skb->sk = sk;  
        kfree_skb(skb, FREE_READ);  
        tcp_statistics.TcpAttemptFails++;  
        return;  
    }  
  
    buff->len += tmp;  
    t1 =(struct tcphdr *)((char *)t1 +tmp);  
  //tcp首部字段设置  
    memcpy(t1, skb->h.th, sizeof(*t1));  
    buff->h.seq = newsk->write_seq;  
    /* 
     *  Swap the send and the receive.  
     */  
    t1->dest = skb->h.th->source;  
    t1->source = newsk->dummy_th.source;  
    t1->seq = ntohl(newsk->write_seq++);  
    t1->ack = 1;//确认控制位，表示这是一个确认数据包  
    newsk->window = tcp_select_window(newsk);  
    newsk->sent_seq = newsk->write_seq;  
    t1->window = ntohs(newsk->window);  
    t1->res1 = 0;  
    t1->res2 = 0;  
    t1->rst = 0;  
    t1->urg = 0;  
    t1->psh = 0;  
    t1->syn = 1;//同步控制位置位，和ack位一起作用  
    t1->ack_seq = ntohl(skb->h.th->seq+1);//确认序列号=发过来的数据包的序列号+1  
    t1->doff = sizeof(*t1)/4+1;  
    ptr =(unsigned char *)(t1+1);  
    ptr[0] = 2;  
    ptr[1] = 4;  
    ptr[2] = ((newsk->mtu) >> 8) & 0xff;  
    ptr[3] =(newsk->mtu) & 0xff;  
  
    //下面是tcp校验和检查  
    tcp_send_check(t1, daddr, saddr, sizeof(*t1)+4, newsk);  
    //调用_queue_xmit函数发送(前面介绍过)  
    newsk->prot->queue_xmit(newsk, ndev, buff, 0);  
    reset_xmit_timer(newsk, TIME_WRITE , TCP_TIMEOUT_INIT);  
    skb->sk = newsk;//这里数据包捆绑的就是新的套接字了  
  
    /* 
     *  Charge the sock_buff to newsk.  
     */  
    //原监听套接字接收队列中存放的字节数减去该数据包大小  
    //新创建的通信套接字则加上该数据包大小  
    sk->rmem_alloc -= skb->mem_len;  
    newsk->rmem_alloc += skb->mem_len;  
      
    //把这个数据包插入到reveive_queue中，这个数据包的宿主是新套接字  
    //在accept函数中，通信套接字则是从这个队列中获取  
    skb_queue_tail(&sk->receive_queue,skb);  
    sk->ack_backlog++;//缓存的数据包个数+1  
    release_sock(newsk);  
    tcp_statistics.TcpOutSegs++;  
}  
```

经过上面这个函数，我们得到了一个重要信息，就是客户端向服务器发出连接请求后，服务器端新建了通信套接字和确认数据包，二者建立关联，并把确认数据包插入到监听套接字的receive_queue 队列中，该数据包的宿主就是新建的通信套接字，而accept函数返回的通信套接字则在监听套接字的 receive_queue 队列中获得。好了，至此整个tcp连接建立过程算是剖析完毕了。

----

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549428366