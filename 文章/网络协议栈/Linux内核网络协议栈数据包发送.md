由于在connect函数中涉及数据包的发送与接收问题，事实上，发送与接收函数不限于connect函数，所以这里单独剖析。

承前文继续剖析 connect 函数，数据包的发送和接收在 ip_queue_xmit 函数和 release_sock 函数中实现。本文着重分析 ip_queue_xmit 函数，下篇将补充分析 connect 函数剩下的部分。

值得注意的是：这些函数是数据包发送函数，在数据传输阶段，基本上都会调用该函数，因为connect涉及该函数，就放在这里介绍了，不意味着这个函数只属于connect下层函数。

## 1、网络层——ip_queue_xmit 函数

```text
/*
 * Queues a packet to be sent, and starts the transmitter
 * if necessary.  if free = 1 then we free the block after
 * transmit, otherwise we don't. If free==2 we not only
 * free the block but also don't assign a new ip seq number.
 * This routine also needs to put in the total length,
 * and compute the checksum
 */
 //数据包发送函数
 //sk:被发送数据包对应的套接字；dev:发送数据包的网络设备
 //skb:被发送的数据包         ；flags:是否对数据包进行缓存以便于此后的超时重发
void ip_queue_xmit(struct sock *sk, struct device *dev,
	      struct sk_buff *skb, int free)
{
	struct iphdr *iph;
	unsigned char *ptr;
 
	/* Sanity check */
	//发送设备检查
	if (dev == NULL)
	{
		printk("IP: ip_queue_xmit dev = NULL\n");
		return;
	}
 
	IS_SKB(skb);//数据包合法性检查
 
	/*
	 *	Do some book-keeping in the packet for later
	 */
 
 
	skb->dev = dev;
	skb->when = jiffies;//重置数据包的发送时间，只有一个定时器，每次发数据包时，都要重新设置
 
	/*
	 *	Find the IP header and set the length. This is bad
	 *	but once we get the skb data handling code in the
	 *	hardware will push its header sensibly and we will
	 *	set skb->ip_hdr to avoid this mess and the fixed
	 *	header length problem
	 */
    //skb->data指向的地址空间的布局: MAC首部 | IP首部 | TCP首部 | 有效负载
	ptr = skb->data;//获取数据部分首地址
	ptr += dev->hard_header_len;//后移硬件(MAC)首部长度个字节，定位到ip首部
	iph = (struct iphdr *)ptr;//获取ip首部
	skb->ip_hdr = iph;//skb对应字段建立关联
	//ip数据报的总长度(ip首部+数据部分) = skb的总长度 - 硬件首部长度
	iph->tot_len = ntohs(skb->len-dev->hard_header_len);
 
#ifdef CONFIG_IP_FIREWALL
	//数据包过滤，用于防火墙安全性检查
	if(ip_fw_chk(iph, dev, ip_fw_blk_chain, ip_fw_blk_policy, 0) != 1)
		/* just don't send this packet */
		return;
#endif	
 
	/*
	 *	No reassigning numbers to fragments...
	 */
    //如果不是分片数据包，就需要递增id字段
//free==2，表示这是个分片数据包，所有分片数据包必须具有相同的id字段，方便以后分片数据包重组
	if(free!=2)
		iph->id      = htons(ip_id_count++);//ip数据报标识符
		//ip_id_count是全局变量，用于下一个数据包中ip首部id字段的赋值
	else
		free=1;
 
	/* All buffers without an owner socket get freed */
	if (sk == NULL)//没有对应sock结构，则无法对数据包缓存
		free = 1;
 
	skb->free = free;//用于标识数据包发送之后是缓存还是立即释放,=1表示无缓存
 
	/*
	 *	Do we need to fragment. Again this is inefficient.
	 *	We need to somehow lock the original buffer and use
	 *	bits of it.
	 */
	//数据包拆分
    //如果ip层数据包的数据部分(各层首部+有效负载)长度大于网络设备的最大传输单元，就需要拆分发送
    //实际是skb->len - dev->hard_header_len > dev->mtu
    //因为MTU最大报文长度表示的仅仅是IP首部及其数据负载的长度，所以要考虑MAC首部长度
	if(skb->len > dev->mtu + dev->hard_header_len)
	{
	//拆分成分片数据包传输
		ip_fragment(sk,skb,dev,0);
		IS_SKB(skb);//检查数据包skb相关字段
		kfree_skb(skb,FREE_WRITE);
		return;
	}
 
	/*
	 *	Add an IP checksum
	 */
    //ip首部校验和计算
	ip_send_check(iph);
 
	/*
	 *	Print the frame when debugging
	 */
 
	/*
	 *	More debugging. You cannot queue a packet already on a list
	 *	Spot this and moan loudly.
	 */
	if (skb->next != NULL)
	{
		printk("ip_queue_xmit: next != NULL\n");
		skb_unlink(skb);
	}
 
	/*
	 *	If a sender wishes the packet to remain unfreed
	 *	we add it to his send queue. This arguably belongs
	 *	in the TCP level since nobody else uses it. BUT
	 *	remember IPng might change all the rules.
	 */
    //free=0，表示对数据包进行缓存，一旦发生丢弃的情况，进行数据包重传(可靠性数据传输协议)
	if (!free)
	{
		unsigned long flags;
		/* The socket now has more outstanding blocks */
 
		sk->packets_out++;//本地发送出去但未得到应答的数据包数目
 
		/* Protect the list for a moment */
		save_flags(flags);
		cli();
 
		//数据包重发队列
		if (skb->link3 != NULL)
		{
			printk("ip.c: link3 != NULL\n");
			skb->link3 = NULL;
		}
		if (sk->send_head == NULL)
		{
		//数据包重传缓存队列则是由下列两个字段维护
			sk->send_tail = skb;
			sk->send_head = skb;
		}
		else
		{
			sk->send_tail->link3 = skb;
			sk->send_tail = skb;
		}
		/* skb->link3 is NULL */
 
		/* Interrupt restore */
		restore_flags(flags);
	}
	else
		/* Remember who owns the buffer */
		skb->sk = sk;
 
	/*
	 *	If the indicated interface is up and running, send the packet.
	 */
	 
	ip_statistics.IpOutRequests++;
#ifdef CONFIG_IP_ACCT
//下面函数内部调用ip_fw_chk，也是数据包过滤
	ip_acct_cnt(iph,dev, ip_acct_chain);
#endif	
	
#ifdef CONFIG_IP_MULTICAST	
    //对于多播和广播数据包，其必须复制一份回送给本机
	/*
	 *	Multicasts are looped back for other local users
	 */
	 /*对多播和广播数据包进行处理*/
	 //检查目的地址是否为一个多播地址
	if (MULTICAST(iph->daddr) && !(dev->flags&IFF_LOOPBACK))
	{
	//检查发送设备是否为一个回路设备
		if(sk==NULL || sk->ip_mc_loop)
		{
			if(iph->daddr==IGMP_ALL_HOSTS)//如果是224.0.0.1(默认多播地址)
				ip_loopback(dev,skb);//数据包回送给发送端
			else
			{  //检查多播地址列表，对数据包进行匹配
				struct ip_mc_list *imc=dev->ip_mc_list;
				while(imc!=NULL)
				{
					if(imc->multiaddr==iph->daddr)//如果存在匹配项，则回送数据包
					{
						ip_loopback(dev,skb);
						break;
					}
					imc=imc->next;
				}
			}
		}
		/* Multicasts with ttl 0 must not go beyond the host */
		//检查ip首部ttl字段，如果为0，则不可进行数据包发送(转发)
		if(skb->ip_hdr->ttl==0)
		{
			kfree_skb(skb, FREE_READ);
			return;
		}
	}
#endif
	//对广播数据包的判断
	if((dev->flags&IFF_BROADCAST) && iph->daddr==dev->pa_brdaddr && !(dev->flags&IFF_LOOPBACK))
		ip_loopback(dev,skb);
 
	//对发送设备当前状态的检查，如果处于非工作状态，则无法发送数据包，此时进入else执行
	if (dev->flags & IFF_UP)
	{
		/*
		 *	If we have an owner use its priority setting,
		 *	otherwise use NORMAL
		 */
        //调用下层接口函数dev_queue_xmit，将数据包交由链路层处理
		if (sk != NULL)
		{
			dev_queue_xmit(skb, dev, sk->priority);
		}
		else
		{
			dev_queue_xmit(skb, dev, SOPRI_NORMAL);
		}
	}
	else
	{
		ip_statistics.IpOutDiscards++;
		if (free)
			kfree_skb(skb, FREE_WRITE);//丢弃数据包，对tcp可靠传输而言，将造成数据包超时重传
	}
}
```

上面函数功能可以总结为：

\1. 相关合法性检查；

\2. 防火墙过滤；

\3. 对数据包是否需要分片发送进行检查；

\4. 进行可能的数据包缓存处理；

\5. 对多播和广播数据报是否需要回送本机进行检查；

\6. 调用下层接口函数 dev_queue_xmit 将数据包送往链路层进行处理。

上面函数内部涉及到一个函数，把数据包分片，当数据包大小大于最大传输单元时，需要将数据包分片传送，这里则是通过函数 ip_fragment 实现的。【Linux 内核网络协议栈源码剖析】数据包发送上面函数内部涉及到一个函数，把数据包分片，当数据包大小大于最大传输单元时，需要将数据包分片传送，这里则是通过函数 ip_fragment 实现的。

ip_fragment函数：将大数据包（大于最大传输单元(最大传输单元指的是ip报，不包含mac头部)）分片发送。这个函数条理很清楚

```text
/*
 *	This IP datagram is too large to be sent in one piece.  Break it up into
 *	smaller pieces (each of size equal to the MAC header plus IP header plus
 *	a block of the data of the original IP data part) that will yet fit in a
 *	single device frame, and queue such a frame for sending by calling the
 *	ip_queue_xmit().  Note that this is recursion, and bad things will happen
 *	if this function causes a loop...
 *
 *	Yes this is inefficient, feel free to submit a quicker one.
 *
 *	**Protocol Violation**
 *	We copy all the options to each fragment. !FIXME!
 */
void ip_fragment(struct sock *sk, struct sk_buff *skb, struct device *dev, int is_frag)
{
	struct iphdr *iph;
	unsigned char *raw;
	unsigned char *ptr;
	struct sk_buff *skb2;
	int left, mtu, hlen, len;
	int offset;
	unsigned long flags;
 
	/*
	 *	Point into the IP datagram header.
	 */
 
	raw = skb->data;//得到数据部分，如果你还没清楚skb与data表示什么的话，请自行面壁
	iph = (struct iphdr *) (raw + dev->hard_header_len);//偏移mac首部就到了ip首部位置
 
	skb->ip_hdr = iph;//指定ip首部
 
	/*
	 *	Setup starting values.
	 */
    //ip数据报由ip首部和数据负载部分组成，其中数据负载部分又有tcp首部和tcp有效负载组成
	hlen = (iph->ihl * sizeof(unsigned long));//ip首部长度
	left = ntohs(iph->tot_len) - hlen;//ip数据负载长度	/* Space per frame */
	hlen += dev->hard_header_len;//加上mac首部长度，得到这两者的长度和		/* Total header size */
	mtu = (dev->mtu - hlen);//mtu初始化为ip数据负载长度		/* Size of data space */
	ptr = (raw + hlen);	//指向ip负载数据开始位置		/* Where to start from */
 
	/*
	 *	Check for any "DF" flag. [DF means do not fragment]
	 */
    //检查发送端是否允许进行分片
	if (ntohs(iph->frag_off) & IP_DF)//如果不允许
	{
		/*
		 *	Reply giving the MTU of the failed hop.
		 */
		 //返回一个ICMP错误
		ip_statistics.IpFragFails++;
		icmp_send(skb,ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED, dev->mtu, dev);
		return;
	}
 
	/*
	 *	The protocol doesn't seem to say what to do in the case that the
	 *	frame + options doesn't fit the mtu. As it used to fall down dead
	 *	in this case we were fortunate it didn't happen
	 */
    //如果ip数据负载长度<8，则无法为其创建分片
    //规定，分片中数据长度必须是8的倍数
	if(mtu<8)
	{
		/* It's wrong but it's better than nothing */
		icmp_send(skb,ICMP_DEST_UNREACH,ICMP_FRAG_NEEDED,dev->mtu, dev);
		ip_statistics.IpFragFails++;
		return;
	}
 
	/*
	 *	Fragment the datagram.
	 */
 
	/*
	 *	The initial offset is 0 for a complete frame. When
	 *	fragmenting fragments it's wherever this one starts.
	 */
 
	if (is_frag & 2)//表示被分片数据包本身是一个分片
		offset = (ntohs(iph->frag_off) & 0x1fff) << 3;
	else
		offset = 0;
 
 
	/*
	 *	Keep copying data until we run out.
	 */
    //直到所有分片数据处理完
	while(left > 0)
	{
		len = left;
		/* IF: it doesn't fit, use 'mtu' - the data space left */
		if (len > mtu)//如果这个数据还是大于mtu，表示不是最后一个分片，还要继续分片处理
			len = mtu;
		/* IF: we are not sending upto and including the packet end
		   then align the next start on an eight byte boundary */
		if (len < left)
		{
//得到小于mtu的最大的一个为8的倍数的数值，这个运算很有意思，值得借鉴一下
//可以看出分片的原则是每个分片在条件下尽量携带最多数据，这个条件就是不能大于mtu值，且必须是8的倍数
			len/=8;
			len*=8;
		}
		/*
		 *	Allocate buffer.
		 */
//分配一个skb_buff，注意这里分配的大小，hlen是ip首部大小，得加上ip首部
//这里可以看出，每个分片数据包还得加上ip首部，典型的1+1>2，相比不分片下降低了效率，
//但是这是不可避免的，必须增加火车头，不然不知道往哪开
		if ((skb2 = alloc_skb(len + hlen,GFP_ATOMIC)) == NULL)
		{
			printk("IP: frag: no memory for new fragment!\n");
			ip_statistics.IpFragFails++;
			return;
		}
 
		/*
		 *	Set up data on packet
		 */
 
		skb2->arp = skb->arp;//表示mac首部是否创建成功，
		//参见ip_build_header函数，其内部调用了ip_send函数(eth_header)
		if(skb->free==0)
			printk("IP fragmenter: BUG free!=1 in fragmenter\n");
		skb2->free = 1;//数据包无须缓存
		skb2->len = len + hlen;//数据部分长度(分片大小+ip首部)
		skb2->h.raw=(char *) skb2->data;//让skb2对应层的raw指向分片数据包的数据部分
		/*
		 *	Charge the memory for the fragment to any owner
		 *	it might possess
		 */
 
		save_flags(flags);
		if (sk)
		{
			cli();
			sk->wmem_alloc += skb2->mem_len;//设置sk当前写缓冲大小为分片数据包的大小
			skb2->sk=sk;//关联
		}
		restore_flags(flags);
		skb2->raddr = skb->raddr;//数据包的下一站地址，所有分片数据包自然是原数据包是一个地址	/* For rebuild_header - must be here */
 
		/*
		 *	Copy the packet header into the new buffer.
		 */
//把skb数据部分拷贝到raw指向的位置，这里拷贝的是首部(mac首部+ip首部)
//实际上raw和skb2->data是指向同一个地址
		memcpy(skb2->h.raw, raw, hlen);
		//raw随着层次变化，链路层=eth，ip层=iph
 
		/*
		 *	Copy a block of the IP datagram.
		 */
		//这里则是拷贝ip数据负载部分
		memcpy(skb2->h.raw + hlen, ptr, len);
		left -= len;//剩下未传送的数据大小
 
		skb2->h.raw+=dev->hard_header_len;//raw位置定位到了ip首部
        //一定要清楚skb_buff->data到了某一层的数据布局
		/*
		 *	Fill in the new header fields.
		 */
		 //获取ip首部
		iph = (struct iphdr *)(skb2->h.raw/*+dev->hard_header_len*/);
		iph->frag_off = htons((offset >> 3));//片位移，offset在前面进行了设置
		/*
		 *	Added AC : If we are fragmenting a fragment thats not the
		 *		   last fragment then keep MF on each bit
		 */
		if (left > 0 || (is_frag & 1))//left>0，表示这不是最后一个分片，还有剩下数据包未发送
			iph->frag_off |= htons(IP_MF);
 
	//ip数据负载已经发送了len各大小的分片数据包，那么就要更新下一个分片数据包的位置，以便发送
		ptr += len;//ip数据负载位置更新
		offset += len;//偏移量更新，
 
		/*
		 *	Put this fragment into the sending queue.
		 */
 
		ip_statistics.IpFragCreates++;
 
		ip_queue_xmit(sk, dev, skb2, 2);//发送数据包
		//可以看出，发送一个数据包的过程就是，检查其大小是否小于mtu，否则需要进行分片，
		//然后对分片进行发送，分片数据包自然是小于mtu，直到原来的大于mtu的数据包全部分片发送完
	}
	ip_statistics.IpFragOKs++;
}
```

ip_fragment 函数条理很清晰，就是将大的拆分为小的，其拆分过程为，新建指定大小（小于MTU的是8的倍数的最大值）的分片数据包，然后将原大数据包中的数据负载截取前分片大小，再加上ip首部，每个分片数据包都要加上ip首部，这样降低效率的措施不得不采用。然后就是发送这个分片数据包，直到大数据包分片发送完成。

ip_queue_xmit 最后通过调用 dev_queue_xmit 函数将数据包发往链路层进行处理。

## 2、链路层——dev_queue_xmit 函数

```text
/*
 *	Send (or queue for sending) a packet. 
 *
 *	IMPORTANT: When this is called to resend frames. The caller MUST
 *	already have locked the sk_buff. Apart from that we do the
 *	rest of the magic.
 */
 //该函数本身负责将数据包传递给驱动程序，由驱动程序最终将数据发送到物理介质上。
 /*
 skb:被发送的数据包；dev:数据包发送网络接口设备；pri:网络接口设备忙时，缓存该数据包时使用的优先级
 */
void dev_queue_xmit(struct sk_buff *skb, struct device *dev, int pri)
{
	unsigned long flags;
	int nitcount;
	struct packet_type *ptype;//用于网络层协议
	int where = 0;		/* used to say if the packet should go	*/
				/* at the front or the back of the	*/
				/* queue - front is a retransmit try	*/
 
	if (dev == NULL) 
	{
		printk("dev.c: dev_queue_xmit: dev = NULL\n");
		return;
	}
 
	//加锁
	if(pri>=0 && !skb_device_locked(skb))
		skb_device_lock(skb);	/* Shove a lock on the frame */
#ifdef CONFIG_SLAVE_BALANCING
	save_flags(flags);//保存状态
	cli();
	//检查是否使用了主从设备的连接方式
	//如果采用了这种方式，则发送数据包时，可在两个设备之间平均负载
	if(dev->slave!=NULL && dev->slave->pkt_queue < dev->pkt_queue &&
				(dev->slave->flags & IFF_UP))
		dev=dev->slave;
	restore_flags(flags);
#endif		
#ifdef CONFIG_SKB_CHECK 
	IS_SKB(skb);//检查数据包的合法性
#endif    
	skb->dev = dev;//指向数据包发送设备对应结构
 
	/*
	 *	This just eliminates some race conditions, but not all... 
	 */
    //检查以免造成竞争条件，事实上skb->next == NULL的
	if (skb->next != NULL) 
	{
		/*
		 *	Make sure we haven't missed an interrupt. 
		 */
		printk("dev_queue_xmit: worked around a missed interrupt\n");
		start_bh_atomic();//原子操作，宏定义
		dev->hard_start_xmit(NULL, dev);
		end_bh_atomic();
		return;
  	}
 
	/*
	 *	Negative priority is used to flag a frame that is being pulled from the
	 *	queue front as a retransmit attempt. It therefore goes back on the queue
	 *	start on a failure.
	 */
//优先级为负数，表示当前处理的数据包是从硬件队列中取下的，而非上层传递的新数据包
  	if (pri < 0) 
  	{
		pri = -pri-1;
		where = 1;
  	}
 
	if (pri >= DEV_NUMBUFFS) 
	{
		printk("bad priority in dev_queue_xmit.\n");
		pri = 1;
	}
 
	/*
	 *	If the address has not been resolved. Call the device header rebuilder.
	 *	This can cover all protocols and technically not just ARP either.
	 */
//arp标识是否完成链路层的硬件地址解析，如果没完成，则需要调用rebuild_header(eth_rebuild_header函数)
//完成链路层首部的创建工作
	if (!skb->arp && dev->rebuild_header(skb->data, dev, skb->raddr, skb)) {
		return;//这将启动arp地址解析过程，则数据包的发送则由arp协议模块负责，所以这里直接返回
	}
 
	save_flags(flags);
	cli();	
	if (!where) {//where=1，表示这是从上层接受的新数据包
#ifdef CONFIG_SLAVE_BALANCING	
		skb->in_dev_queue=1;//标识该数据包缓存在设备队列中
#endif		
		skb_queue_tail(dev->buffs + pri,skb);//插入到设备缓存队列的尾部
		skb_device_unlock(skb);		/* Buffer is on the device queue and can be freed safely */
		skb = skb_dequeue(dev->buffs + pri);//从设备缓存队列的首部读取数据包，这样取得的数据包可能不是我们之前插入的数据包
		skb_device_lock(skb);		/* New buffer needs locking down */
#ifdef CONFIG_SLAVE_BALANCING		
		skb->in_dev_queue=0;//该数据包当前不在缓存队列中
#endif		
	}
	restore_flags(flags);//恢复状态
 
	/* copy outgoing packets to any sniffer packet handlers */
	//内核对混杂模式的支持。不明白...
	if(!where)
	{
		for (nitcount= dev_nit, ptype = ptype_base; nitcount > 0 && ptype != NULL; ptype = ptype->next) 
		{
			/* Never send packets back to the socket
			 * they originated from - MvS (miquels@drinkel.ow.org)
			 */
			if (ptype->type == htons(ETH_P_ALL) &&
			   (ptype->dev == dev || !ptype->dev) &&
			   ((struct sock *)ptype->data != skb->sk))
			{
				struct sk_buff *skb2;
				if ((skb2 = skb_clone(skb, GFP_ATOMIC)) == NULL)//复制一份数据包
					break;
				/*
				 *	The protocol knows this has (for other paths) been taken off
				 *	and adds it back.
				 */
				skb2->len-=skb->dev->hard_header_len;//长度
				ptype->func(skb2, skb->dev, ptype);//协议处理函数
				nitcount--;
			}
		}
	}
	start_bh_atomic();
//下面调用hard_start_xmit函数，前面skb->next不为NULL时，也调用这个函数，不过参数数据包skb是NULL
//驱动层发送数据包，关联到了具体的网络设备处理函数，将进入真实的网卡驱动(物理层)
//高版本的内核协议栈，还有虚拟设备，这个版本就是直接进入真实设备
	if (dev->hard_start_xmit(skb, dev) == 0) {
		end_bh_atomic();
		/*
		 *	Packet is now solely the responsibility of the driver
		 */
		return;
	}
	end_bh_atomic();
 
	/*
	 *	Transmission failed, put skb back into a list. Once on the list it's safe and
	 *	no longer device locked (it can be freed safely from the device queue)
	 */
	cli();
#ifdef CONFIG_SLAVE_BALANCING
	skb->in_dev_queue=1;//如果使用主从设备，就缓存在队列中
	dev->pkt_queue++;//该设备缓存的待发送数据包个数加1
#endif		
	skb_device_unlock(skb);
	skb_queue_head(dev->buffs + pri,skb);//把数据包插入到数据包队列头中
	restore_flags(flags);
}
```

## 3、物理层

物理层则牵扯到具体的网络接口硬件设备了，实则是一个网络驱动程序。不同的网卡其驱动程序有所不同，这跟硬件的时序，延迟等有关。

![img](https://pic4.zhimg.com/80/v2-a8c44790925bc12839c9817d1cfb83ff_720w.webp)

关于驱动，这里我们就不介绍了。

这部分介绍的就是数据包的发送过程，从网络层到最底层的网卡驱动。下篇将介绍数据包的接收过程。

connect 函数可真是渗透到网络栈的各个层啊，connect 函数是客户端向服务器端发出连接请求数据包，该数据包需要最终到达服务器端处理，自然要从客户端从上至下经过应用层、传输层、网络层、链路层、硬件接口到达对端（对端接收则是反过来从下往上）。所以通信双方进行数据传输的函数都要经过这些网络协议栈。

从侧面可看出，内核网络协议栈的设计体现了高内聚，低耦合的原则，各层之间只提供接口函数，协议栈某一层的改动，不需要取改动其余层，保持接口的一致性就可以了，面对日益复杂的网络栈，这种设计风格无疑很有利于维护和升级。

另外中间协议各层还有一些牵扯到的操作函数，会在后面一一介绍。

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549023845