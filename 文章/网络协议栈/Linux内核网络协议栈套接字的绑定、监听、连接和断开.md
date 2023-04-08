## **1、套接字的绑定**

创建完套接字服务器端会在应用层使用bind函数进行套接字的绑定，这时会产生系统调用，sys_bind内核函数进行套接字。

系统调用函数的具体实现

```text
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		err = move_addr_to_kernel(umyaddr, addrlen, (struct sockaddr *)&address);
		if (err >= 0) {
			err = security_socket_bind(sock,
						   (struct sockaddr *)&address,
						   addrlen);
			if (!err)
				err = sock->ops->bind(sock,
						      (struct sockaddr *)
						      &address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

首先调用函数sockfd_lookup_light()函数通过文件描述符来查找对应的套接字sock。

```text
static struct socket *sockfd_lookup_light(int fd, int *err, int *fput_needed)
{
	struct file *file;
	struct socket *sock;

	*err = -EBADF;
	file = fget_light(fd, fput_needed);
	if (file) {
		sock = sock_from_file(file, err);
		if (sock)
			return sock;
		fput_light(file, *fput_needed);
	}
	return NULL;
}
```

上面函数中先调用fget_light函数通过文件描述符返回对应的文件结构，然后调用函数sock_from_file函数返回该文件对应的套接字结构体地址，它存储在file->private_data属性中。

再回到sys_bind函数，在返回了对应的套接字结构之后，调用move_addr_to_kernel将用户地址空间的socket拷贝到内核空间。

然后调用INET协议族的操作集中bind函数inet_bind函数将socket地址（内核空间）和socket绑定。

```text
int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
	struct sockaddr_in *addr = (struct sockaddr_in *)uaddr;
	struct sock *sk = sock->sk;
	struct inet_sock *inet = inet_sk(sk);
	unsigned short snum;
	int chk_addr_ret;
	int err;

	//RAW类型套接字若有自己的bind函数，则使用之
	if (sk->sk_prot->bind) {
		err = sk->sk_prot->bind(sk, uaddr, addr_len);
		goto out;
	}
	err = -EINVAL;
	.....................
        //地址合法性检查
	chk_addr_ret = inet_addr_type(sock_net(sk), addr->sin_addr.s_addr);

	/* Not specified by any standard per-se, however it breaks too
	 * many applications when removed.  It is unfortunate since
	 * allowing applications to make a non-local bind solves
	 * several problems with systems using dynamic addressing.
	 * (ie. your servers still start up even if your ISDN link
	 *  is temporarily down)
	 */
	err = -EADDRNOTAVAIL;
	if (!sysctl_ip_nonlocal_bind &&
	    !(inet->freebind || inet->transparent) &&
	    addr->sin_addr.s_addr != htonl(INADDR_ANY) &&
	    chk_addr_ret != RTN_LOCAL &&
	    chk_addr_ret != RTN_MULTICAST &&
	    chk_addr_ret != RTN_BROADCAST)
		goto out;

	snum = ntohs(addr->sin_port);
	err = -EACCES;
	if (snum && snum < PROT_SOCK && !capable(CAP_NET_BIND_SERVICE))
		goto out;

	/*      We keep a pair of addresses. rcv_saddr is the one
	 *      used by hash lookups, and saddr is used for transmit.
	 *
	 *      In the BSD API these are the same except where it
	 *      would be illegal to use them (multicast/broadcast) in
	 *      which case the sending device address is used.
	 */
	lock_sock(sk);

	/* Check these errors (active socket, double bind). */
	err = -EINVAL;
	if (sk->sk_state != TCP_CLOSE || inet->inet_num)//如果sk的状态是CLOSE或者本地端口已经被绑定
		goto out_release_sock;

	inet->inet_rcv_saddr = inet->inet_saddr = addr->sin_addr.s_addr;//设置源地址
	if (chk_addr_ret == RTN_MULTICAST || chk_addr_ret == RTN_BROADCAST)
		inet->inet_saddr = 0;  /* Use device */

	/* Make sure we are allowed to bind here. */
	if (sk->sk_prot->get_port(sk, snum)) {
		inet->inet_saddr = inet->inet_rcv_saddr = 0;
		err = -EADDRINUSE;
		goto out_release_sock;
	}

	if (inet->inet_rcv_saddr)
		sk->sk_userlocks |= SOCK_BINDADDR_LOCK;
	if (snum)
		sk->sk_userlocks |= SOCK_BINDPORT_LOCK;
	inet->inet_sport = htons(inet->inet_num);//设置源端口号，标明该端口已经被占用
	inet->inet_daddr = 0;
	inet->inet_dport = 0;
	sk_dst_reset(sk);
	err = 0;
out_release_sock:
	release_sock(sk);
out:
	return err;
}
```

这样套接字绑定结束。

## **2、套接字的监听**

```text
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	struct socket *sock;
	int err, fput_needed;
	int somaxconn;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		......................

		err = security_socket_listen(sock, backlog);
		if (!err)
			err = sock->ops->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

该函数先通过文件描述符查找到对应的套接字结构，然后调用inet_listen函数对将套接字sk的状态设置为TCP_LISTEN。

```text
int inet_listen(struct socket *sock, int backlog)
{
	struct sock *sk = sock->sk;
	unsigned char old_state;
	int err;
	lock_sock(sk);

	err = -EINVAL;
	if (sock->state != SS_UNCONNECTED || sock->type != SOCK_STREAM)
		goto out;

	old_state = sk->sk_state;
	if (!((1 << old_state) & (TCPF_CLOSE | TCPF_LISTEN)))
		goto out;

	if (old_state != TCP_LISTEN) {
		err = inet_csk_listen_start(sk, backlog);//该函数将sk的状态设置为TCP_LISTEN
		if (err)
			goto out;
	}
	sk->sk_max_ack_backlog = backlog;
	err = 0;
out:
	release_sock(sk);
	return err;
}
```

## **3、套接字的连接和接受连接**

### **3.1申请连接**

```text
SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
		int, addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (!sock)
		goto out;
	err = move_addr_to_kernel(uservaddr, addrlen, (struct sockaddr *)&address);
	if (err < 0)
		goto out_put;

	err =
	    security_socket_connect(sock, (struct sockaddr *)&address, addrlen);
	if (err)
		goto out_put;

	err = sock->ops->connect(sock, (struct sockaddr *)&address, addrlen,
				 sock->file->f_flags);
out_put:
	fput_light(sock->file, fput_needed);
out:
	return err;
}
```

还是先调用sockfd_lookup_light函数获得socket指针，然后将用户空间地址移到内核空间，然后调用函数inet_stream_connect函数。

```text
int inet_stream_connect(struct socket *sock, struct sockaddr *uaddr,
			int addr_len, int flags)
{
	struct sock *sk = sock->sk;
	int err;
	long timeo;

	if (addr_len < sizeof(uaddr->sa_family))
		return -EINVAL;

	lock_sock(sk);

	......................

	switch (sock->state) {
	default:
		err = -EINVAL;
		goto out;
	case SS_CONNECTED:
		err = -EISCONN;
		goto out;
	case SS_CONNECTING:
		err = -EALREADY;
		/* Fall out of switch with err, set for this state */
		break;
	case SS_UNCONNECTED:
		err = -EISCONN;
		if (sk->sk_state != TCP_CLOSE)
			goto out;

		err = sk->sk_prot->connect(sk, uaddr, addr_len);
		if (err < 0)
			goto out;

		sock->state = SS_CONNECTING;

		err = -EINPROGRESS;
		break;
	}

	timeo = sock_sndtimeo(sk, flags & O_NONBLOCK);

	if ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) {
		/* Error code is set above */
		if (!timeo || !inet_wait_for_connect(sk, timeo))
			goto out;

		err = sock_intr_errno(timeo);
		if (signal_pending(current))
			goto out;
	}

	/* Connection was closed by RST, timeout, ICMP error
	 * or another process disconnected us.
	 */
	if (sk->sk_state == TCP_CLOSE)
		goto sock_error;

	sock->state = SS_CONNECTED;
	err = 0;
out:
	release_sock(sk);
	return err;

sock_error:
	err = sock_error(sk) ? : -ECONNABORTED;
	sock->state = SS_UNCONNECTED;
	if (sk->sk_prot->disconnect(sk, flags))
		sock->state = SS_DISCONNECTING;
	goto out;
}
```

调用函数tcp_v4_connect函数后然后将sock的状态置SS_CONNECTING。

```text
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
	struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
	struct inet_sock *inet = inet_sk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	__be16 orig_sport, orig_dport;
	__be32 daddr, nexthop;
	struct flowi4 *fl4;
	struct rtable *rt;
	int err;
	struct ip_options_rcu *inet_opt;
        //合法性检查
	if (addr_len < sizeof(struct sockaddr_in))
		return -EINVAL;

	if (usin->sin_family != AF_INET)
		return -EAFNOSUPPORT;
        //记录吓一跳地址和目的地址
	nexthop = daddr = usin->sin_addr.s_addr;
	inet_opt = rcu_dereference_protected(inet->inet_opt,
					     sock_owned_by_user(sk));
	if (inet_opt && inet_opt->opt.srr) {
		if (!daddr)
			return -EINVAL;
		nexthop = inet_opt->opt.faddr;
	}
        //本地端口和目的端口
	orig_sport = inet->inet_sport;
	orig_dport = usin->sin_port;
	fl4 = &inet->cork.fl.u.ip4;
	rt = ip_route_connect(fl4, nexthop, inet->inet_saddr,
			      RT_CONN_FLAGS(sk), sk->sk_bound_dev_if,
			      IPPROTO_TCP,
			      orig_sport, orig_dport, sk, true);//维护路由表
	if (IS_ERR(rt)) {
		err = PTR_ERR(rt);
		if (err == -ENETUNREACH)
			IP_INC_STATS_BH(sock_net(sk), IPSTATS_MIB_OUTNOROUTES);
		return err;
	}
        //处理多播或广播
	if (rt->rt_flags & (RTCF_MULTICAST | RTCF_BROADCAST)) {
		ip_rt_put(rt);
		return -ENETUNREACH;
	}

	if (!inet_opt || !inet_opt->opt.srr)
		daddr = fl4->daddr;

	if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr;
	inet->inet_rcv_saddr = inet->inet_saddr;

	if (tp->rx_opt.ts_recent_stamp && inet->inet_daddr != daddr) {
		/* Reset inherited state */
		tp->rx_opt.ts_recent	   = 0;
		tp->rx_opt.ts_recent_stamp = 0;
		tp->write_seq		   = 0;
	}

	if (tcp_death_row.sysctl_tw_recycle &&
	    !tp->rx_opt.ts_recent_stamp && fl4->daddr == daddr) {
		struct inet_peer *peer = rt_get_peer(rt, fl4->daddr);
		/*
		 * VJ's idea. We save last timestamp seen from
		 * the destination in peer table, when entering state
		 * TIME-WAIT * and initialize rx_opt.ts_recent from it,
		 * when trying new connection.
		 */
		if (peer) {
			inet_peer_refcheck(peer);
			if ((u32)get_seconds() - peer->tcp_ts_stamp <= TCP_PAWS_MSL) {
				tp->rx_opt.ts_recent_stamp = peer->tcp_ts_stamp;
				tp->rx_opt.ts_recent = peer->tcp_ts;
			}
		}
	}
        //设置套接字中的目的端口和目的地址
	inet->inet_dport = usin->sin_port;
	inet->inet_daddr = daddr;

	inet_csk(sk)->icsk_ext_hdr_len = 0;
	if (inet_opt)
		inet_csk(sk)->icsk_ext_hdr_len = inet_opt->opt.optlen;

	tp->rx_opt.mss_clamp = TCP_MSS_DEFAULT;

	//设置sk的状态为TCP_SYN_SENT
	tcp_set_state(sk, TCP_SYN_SENT);
	err = inet_hash_connect(&tcp_death_row, sk);
	if (err)
		goto failure;

	rt = ip_route_newports(fl4, rt, orig_sport, orig_dport,
			       inet->inet_sport, inet->inet_dport, sk);
	if (IS_ERR(rt)) {
		err = PTR_ERR(rt);
		rt = NULL;
		goto failure;
	}
	/* OK, now commit destination to socket.  */
	sk->sk_gso_type = SKB_GSO_TCPV4;
	sk_setup_caps(sk, &rt->dst);

	if (!tp->write_seq)
		tp->write_seq = secure_tcp_sequence_number(inet->inet_saddr,
							   inet->inet_daddr,
							   inet->inet_sport,
							   usin->sin_port);

	inet->inet_id = tp->write_seq ^ jiffies;

	err = tcp_connect(sk);//创建SYN报文并发送，该函数实现过程挺复杂，需进行TCP连接初始化以及发送
	rt = NULL;
	if (err)
		goto failure;

	return 0;

failure:
	//失败处理
	tcp_set_state(sk, TCP_CLOSE);
	ip_rt_put(rt);
	sk->sk_route_caps = 0;
	inet->inet_dport = 0;
	return err;
}
```

### **3.2接受连接**

系统调用函数sys_accept实现如下：

```text
SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen)
{
	return sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```

调用系统调用sys_accept4

```text
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen, int, flags)
{
	struct socket *sock, *newsock;
	struct file *newfile;
	int err, len, newfd, fput_needed;
	struct sockaddr_storage address;
	.......................
	sock = sockfd_lookup_light(fd, &err, &fput_needed);//根据fd获得一个socket
	if (!sock)
		goto out;

	err = -ENFILE;
	newsock = sock_alloc();//重新创建一个新的socket
	if (!newsock)
		goto out_put;
	//复制套接字部分属性
	newsock->type = sock->type;
	newsock->ops = sock->ops;
	__module_get(newsock->ops->owner);
	//给新建的socket分配文件结构，并返回新的文件描述符
	newfd = sock_alloc_file(newsock, &newfile, flags);
	if (unlikely(newfd < 0)) {
		err = newfd;
		sock_release(newsock);
		goto out_put;
	}

	err = security_socket_accept(sock, newsock);
	if (err)
		goto out_fd;
	//调用inet_accept接受连接
	err = sock->ops->accept(sock, newsock, sock->file->f_flags);
	if (err < 0)
		goto out_fd;

	if (upeer_sockaddr) {//将地址信息从内核移到用户空间
		if (newsock->ops->getname(newsock, (struct sockaddr *)&address,
					  &len, 2) < 0) {
			err = -ECONNABORTED;
			goto out_fd;
		}
		err = move_addr_to_user((struct sockaddr *)&address,
					len, upeer_sockaddr, upeer_addrlen);
		if (err < 0)
			goto out_fd;
	}

	/* File flags are not inherited via accept() unlike another OSes. */
	//安装文件描述符
	fd_install(newfd, newfile);
	err = newfd;

out_put:
	fput_light(sock->file, fput_needed);
out:
	return err;
out_fd:
	fput(newfile);
	put_unused_fd(newfd);
	goto out_put;
}
```

该函数创建一个新的套接字，设置客户端连接并唤醒客户端并返回一个新的文件描述符fd。

下面是inet_accept函数的实现

```text
int inet_accept(struct socket *sock, struct socket *newsock, int flags)
{
	struct sock *sk1 = sock->sk;
	int err = -EINVAL;
	struct sock *sk2 = sk1->sk_prot->accept(sk1, flags, &err);//调用inet_csk_accept函数从队列icsk_accept_queue取出已经连接的套接字

	if (!sk2)
		goto do_err;

	lock_sock(sk2);

	sock_rps_record_flow(sk2);
	WARN_ON(!((1 << sk2->sk_state) &
		  (TCPF_ESTABLISHED | TCPF_CLOSE_WAIT | TCPF_CLOSE)));

	sock_graft(sk2, newsock);

	newsock->state = SS_CONNECTED;//设置套接字状态
	err = 0;
	release_sock(sk2);
do_err:
	return err;
}
```

## **4、关闭连接**

关闭一个socket连接，系统调用sys_shutdown

```text
SYSCALL_DEFINE2(shutdown, int, fd, int, how)
{
	int err, fput_needed;
	struct socket *sock;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock != NULL) {
		err = security_socket_shutdown(sock, how);
		if (!err)
			err = sock->ops->shutdown(sock, how);
		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

函数最后调用inet_shutdown关闭套接字

```text
int inet_shutdown(struct socket *sock, int how)
{
	struct sock *sk = sock->sk;
	int err = 0;
	.................
	lock_sock(sk);
	if (sock->state == SS_CONNECTING) {
		if ((1 << sk->sk_state) &
		    (TCPF_SYN_SENT | TCPF_SYN_RECV | TCPF_CLOSE))
			sock->state = SS_DISCONNECTING;
		else
			sock->state = SS_CONNECTED;
	}

	switch (sk->sk_state) {
	case TCP_CLOSE:
		err = -ENOTCONN;
	default:
		sk->sk_shutdown |= how;
		if (sk->sk_prot->shutdown)
			sk->sk_prot->shutdown(sk, how);//调用tcp_shutdown强制关闭连接
		break;

	/* Remaining two branches are temporary solution for missing
	 * close() in multithreaded environment. It is _not_ a good idea,
	 * but we have no choice until close() is repaired at VFS level.
	 */
	case TCP_LISTEN:
		if (!(how & RCV_SHUTDOWN))
			break;
		/* Fall through */
	case TCP_SYN_SENT:
		err = sk->sk_prot->disconnect(sk, O_NONBLOCK);//调用tcp_disconnect断开连接
		sock->state = err ? SS_DISCONNECTING : SS_UNCONNECTED;//设置套接字状态
		break;
	}

	sk->sk_state_change(sk);
	release_sock(sk);
	return err;
}
```

后面会详细分析TCP协议的发送和接收过程。

----

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549563171