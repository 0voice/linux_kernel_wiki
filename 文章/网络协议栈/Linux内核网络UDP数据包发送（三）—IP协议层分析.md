## 1. 前言
Linux内核网络 UDP 协议层通过调用 ip_send_skb 将 skb 交给 IP 协议层，本文通过分析内核 IP 协议层的关键函数来分享内核数据包发送在 IP 协议层的处理，并分享了监控IP层的方法。

## 2. ip_send_skb

ip_send_skb 函数定义在 net/ipv4/ip_output.c 中，非常简短。它只是调用ip_local_out，如果调用失败，就更新相应的错误计数：
```c
int ip_send_skb(struct net *net, struct sk_buff *skb)
{
        int err;

        err = ip_local_out(skb);
        if (err) {
                if (err > 0)
                        err = net_xmit_errno(err);
                if (err)
                        IP_INC_STATS(net, IPSTATS_MIB_OUTDISCARDS);
        }

        return err;
}
```

net_xmit_errno 函数将低层错误转换为 IP 和 UDP 协议层所能理解的错误。如果发生错误， IP 协议计数器 OutDiscards 会递增。稍后我们将看到读取哪些文件可以获取此统计信息。接下来看 ip_local_out。

## 3. ip_local_out and __ip_local_out

ip_local_out 和__ip_local_out 都很简单。ip_local_out 只需调用__ip_local_out，如果返回值为 1，则调用路由层 dst_output 发送数据包：

```c
int ip_local_out(struct sk_buff *skb)
{
        int err;

        err = __ip_local_out(skb);
        if (likely(err == 1))
                err = dst_output(skb);

        return err;
}
```

接下来看__ip_local_out 的代码：
```c
int __ip_local_out(struct sk_buff *skb)
{
        struct iphdr *iph = ip_hdr(skb);

        iph->tot_len = htons(skb->len);
        ip_send_check(iph);
        return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,
                       skb_dst(skb)->dev, dst_output);
}
```

可以看到，该函数首先做了两件重要的事情：<br>
设置 IP 数据包的长度<br>
调用 ip_send_check 来计算要写入 IP 头的校验和。ip_send_check 函数将进一步调用名为 ip_fast_csum 的函数来计算校验和。在 x86 和 x86_64 体系结构上，此函数用汇编实 现。<br>
接下来，IP 协议层将通过调用 nf_hook 进入 netfilter，其返回值将传递回 ip_local_out 。如果 nf_hook 返回 1，则表示允许数据包通过，并且调用者应该自己发送数据包。这正是我们在上面看到的情况：ip_local_out 检查返回值 1 时，自己通过调用 dst_output 发送数据包。

### 3.1 netfilter and nf_hook

nf_hook 只是一个 wrapper，它调用 nf_hook_thresh，首先检查是否有为这个协议族和hook 类型（这里分别为 NFPROTO_IPV4 和 NF_INET_LOCAL_OUT）安装的过滤器，然后将返回到 IP 协议层，避免深入到 netfilter 或更下面，比如 iptables 和 conntrack。

如果有非常多或者非常复杂的 netfilter 或 iptables 规则，那些规则将在触发 sendmsg 系统调的用户进程的上下文中执行。如果对这个用户进程设置了 CPU 亲和性，相应的 CPU 将花费系统时间（system time）处理出站（outbound）iptables 规则。如果做性能回归测试，那可能要考虑根据系统的负载，将相应的用户进程绑到到特定的 CPU，或者是减少 netfilter/iptables 规则的复杂度，以减少对性能测试的影响。

出于讨论目的，我们假设 nf_hook 返回 1，表示调用者（在这种情况下是 IP 协议层）应该自己发送数据包。

### 3.2 目的（路由）缓存

dst 代码在 Linux 内核中实现协议无关的目标缓存。为了继续学习发送 UDP 数据报的流程 ，我们需要了解 dst 条目是如何被设置的，首先来看 dst 条目和路由是如何生成的。目标缓存，路由和邻居子系统，任何一个都可以拿来单独详细的介绍。现在不深入细节，只是快速地看一下它们是如何组合到一起的。
上面看到的代码调用了 dst_output(skb)。此函数只是查找关联到这个 skb 的 dst 条目 ，然后调用 output 方法。代码如下：

```c
/* Output packet to network from transport.  */
static inline int dst_output(struct sk_buff *skb)
{
        return skb_dst(skb)->output(skb);
}
```
看起来很简单，但是 output 方法之前是如何关联到 dst 条目的？<br>
首先很重要的一点，目标缓存条目是以多种不同方式添加的。到目前为止，我们已经在代码中看到的一种方法是从 udp_sendmsg 调用ip_route_output_flow。ip_route_output_flow 函数调用 __ip_route_output_key ，后者进而调用 __mkroute_output。 __mkroute_output 函数创建路由和目标缓存条目。当它执行创建操作时，它会判断哪个 output 方法适合此 dst。大多数时候，这个函数是 ip_output。

## 4. ip_output
在 UDP IPv4 情况下，上面的 output 方法指向的是 ip_output:
```c
int ip_output(struct sk_buff *skb)
{
        struct net_device *dev = skb_dst(skb)->dev;

        IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUT, skb->len);

        skb->dev = dev;
        skb->protocol = htons(ETH_P_IP);

        return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
                            ip_finish_output,
                            !(IPCB(skb)->flags & IPSKB_REROUTED));
}
```
首先，更新 IPSTATS_MIB_OUT 统计计数。IP_UPD_PO_STATS 宏将更新字节数和包数统计。接下来，设置要发送此 skb 的设备，以及协议。<br>
最后，通过调用 NF_HOOK_COND 将控制权交给 netfilter。查看 NF_HOOK_COND 的函数原型 有助于更清晰地解释它如何工作：

```c
static inline int
NF_HOOK_COND(uint8_t pf, unsigned int hook, struct sk_buff *skb,
             struct net_device *in, struct net_device *out,
             int (*okfn)(struct sk_buff *), bool cond)
```
NF_HOOK_COND 通过检查传入的条件来工作。在这里条件是!(IPCB(skb)->flags & IPSKB_REROUTED。如果此条件为真，则 skb 将发送给 netfilter。如果 netfilter 允许包通过 ，okfn 回调函数将被调用。在这里，okfn 是 ip_finish_output。

## 5. ip_finish_output
```c
static int ip_finish_output(struct sk_buff *skb)
{
#if defined(CONFIG_NETFILTER) && defined(CONFIG_XFRM)
        /* Policy lookup after SNAT yielded a new policy */
        if (skb_dst(skb)->xfrm != NULL) {
                IPCB(skb)->flags |= IPSKB_REROUTED;
                return dst_output(skb);
        }
#endif
        if (skb->len > ip_skb_dst_mtu(skb) && !skb_is_gso(skb))
                return ip_fragment(skb, ip_finish_output2);
        else
                return ip_finish_output2(skb);
}
```
如果内核启用了 netfilter 和数据包转换（XFRM），则更新 skb 的标志并通过 dst_output 将 其发回。<br>

更常见的两种情况是：<br>
1、如果数据包的长度大于 MTU 并且分片不会 offload 到设备，则会调用 ip_fragment 在发送之前对数据包进行分片<br>
2、否则，数据包将直接发送到 ip_finish_output2<br>

**Path MTU Discovery**<br>

Linux 提供了一个功能：路径 MTU 发现 。此功能允许内核自动确定路由的最大传输单元（ MTU ）。发送小于或等于该路由的 MTU 的包意味着可以避免 IP 分片，这是推荐设置，因为数据包分片会消耗系统资源，而避免分片看起来很容易：只需发送足够小的不需要分片的数据包。

可以在应用程序中通过调用 setsockopt 带 SOL_IP 和 IP_MTU_DISCOVER 选项，在 packet 级别来调整路径 MTU 发现设置，相应的合法值参考 IP 协议的man page。例如，可能想设置的值是 ：IP_PMTUDISC_DO，表示“始终执行路径 MTU 发现”。更高级的网络应用程序或诊断工具可能选择自己实现RFC 4821，以在应用程序启动时针对特定的路由做 PMTU。在这种情况下，可以使用 IP_PMTUDISC_PROBE 选项告诉内核设置“Do not Fragment”位，这就会允许发送大于 PMTU 的数据。

应用程序可以通过调用 getsockopt 带 SOL_IP 和 IP_MTU 选项来查看当前 PMTU。可以使用它指导应用程序在发送之前，构造 UDP 数据报的大小。

如果已启用 PMTU 发现，则发送大于 PMTU 的 UDP 数据将导致应用程序收到 EMSGSIZE 错误。这种情况下，应用程序只能减小 packet 大小重试。

强烈建议启用 PTMU 发现，当查看 IP 协议层统计信息时，将解释所有统计信息，包括与分片相关的统计信息。其中许多计数都在 ip_fragment 中更新的。不管分片与否，代码最后都会调到 ip_finish_output2。

## 6. ip_finish_output2

IP 分片后调用 ip_finish_output2，另外 ip_finish_output 也会直接调用它。这个函数在将包发送到邻居缓存之前处理各种统计计数器：
```c
static inline int ip_finish_output2(struct sk_buff *skb)
{

        /* variable declarations */

        if (rt->rt_type == RTN_MULTICAST) {
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTMCAST, skb->len);
        } else if (rt->rt_type == RTN_BROADCAST)
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTBCAST, skb->len);

        /* Be paranoid, rather than too clever. */
        if (unlikely(skb_headroom(skb) < hh_len && dev->header_ops)) {
                struct sk_buff *skb2;

                skb2 = skb_realloc_headroom(skb, LL_RESERVED_SPACE(dev));
                if (skb2 == NULL) {
                        kfree_skb(skb);
                        return -ENOMEM;
                }
                if (skb->sk)
                        skb_set_owner_w(skb2, skb->sk);
                consume_skb(skb);
                skb = skb2;
        }
```
如果与此数据包关联的路由是多播类型，则使用 IP_UPD_PO_STATS 宏来增加 OutMcastPkts 和 OutMcastOctets 计数。如果广播路由，则会增加 OutBcastPkts 和 OutBcastOctets 计数。<br>
接下来，确保 skb 结构有足够的空间容纳需要添加的任何链路层头。如果空间不够，则调用 skb_realloc_headroom 分配额外的空间，并且新的 skb 的费用（charge）记在相关的 socket 上。
```c
rcu_read_lock_bh();
        nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
        neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
        if (unlikely(!neigh))
                neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
```
继续，查询路由层找到下一跳，再根据下一跳信息查找邻居缓存。如果未找到，则调用__neigh_create 创建一个邻居。例如，第一次将数据发送到另一台主机的时候，就是这种情况。创建邻居缓存的时候带了 arp_tbl参数。其他系统（如 IPv6 或 DECnet）维护自己的 ARP 表，并将不同的变量传给__neigh_create。邻居缓存如果创建， 会导致缓存表增大。邻居缓存会导出一组统计信息，以便可以衡量这种增长。
```c
if (!IS_ERR(neigh)) {
                int res = dst_neigh_output(dst, neigh, skb);

                rcu_read_unlock_bh();
                return res;
        }
        rcu_read_unlock_bh();

        net_dbg_ratelimited("%s: No header cache and no neighbour!\n",
                            __func__);
        kfree_skb(skb);
        return -EINVAL;
}
```
最后，如果创建邻居缓存成功，则调用 `dst_neigh_output` 继续传递 skb；否则，释放 skb 并返回 EINVAL，这会向上传递，导致 OutDiscards 在 ip_send_skb 中递增。

## 7. dst_neigh_output
`dst_neigh_output` 函数做了两件重要的事情。首先，如果用户调用 `sendmsg `并通过辅助消息指定 MSG_CONFIRM 参数，则会设置一个标志位以指示目标高速缓存条目仍然有效且不应进行垃圾回收。这个检查就是在这个函数里面做的，并且邻居上的 confirm 字段设置为当前的 jiffies 计数。
```c
static inline int dst_neigh_output(struct dst_entry *dst, struct neighbour *n,
                                   struct sk_buff *skb)
{
        const struct hh_cache *hh;

        if (dst->pending_confirm) {
                unsigned long now = jiffies;

                dst->pending_confirm = 0;
                /* avoid dirtying neighbour */
                if (n->confirmed != now)
                        n->confirmed = now;
        }
```
其次，检查邻居的状态并调用适当的 output 函数。
```c
hh = &n->hh;
        if ((n->nud_state & NUD_CONNECTED) && hh->hh_len)
                return neigh_hh_output(hh, skb);
        else
                return n->output(n, skb);
}
```
邻居被认为是 NUD_CONNECTED，如果它满足以下一个或多个条件：<br>
1、NUD_PERMANENT：静态路由<br>
2、NUD_NOARP：不需要 ARP 请求（例如，目标是多播或广播地址，或环回设备）<br>
3、NUD_REACHABLE：邻居是“可达的。”只要成功处理了ARP 请求，目标就会被标记为可达<br>

进一步，如果“硬件头”（hh）被缓存（之前已经发送过数据，并生成了缓存），将调用 neigh_hh_output。<br>
否则，调用 output 函数。<br>
以上两种情况，最后都会到 `dev_queue_xmit`，它将 skb 发送给 Linux 网络设备子系统，在它 进入设备驱动程序层之前将对其进行更多处理。让我们沿着 `neigh_hh_output` 和 `n->output` 代码继续向下，直到达到 `dev_queue_xmit`。

### 7.1 neigh_hh_output
如果目标是 NUD_CONNECTED 并且硬件头已被缓存，则将调用 `neigh_hh_output`，在将 skb 移交 给 `dev_queue_xmit` 之前执行一小部分处理 ：
```c
static inline int neigh_hh_output(const struct hh_cache *hh, struct sk_buff *skb)
{
        unsigned int seq;
        int hh_len;

        do {
                seq = read_seqbegin(&hh->hh_lock);
                hh_len = hh->hh_len;
                if (likely(hh_len <= HH_DATA_MOD)) {
                        /* this is inlined by gcc */
                        memcpy(skb->data - HH_DATA_MOD, hh->hh_data, HH_DATA_MOD);
                 } else {
                         int hh_alen = HH_DATA_ALIGN(hh_len);

                         memcpy(skb->data - hh_alen, hh->hh_data, hh_alen);
                 }
         } while (read_seqretry(&hh->hh_lock, seq));

         skb_push(skb, hh_len);
         return dev_queue_xmit(skb);
}
```
这个函数理解有点难，部分原因是seqlock这个东西，它用于在缓存的硬件头上做读/写锁。可以将上面的 do {} while ()循环想象成一个简单的重试机制，它将尝试在循环中执行，直到成功。<br>
循环里处理硬件头的长度对齐。这是必需的，因为某些硬件头（如IEEE 802.11 头）大于 HH_DATA_MOD（16 字节）。<br>
将头数据复制到 skb 后，`skb_push` 将更新 skb 内指向数据缓冲区的指针。最后调用 `dev_queue_xmit` 将 skb 传递给 Linux 网络设备子系统。<br>

### 7.2 n->output
如果目标不是 NUD_CONNECTED 或硬件头尚未缓存，则代码沿 n->output 路径向下。neigbour 结构上的 output 指针指向哪个函数？这得看情况。要了解这是如何设置的，我们需要更多地了解邻居缓存的工作原理。

`struct neighbour` 包含几个重要字段：我们在上面看到的 `nud_state` 字段，output 函数和 ops 结构。回想一下，我们之前看到如果在缓存中找不到现有条目，会从 `ip_finish_output2` 调用`__neigh_create 创建一个。当调用__neigh_creaet` 时，将分配邻居，其 output 函数最初设置为` neigh_blackhole`。随着`__neigh_create` 代码的进行，它将根据邻居的状态修改 output 值以指向适当的发送方法。

例如，当代码确定是“已连接的”邻居时，`neigh_connect` 会将 `output` 设置为 `neigh->ops->connected_output`。或者，当代码怀疑邻居可能已关闭时，`neigh_suspect` 会将 `output` 设置为 `neigh->ops->output`（例如，如果已超过 `/proc/sys/net/ipv4/neigh/default/delay_first_probe_time` 自发送探测以来的 `delay_first_probe_time` 秒）。

换句话说：`neigh->output` 会被设置为 `neigh->ops_connected_output` 或 `neigh->ops->output`，具体取决于邻居的状态。`neigh->ops` 来自哪里？

分配邻居后，调用 `arp_constructor（net/ipv4/arp.c ）`来设置 `struct neighbor` 的某些字段。特别是，此函数会检查与此邻居关联的设备是否导出来一个 `struct header_ops` 实例， 该结构体有一个 `cache` 方法。

`neigh->ops` 设置为 `net/ipv4/arp` 中定义的以下实例：

```c
static const struct neigh_ops arp_hh_ops = {
        .family =               AF_INET,
        .solicit =              arp_solicit,
        .error_report =         arp_error_report,
        .output =               neigh_resolve_output,
        .connected_output =     neigh_resolve_output,
};
```
所以，不管 neighbor 是不是“已连接的”，或者邻居缓存代码是否怀疑连接“已关闭”， `neigh_resolve_output` 最终都会被赋给 `neigh->output`。当执行到 `n->output` 时就会调用它。

### 7.3 neigh_resolve_output
此函数的目的是解析未连接的邻居，或已连接但没有缓存硬件头的邻居。

```c
/* Slow and careful. */

int neigh_resolve_output(struct neighbour *neigh, struct sk_buff *skb)
{
        struct dst_entry *dst = skb_dst(skb);
        int rc = 0;

        if (!dst)
                goto discard;

        if (!neigh_event_send(neigh, skb)) {
                int err;
                struct net_device *dev = neigh->dev;
                unsigned int seq;
       }
} 
```
代码首先进行一些基本检查，然后调用 `neigh_event_send。 neigh_event_send` 函数是 `__neigh_event_send` 的简单封装，后者干大部分脏话累活。可以在 `net/core/neighbour.c` 中读`__neigh_event_send`的源代码，从大的层面看，三种情况：
* NUD_NONE 状态（默认状态）的邻居：假设 `/proc/sys/net/ipv4/neigh/default/app_solicit` 和 `/proc/sys/net/ipv4/neigh/default/mcast_solicit` 配置允许发送探测（如果不是， 则将状态标记为 NUD_FAILED），将导致立即发送 ARP 请求。邻居状态将更新为 `NUD_INCOMPLETE`
* NUD_STALE 状态的邻居：将更新为 `NUD_DELAYED` 并且将设置计时器以稍后探测它们（ 稍后是现在的时间+/proc/sys/net/ipv4/neigh/default/delay_first_probe_time 秒 ）
* 检查 NUD_INCOMPLETE 状态的邻居（包括上面第一种情形），以确保未解析邻居的排 队 packet 的数量小于等于/proc/sys/net/ipv4/neigh/default/unres_qlen。如果超过 ，则数据包会出列并丢弃，直到小于等于 proc 中的值。统计信息中有个计数器会因此 更新

如果需要 ARP 探测，ARP 将立即被发送。`__neigh_event_send` 将返回 0，表示邻居被视为“已 连接”或“已延迟”，否则返回 1。返回值 0 允许 `eigh_resolve_output` 继续：

```c
if (dev->header_ops->cache && !neigh->hh.hh_len)
                        neigh_hh_init(neigh, dst);
```
如果邻居关联的设备的协议实现（在我们的例子中是以太网）支持缓存硬件头，并且当前没有缓存，`neigh_hh_init` 将缓存它。
```c
do {
                        __skb_pull(skb, skb_network_offset(skb));
                        seq = read_seqbegin(&neigh->ha_lock);
                        err = dev_hard_header(skb, dev, ntohs(skb->protocol),
                                              neigh->ha, NULL, skb->len);
                } while (read_seqretry(&neigh->ha_lock, seq));
```
接下来，seqlock 锁控制对邻居的硬件地址字段（neigh->ha）的访问。`ev_hard_header` 为 skb 创建以太网头时将读取该字段。之后是错误检查：
```c
if (err >= 0)
                        rc = dev_queue_xmit(skb);
                else
                        goto out_kfree_skb;
        }
```
如果以太网头写入成功，将调用 `dev_queue_xmit` 将 skb 传递给 Linux 网络设备子系统进行发 送。如果出现错误，goto 将删除 skb，设置并返回错误码：
```
out:
        return rc;
discard:
        neigh_dbg(1, "%s: dst=%p neigh=%p\n", __func__, dst, neigh);
out_kfree_skb:
        rc = -EINVAL;
        kfree_skb(skb);
        goto out;
}
EXPORT_SYMBOL(neigh_resolve_output);
```
接下来看一些用于监控和转换 IP 协议层的文件。

## 8. 监控: IP 层
### 8.1 /proc/net/snmp

这个文件包扩多种协议的统计，IP 层的在最前面，每一列代表什么有说明。<br>
前面我们已经看到 IP 协议层有一些地方会更新计数器。这些计数器的类型是 C 枚举类型，定义在`include/uapi/linux/snmp.h`:
```
enum
{
  IPSTATS_MIB_NUM = 0,
/* frequently written fields in fast path, kept in same cache line */
  IPSTATS_MIB_INPKTS,     /* InReceives */
  IPSTATS_MIB_INOCTETS,     /* InOctets */
  IPSTATS_MIB_INDELIVERS,     /* InDelivers */
  IPSTATS_MIB_OUTFORWDATAGRAMS,   /* OutForwDatagrams */
  IPSTATS_MIB_OUTPKTS,      /* OutRequests */
  IPSTATS_MIB_OUTOCTETS,      /* OutOctets */

  /* ... */
```
一些有趣的统计：
```
OutRequests: Incremented each time an IP packet is attempted to be sent. It appears that this is incremented for every send, successful or not.
OutDiscards: Incremented each time an IP packet is discarded. This can happen if appending data to the skb (for corked sockets) fails, or if the layers below IP return an error.
OutNoRoute: Incremented in several places, for example in the UDP protocol layer (udp_sendmsg) if no route can be generated for a given destination. Also incremented when an application calls “connect” on a UDP socket but no route can be found.
FragOKs: Incremented once per packet that is fragmented. For example, a packet split into 3 fragments will cause this counter to be incremented once.
FragCreates: Incremented once per fragment that is created. For example, a packet split into 3 fragments will cause this counter to be incremented thrice.
FragFails: Incremented if fragmentation was attempted, but is not permitted (because the “Don’t Fragment” bit is set). Also incremented if outputting the fragment fails.
```
### 8.2 /proc/net/netstat

格式与前面的类似，除了每列的名称都有 IpExt 前缀之外。<br>
一些有趣的统计：
```
OutMcastPkts: Incremented each time a packet destined for a multicast address is sent.
OutBcastPkts: Incremented each time a packet destined for a broadcast address is sent.
OutOctects: The number of packet bytes output.
OutMcastOctets: The number of multicast packet bytes output.
OutBcastOctets: The number of broadcast packet bytes output.
```
## 9. 总结

Linux内核网络数据包发送时，主要用到` ip_send_skb`、 `ip_local_out`、`ip_output`、`ip_finish_output`、`ip_finish_output2`、 `st_neigh_output`等函数，本文通过分析这些函数来分享Linux内核数据包发送在 IP 层的处理，并对 IP 层进行了数据监控。


