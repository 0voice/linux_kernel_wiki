## 1.前言

本文分享了Linux内核网络数据包发送在UDP协议层的处理，主要分析了udp_sendmsg和udp_send_skb函数，并分享了UDP层的数据统计和监控以及套接字发送大小的调优。

## 2.udp_sendmsg 

这个函数定义在net / ipv4 / udp.c，函数很长，分段来看。

### 2.1 UDP插入

UDP udp_sendmsg corking是一项优化技术，允许内核将多个数据累积成一体的数据报发送。在用户程序中有两种方法可以启用此选项：

* 使用 setsockopt 系统调用设置socket的 UDP_CORK 选项
* 程序调用 send，sendto 或 sendmsg 时，带 MSG_MORE 参数

udp_sendmsg 代码检查 up->pending 套接字socket当前是否已被塞住（corked），如果是，则直接跳到 do_append_data 进行数据追加（append）。

```c
int udp_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
                size_t len)
{

    /* variables and error checking ... */

  fl4 = &inet->cork.fl.u.ip4;
  if (up->pending) {
          /*
           * There are pending frames.
           * The socket lock must be held while it's corked.
           */
          lock_sock(sk);
          if (likely(up->pending)) {
                  if (unlikely(up->pending != AF_INET)) {
                          release_sock(sk);
                          return -EINVAL;
                  }
                  goto do_append_data;
          }
          release_sock(sk);
  }

```

## 2.2获取目的IP地址和端口

接下来获取目的IP地址和端口，有两个可能的来源：

* 如果之前socket已经建立连接，那socket本身就存储了目标地址
* 地址通过辅助结构（struct msghdr）预测，逐步我们在 sendto 的内核代码中看到的那样

具体逻辑：

```c
/*
 *      Get and verify the address.
 */
  if (msg->msg_name) {
          struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
          if (msg->msg_namelen < sizeof(*usin))
                  return -EINVAL;
          if (usin->sin_family != AF_INET) {
                  if (usin->sin_family != AF_UNSPEC)
                          return -EAFNOSUPPORT;
          }

          daddr = usin->sin_addr.s_addr;
          dport = usin->sin_port;
          if (dport == 0)
                  return -EINVAL;
  } else {
          if (sk->sk_state != TCP_ESTABLISHED)
                  return -EDESTADDRREQ;
          daddr = inet->inet_daddr;
          dport = inet->inet_dport;
          /* Open fast path for connected socket.
             Route will not be used, if at least one option is set.
           */
          connected = 1;
  }

```

UDP代码中出现了 TCP_ESTABLISHED！UDP套接字的状态使用了TCP状态来描述。的上面显示代码了内核如何解析该变量以便设置 daddr 状语从句：  dport。

如果没有 struct msghdr 变量，内核函数到达 udp_sendmsg 函数时，会从socket本身检索目的地址和端口，插入socket标记为“已连接”。

### 2.3套接字发送：簿记和打合并

接下来，获取存储在插座上的源地址，设备索引（设备索引）和时间戳选项（例如SOCK_TIMESTAMPING_TX_HARDWARE，  SOCK_TIMESTAMPING_TX_SOFTWARE，  SOCK_WIFI_STATUS）：

```c
ipc.addr = inet->inet_saddr;

ipc.oif = sk->sk_bound_dev_if;

sock_tx_timestamp(sk, &ipc.tx_flags);
```

### 2.4辅助消息（辅助消息）

除了发送或接收数据包之外，sendmsg 状语从句： recvmsg 系统-调用还网求允许用户设置或请求辅助数据。程序用户可以通过将请求信息组织分类中翻译 struct msghdr 类型变量来利用此辅助数据。一些辅助数据类型记录在IP手册页中。

辅助数据的一个常见例子是 IP_PKTINFO，对于 sendmsg，IP_PKTINFO 网求允许程序在发送数据时设置一个 in_pktinfo 变量。可以程序通过填写回复时 struct in_pktinfo 变量中的字段来指定要在包上使用的源地址。如果程序是监听多个IP地址的服务端程序，那这是一个很有用的选项。在这种情况下，服务端可能想使用客户端连接服务端的那个IP地址来回复客户端，IP_PKTINFO 非常适合这种场景。

setsockopt 可以在套接字级别设置发送包的IP_TTL和IP_TOS。而辅助消息允许在每个数据包级别设置TTL和TOS值。时从qdisc中发送出去。

可以看到内核如何在UDP套接字上处理 sendmsg 的辅助消息：

```c
if (msg->msg_controllen) {
        err = ip_cmsg_send(sock_net(sk), msg, &ipc,
                           sk->sk_family == AF_INET6);
        if (err)
                return err;
        if (ipc.opt)
                free = 1;
        connected = 0;
}
```

解析辅助消息的工作是由 ip_cmsg_send 完成的，定义在net / ipv4 / ip_sockglue.c。传递一个未初始化的辅助数据，将会把这个套接字标记为“未建立连接的”。

### 2.5设置自定义IP选项

接下来，sendmsg 将检查用户是否通过辅助消息设置了任何自定义IP选项。如果设置了，将使用这些自定义值；如果没有，那就使用socket中（已经在用）的参数：

```c
if (!ipc.opt) {
        struct ip_options_rcu *inet_opt;

        rcu_read_lock();
        inet_opt = rcu_dereference(inet->inet_opt);
        if (inet_opt) {
                memcpy(&opt_copy, inet_opt,
                       sizeof(*inet_opt) + inet_opt->opt.optlen);
                ipc.opt = &opt_copy.opt;
        }
        rcu_read_unlock();
}
```

接下来，该函数检查是否设置了源记录路由（源记录路由，SRR）IP选项。SRR有两种类型：宽松的源记录路由和严格的源记录路由。地址串联其保存到 faddr，插入套接字标记为“未连接”。这将在后面用到：

```c
ipc.addr = faddr = daddr;

if (ipc.opt && ipc.opt->opt.srr) {
        if (!daddr)
                return -EINVAL;
        faddr = ipc.opt->opt.faddr;
        connected = 0;
}
```

处理完SRR选项后，将处理TOS选项，这可以从辅助消息中获取，或者从socket当前值中获取。然后检查：

1、是否（使用 setsockopt）在插座上设置了 SO_DONTROUTE，或<br>
2、是否（调用 sendto 或 sendmsg 时）指定了 MSG_DONTROUTE 标志，或<br>
3、是否已设置了 is_strictroute，表示需要严格的SRR任何一个为真，tos 扩展的 RTO_ONLINK 位将置1，并且socket被视为“未连接”：<br>

```c
tos = get_rttos(&ipc, inet);
if (sock_flag(sk, SOCK_LOCALROUTE) ||
    (msg->msg_flags & MSG_DONTROUTE) ||
    (ipc.opt && ipc.opt->opt.is_strictroute)) {
        tos |= RTO_ONLINK;
        connected = 0;
}
```

### 2.6多播或单播（多播或单播）

这有点复杂，因为用户可以通过 IP_PKTINFO 辅助消息来指定发送包的源地址或设备号，如前所述。

如果目标地址是多播地址：<br>
1、将多播设备（设备）的索引（索引）设置为发送（写）这个数据包的设备索引，并且<br>
2、数据包的源地址将设置为多播<br>

如果目标地址不是一个收件人地址，则发送数据包的设备准备为 inet->uc_index（单播），除非用户使用 IP_PKTINFO 辅助消息覆盖了它。

```c
if (ipv4_is_multicast(daddr)) {
        if (!ipc.oif)
                ipc.oif = inet->mc_index;
        if (!saddr)
                saddr = inet->mc_addr;
        connected = 0;
} else if (!ipc.oif)
        ipc.oif = inet->uc_index;

```

### 2.7路由

现在开始路由，UDP层中处理路由的代码以快速路径（fast path）开始。如果套接字已连接，则直接尝试获取路由：

```c
if (connected)
        rt = (struct rtable *)sk_dst_check(sk, 0);
```

如果socket未连接，或者虽然已经连接，但sk_dst_check 路由辅助 功能已确定路由已过期，则代码将进入慢速路径（slow path）以生成一条路由记录。首先调用 flowi4_init_output 构造一个描述此UDP流的变量：

```c
if (rt == NULL) {
        struct net *net = sock_net(sk);

        fl4 = &fl4_stack;
        flowi4_init_output(fl4, ipc.oif, sk->sk_mark, tos,
                           RT_SCOPE_UNIVERSE, sk->sk_protocol,
                           inet_sk_flowi_flags(sk)|FLOWI_FLAG_CAN_SLEEP,
                           faddr, saddr, dport, inet->inet_sport);

````

然后，socket为此流程实例实例传递给安全实例，这样SELinux或SMACK这样的系统就可以在流程实例上设置安全ID。接下来，ip_route_output_flow 将调用IP路由代码，创建一个路由实例：

```c
security_sk_classify_flow(sk, flowi4_to_flowi(fl4));
rt = ip_route_output_flow(net, fl4, sk);
```

如果创建路由实例失败，并且返回码是 ENETUNREACH，则 OUTNOROUTES 计数器将会加1。

```c
if (IS_ERR(rt)) {
  err = PTR_ERR(rt);
  rt = NULL;
  if (err == -ENETUNREACH)
    IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
  goto out;
}
```

这些统计计数器所在的源文件，其他可用的计数器及其含义，将在下面的UDP监控部分共享。

接下来，如果是广播路由，但socket的 SOCK_BROADCAST 选项未设置，则处理过程终止。如果socket被视为“已连接”，则路由实例将缓存到socket上：

```c
err = -EACCES;
if ((rt->rt_flags & RTCF_BROADCAST) &&
    !sock_flag(sk, SOCK_BROADCAST))
        goto out;
if (connected)
        sk_dst_set(sk, dst_clone(&rt->dst));
```

### 2.8 MSG_CONFIRM：阻止ARP缓存过期

如果调用 send， sendto 或 sendmsg 的时候指定了 MSG_CONFIRM 参数，UDP协议层将会如下处理：

```c
if (msg->msg_flags&MSG_CONFIRM)
          goto do_confirm;
back_from_confirm:
```

该标志提示系统去确认一下ARP缓存缓存是否仍然有效，防止其被垃圾回收。 do_confirm 标签位于此函数末尾处：

```c
do_confirm:
        dst_confirm(&rt->dst);
        if (!(msg->msg_flags&MSG_PROBE) || len)
                goto back_from_confirm;
        err = 0;
        goto out;
```

dst_confirm 函数只是在相应的缓存缓存上设置一个标记位，稍后当查询邻居缓存并找到放置时将检查该标志，我们后面一些会看到。此功能通常用于UDP网络应用程序，以减少不必要的ARP流量。此代码确认缓存并然后跳回 back_from_confirm 标签。一旦 do_confirm 代码跳回到 back_from_confirm（或者之前就没有执行到 do_confirm ），代码接下来将处理UDP软木塞和未塞木塞的情况。

### 2.9 uncorked UDP套接字快速路径：准备待发送数据

如果不需要corking，数据就可以封装到一个 struct sk_buff 实例中并传递给 udp_send_skb，离IP协议层更进了一步。这是通过调用 ip_make_skb 来完成的。

先前通过调用ip_route_output_flow 生成的路由可能也会 一起传进来，并保存到skb里。

```c
/* Lockless fast path for the non-corking case. */
if (!corkreq) {
        skb = ip_make_skb(sk, fl4, getfrag, msg->msg_iov, ulen,
                          sizeof(struct udphdr), &ipc, &rt,
                          msg->msg_flags);
        err = PTR_ERR(skb);
        if (!IS_ERR_OR_NULL(skb))
                err = udp_send_skb(skb, fl4);
        goto out;
}
```

ip_make_skb 函数将创建一个skb，其中需要考虑到很多的事情，例如：

* MTU
* UDP阻塞（如果启用）
* UDP分片卸载（UFO）
* Fragmentation（分片）：如果硬件不支持UFO，但是要传输的数据大于MTU，需要软件做分片

大多数网络设备驱动程序不支持UFO，因为网络硬件本身不支持此功能。

#### 2.9.1 ip_make_skb

定义在net / ipv4 / ip_output.c，这个函数有点复杂。

构建SKB的时候，ip_make_skb 依赖的底层代码需要使用一个加塞变量和一个队列变量，SKB将通过队列变量传入。如果插座未被软木，则会传入一个假的加塞变量和一个空队列。

现在来看看假corking变量和空数值是如何初始化的：

```c
struct sk_buff *ip_make_skb(struct sock *sk, /* more args */)
{
        struct inet_cork cork;
        struct sk_buff_head queue;
        int err;

        if (flags & MSG_PROBE)
                return NULL;

        __skb_queue_head_init(&queue);

        cork.flags = 0;
        cork.addr = 0;
        cork.opt = NULL;
        err = ip_setup_cork(sk, &cork, /* more args */);
        if (err)
                return ERR_PTR(err);
```

如上所示，cork和queue都是在栈上分配的，ip_make_skb 根本不需要它。 ip_setup_cork 初始化cork变量。然后，调用__ip_append_data 并调用cork和queue变量：

```c
err = __ip_append_data(sk, fl4, &queue, &cork,
                       &current->task_frag, getfrag,
                       from, length, transhdrlen, flags);

```

我们将在后面看到这个函数是如何工作的，因为无论socket是否被软木塞，最后都会执行它。

现在，我们只需要知道__ip_append_data 将创建一个skb，向其追加数据，转化为该skb添加到队列的队列变量中。如果追加数据失败，则__ip_flush_pending_frame 调用数据并向上返回错误（指针类型）：

```c
if (err) {
        __ip_flush_pending_frames(sk, &queue, &cork);
        return ERR_PTR(err);
}
```

最后，如果没有发生错误，__ip_make_skb 将skb出队，添加IP选项，并返回一个准备好传递给更连续发送的skb：

```
return __ip_make_skb(sk, fl4, &queue, &cork);
```

#### 2.9.2发送数据

如果没有错误，skb将会交给 udp_send_skb，附属会继续将其传给下一层协议，IP协议：

```c
err = PTR_ERR(skb);
if (!IS_ERR_OR_NULL(skb))
        err = udp_send_skb(skb, fl4);
goto out;
```

如果有错误，错误计数就会有相应增加。后面的“错误计数”部分会详细介绍。

### 2.10没有被软木的数据时的慢路径

如果使用了UDP corking，但之前没有数据被cork，则慢路径开始：

对插座加锁

检查应用程序是否有bug：已经被cork的socket是否再次被cork

设置该UDP流的一些参数，为corking做准备

将要发送的数据追加到现有数据

udp_sendmsg 代码继续向下看，就是这一逻辑：

```c
lock_sock(sk);
  if (unlikely(up->pending)) {
          /* The socket is already corked while preparing it. */
          /* ... which is an evident application bug. --ANK */
          release_sock(sk);

          LIMIT_NETDEBUG(KERN_DEBUG pr_fmt("cork app bug 2\n"));
          err = -EINVAL;
          goto out;
  }
  /*
   *      Now cork the socket to pend data.
   */
  fl4 = &inet->cork.fl.u.ip4;
  fl4->daddr = daddr;
  fl4->saddr = saddr;
  fl4->fl4_dport = dport;
  fl4->fl4_sport = inet->inet_sport;
  up->pending = AF_INET;

do_append_data:
  up->len += ulen;
  err = ip_append_data(sk, fl4, getfrag, msg->msg_iov, ulen,
                       sizeof(struct udphdr), &ipc, &rt,
                       corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
                       
```

#### 2.10.1 ip_append_data

这个函数简单封装了__ip_append_data，在调用之前之前，做了两件重要的事情：

检查是否从用户指定了 MSG_PROBE 标志。该标志表示用户不想真正发送数据，只是做路径探测（例如，确定PMTU）

检查socket的发送物理是否为空。如果为空，则意味着没有cork数据等待处理，因此调用ip_setup_cork 来设置corking 

一旦处理了上述条件，就调用__ip_append_data 函数，该函数包含用于将数据处理成数据包的大量逻辑。

#### 2.10.2 __ip_append_data

如果插座是塞住，则从 ip_append_data 调用此函数;如果插座未被软木，则从 ip_make_skb 。调用此函数在任何一种情况下，函数都将分配一个新缓冲区来存储传入的数据，或者将数据附加到现有数据中。这种工作的方式围绕socket的发送序列。等待发送的现有数据（例如，如果socket被软木塞）将在一部分中有一个对应的对应，可以被追加数据。

这个函数很复杂，它执行很多计算逐步如何构造传递给下面的网络层的skb。

该函数的重点包括：

如果硬件支持，则处理UDP分段卸载（UFO）。绝大多数网络硬件不支持UFO。如果你的网卡驱动程序支持它，设置它将 NETIF_F_UFO 标记位

处理支持分散/收集（分散/聚集）IO的网卡许多卡都支持此功能，并使用。 NETIF_F_SG 标志进行通告支持该特性的网卡可以处理数据被分散到多个缓冲器的数据包;内核不需要花时间将避免这种额外的复制会提升性能，大多数网卡都支持此功能

通过调用sock_wmalloc 跟踪发送帧 的大小。当分配新的skb时，skb的大小由创建它的socket计费（charge），并计入socket发送的已分配字节数。 （超过计费限制），则skb并分配失败并返回错误。我们将在下面的调优部分中看到如何设置socket发送大小（txqueuelen）

更新错误统计信息。此函数中的任何错误都会增加“丢弃”计数。

函数执行成功后返回0，以及一个适用于网络设备传输的skb。

在unorked情况下，持有skb的队列被作为参数传递给上面描述的__ip_make_skb，在那里它被出队并通过 udp_send_skb 发送到更轻松。

在软木的情况下，__ip_append_data 的返回值向上传递。数据位于发送队列中，直到 udp_sendmsg 确定的英文时候调用 udp_push_pending_frames 来完成SKB，后者会进一步调用 udp_send_skb。

#### 2.10.3冲洗软木塞

现在，udp_sendmsg 会继续，检查__ip_append_skb 的返回值（错误码）：

```c
if (err)
        udp_flush_pending_frames(sk);
else if (!corkreq)
        err = udp_push_pending_frames(sk);
else if (unlikely(skb_queue_empty(&sk->sk_write_queue)))
        up->pending = 0;
release_sock(sk);
```

我们来看看每个情况：

如果出现错误（错误为非零），则调用 udp_flush_pending_frames，这将取消cork并从socket的发送数据中删除所有数据

如果在未指定 MSG_MORE 的情况下发送此数据，则调用 udp_push_pending_frames，则数据传递到更下面的网络层

如果发送轴向为空，替换套接字标记为不再软木

如果追加操作完成并且有更多数据要进入cork，则代码将做一些清理工作，并返回追加数据的长度：

```c
ip_rt_put(rt);
if (free)
        kfree(ipc.opt);
if (!err)
        return len;
```

这就是内核如何处理

### 2.11错误会计

如果：

non-corking快速路径创建skb失败，或 udp_send_skb 返回错误，或

ip_append_data 无法将数据附加到corked UDP套接字，或

当 udp_push_pending_frames 调用 udp_send_skb 发送corked skb时间返回错误

仅当返回的错误是 ENOBUFS（内核无可用内存）或套接字已设置 SOCK_NOSPACE（发送串行已满）时，SNDBUFERRORS 统计信息才会增加：

```c
/*
 * ENOBUFS = no kernel mem, SOCK_NOSPACE = no sndbuf space.  Reporting
 * ENOBUFS might not be good (it's not tunable per se), but otherwise
 * we don't have a good statistic (IpOutDiscards but it can be too many
 * things).  We could add another new stat but at least for now that
 * seems like overkill.
 */
if (err == -ENOBUFS || test_bit(SOCK_NOSPACE, &sk->sk_socket->flags)) {
        UDP_INC_STATS_USER(sock_net(sk),
                        UDP_MIB_SNDBUFERRORS, is_udplite);
}
return err;
```

我们接下来会在后面的数据监控里看到如何读取这些计数。

### 3. udp_send_skb 

udp_sendmsg 通过调用 udp_send_skb 函数将skb转到下一网络层，在这里中是IP协议层。

向skb添加UDP头

处理校验和：软件校验和，硬件校验和或无校​​验和（如果替换）

调用 ip_send_skb 将skb发送到IP协议层

更新发送成功或失败的统计计数器

首先，创建UDP头：

```c
static int udp_send_skb(struct sk_buff *skb, struct flowi4 *fl4)
{
                /* useful variables ... */

        /*
         * Create a UDP header
         */
        uh = udp_hdr(skb);
        uh->source = inet->inet_sport;
        uh->dest = fl4->fl4_dport;
        uh->len = htons(len);
        uh->check = 0;
```

接下来，处理校验和。有几种情况：

首先处理UDP-Lite验证和

接下来，如果socket校正和选项被关闭（setsockopt 带 SO_NO_CHECK 参数），则被标记为校验和关闭

接下来，如果硬件支持UDP校验和，则将调用 udp4_hwcsum 来设置它。请注意，如果数据包是分段的，内核将在软件中生成校正和，可以在udp4_hwcsum的源代码中看到这一点

最后，通过调用 udp_csum 生成软件校验和

```c
if (is_udplite)                                  /*     UDP-Lite      */
        csum = udplite_csum(skb);

else if (sk->sk_no_check == UDP_CSUM_NOXMIT) {   /* UDP csum disabled */

        skb->ip_summed = CHECKSUM_NONE;
        goto send;

} else if (skb->ip_summed == CHECKSUM_PARTIAL) { /* UDP hardware csum */

        udp4_hwcsum(skb, fl4->saddr, fl4->daddr);
        goto send;

} else
        csum = udp_csum(skb);
```

接下来，添加了伪头：

```c
uh->check = csum_tcpudp_magic(fl4->saddr, fl4->daddr, len,
                              sk->sk_protocol, csum);
if (uh->check == 0)
        uh->check = CSUM_MANGLED_0;
```

如果校验和为0，则根据RFC 768，校验为全1（全部传输（等效于补码算术））。最后，将skb传递给IP协议层并增加统计计数：

```c
send:
  err = ip_send_skb(sock_net(sk), skb);
  if (err) {
          if (err == -ENOBUFS && !inet->recverr) {
                  UDP_INC_STATS_USER(sock_net(sk),
                                     UDP_MIB_SNDBUFERRORS, is_udplite);
                  err = 0;
          }
  } else
          UDP_INC_STATS_USER(sock_net(sk),
                             UDP_MIB_OUTDATAGRAMS, is_udplite);
  return err;
```

如果 ip_send_skb 成功，将更新 OUTDATAGRAMS 统计。如果IP协议层报告错误，并且错误是 ENOBUFS（内核内核内存）而且错误queue（inet->recverr）没有启用，则更新 SNDBUFERRORS。

接下来看看如何在Linux内核中监视和调优UDP协议层。

### 4.监控：UDP层统计

两个非常有用的获取UDP协议统计文件：

```
/proc/net/snmp
/proc/net/udp
```

#### 4.1 / proc / net / snmp

监控UDP协议层统计：

    cat /proc/net/snmp | grep Udp\:

要准确地理解这些计数，需要仔细地阅读内核代码。一些类型的错误计数并不是只出现在一种计数中，而可能是出现在多个计数中。

* InDatagrams：当userland程序使用recvmsg读取数据报时增加。当UDP数据包被封装并发回进行处理时，该值也会增加。
* NoPorts：当UDP数据包到达没有程序正在侦听的端口时增加。
* InErrors：在以下几种情况下增加：在接收队列中没有内存，看到错误的校验和，以及sk_add_backlog无法添加数据报。
* OutDatagrams：当将UDP数据包无误传递到要发送的IP协议层时增加。
* RcvbufErrors：当sock_queue_rcv_skb报告没有可用的内存时增加；如果sk-> sk_rmem_alloc大于或等于sk-> sk_rcvbuf，则会发生这种情况。
* SndbufErrors：如果IP协议层在尝试发送数据包时报告错误，并且未设置错误队列，则增加。如果没有发送队列空间或内核内存可用，也将增加。
* InCsumErrors：当检测到UDP校验和失败时增加。请注意，在所有情况下，我都能发现InCsumErrors与InErrors同时增加。因此，InErrors-InCsumErros应该在接收端产生与内存相关的错误计数。

UDP协议层发现的某些错误会出现在其他协议层的统计信息中。一个例子：路由错误。 udp_sendmsg 发现的路由错误将导致IP协议层的 OutNoRoutes 统计增加。

#### 4.2 / proc / net / udp

监控UDP套接字统计：

    cat /proc/net/udp

每一列的意思：
* sl：套接字的内核哈希槽
* local_address：套接字的十六进制本地地址和端口号，以：分隔。
* rem_address：套接字的十六进制远程地址和端口号，以：分隔。
* st：套接字的状态。奇怪的是，UDP协议层似乎使用了某些TCP套接字状态。在上面的示例中，7是TCP_CLOSE。
* tx_queue：在内核中为传出UDP数据报分配的内存量。
* rx_queue：在内核中为传入的UDP数据报分配的内存量。
* tr，tm-> when，retrnsmt：UDP协议层未使用这些字段。
* uid：创建此套接字的用户的有效用户ID。
* timeout：未由UDP协议层使用。
* inode：与此套接字相对应的inode编号。您可以使用它来帮助您确定哪个用户进程打开了此套接字。检查/ proc / [pid] / fd，其中将包含指向socket [：inode]的符号链接。
* ref：套接字的当前引用计数。
* pointer：struct sock内核中的内存地址。
* drops：与此套接字关联的数据报丢弃数。请注意，这不包括与发送数据报有关的任何丢弃（在已塞好的UDP套接字上或以其他方式）；从本博客文章所检查的内核版本开始，该值仅在接收路径中递增。

打印这些计数的代码在net / ipv4 / udp.c。

### 5.调优：socket发送收发内存大小

发送副本（也叫“写类别”）的副本可以通过设置 net.core.wmem_max sysctl 进行修改。

    $ sudo sysctl -w net.core.wmem_max=8388608

sk->sk_write_queue 用 net.core.wmem_default 初始化，这个值也可以调整。

调整初始发送缓冲区大小：

    $ sudo sysctl -w net.core.wmem_default=8388608

也可以通过从应用程序调用 setsockopt 并传递 SO_SNDBUF 来设置 sk->sk_write_queue 。通过 setsockopt 设置的可选是 net.core.wmem_max。

不过，可以通过 setsockopt 并传递传递 SO_SNDBUFFORCE 覆盖 net.core.wmem_max 限制，这需要 CAP_NET_ADMIN 权限。

每次调用__ip_append_data 分配skb时，sk->sk_wmem_alloc 都会增长。预先我们所看到的，UDP数据报传输速度很快，通常不会在发送当中花费了太多时间。

### 6.总结

本文重点分析了数据包在传输层（UDP协议）的发送过程，并进行了监控和调优，后面的数据包将到达IP协议层，下次再分享，感谢阅读。






