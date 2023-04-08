socket 函数并没有为套接字绑定本地地址和端口号，对于服务器端则必须显性绑定地址和端口号。bind 函数主要是服务器端使用，把一个本地协议地址赋予套接字。

## 1、应用层——bind 函数

```text
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
/*sockfd是由socket函数返回的套接口描述字，第二个参数是一个指向特定于协议的地址结构的指针，第三个参数是该地址结构的长度*/
```

bind 函数的功能则是将socket 套接字绑定指定的地址。

## 2、BSD Socket 层——sock_bind 函数

同样是通过一个共同的入口函数 sys_socket

```text
/*
 *	Bind a name to a socket. Nothing much to do here since it's
 *	the protocol's responsibility to handle the local address.
 *
 *	We move the socket address to kernel space before we call
 *	the protocol layer (having also checked the address is ok).
 */
 //bind函数对应的BSD层函数，用于绑定一个本地地址，服务器端
 //umyaddr表示需要绑定的地址结构，addrlen表示改地址结构的长度
 //这里的fd，即为套接字描述符
static int sock_bind(int fd, struct sockaddr *umyaddr, int addrlen)
{
	struct socket *sock;
	int i;
	char address[MAX_SOCK_ADDR];
	int err;
    //套接字参数有效性检查
	if (fd < 0 || fd >= NR_OPEN || current->files->fd[fd] == NULL)
		return(-EBADF);
	//获取fd对应的socket结构
	if (!(sock = sockfd_lookup(fd, NULL))) 
		return(-ENOTSOCK);
    //将地址从用户缓冲区复制到内核缓冲区，umyaddr->address
	if((err=move_addr_to_kernel(umyaddr,addrlen,address))<0)
	  	return err;
    //转调用bind指向的函数，下层函数(inet_bind)
	if ((i = sock->ops->bind(sock, (struct sockaddr *)address, addrlen)) < 0) 
	{
		return(i);
	}
	return(0);
}
```

sock_bind 函数主要就是将用户缓冲区的地址结构复制到内核缓冲区，然后转调用下一层的bind函数。

该函数内部的一个用户空间与内核数据空间数据拷贝的函数

```text
//从uaddr拷贝ulen大小的数据到kaddr，实现地址用户空间到内核地址空间的数据拷贝
static int move_addr_to_kernel(void *uaddr, int ulen, void *kaddr)
{
	int err;
	if(ulen<0||ulen>MAX_SOCK_ADDR)
		return -EINVAL;
	if(ulen==0)
		return 0;
	//检查用户空间的指针所指的指定大小存储块是否可读
	if((err=verify_area(VERIFY_READ,uaddr,ulen))<0)
		return err;
	memcpy_fromfs(kaddr,uaddr,ulen);//实质是memcpy函数
	return 0;
}
```

## 3、INET Socket 层——inet_bind 函数

```text
/* this needs to be changed to disallow
   the rebinding of sockets.   What error
   should it return? */
//完成本地地址绑定，本地地址绑定包括IP地址和端口号两个部分
static int inet_bind(struct socket *sock, struct sockaddr *uaddr,int addr_len)
{
	struct sockaddr_in *addr=(struct sockaddr_in *)uaddr;
	struct sock *sk=(struct sock *)sock->data, *sk2;
	unsigned short snum = 0 /* Stoopid compiler.. this IS ok */;
	int chk_addr_ret;
 
	/* check this error. */
	//在进行地址绑定时，该套接字应该处于关闭状态
	if (sk->state != TCP_CLOSE)
		return(-EIO);
	//地址长度字段校验
	if(addr_len<sizeof(struct sockaddr_in))
		return -EINVAL;
 
    //非原始套接字类型，绑定前，没有端口号，则绑定端口号
	if(sock->type != SOCK_RAW)
	{
		if (sk->num != 0)//从inet_create函数可以看出，非原始套接字类型，端口号是初始化为0的 
			return(-EINVAL);
 
		snum = ntohs(addr->sin_port);//将地址结构中的端口号转为主机字节顺序
 
		/*
		 * We can't just leave the socket bound wherever it is, it might
		 * be bound to a privileged port. However, since there seems to
		 * be a bug here, we will leave it if the port is not privileged.
		 */
		 //如果端口号为0，则自动分配一个
		if (snum == 0) 
		{
			snum = get_new_socknum(sk->prot, 0);//得到一个新的端口号
		}
		//端口号有效性检验，1024以上，超级用户权限
		if (snum < PROT_SOCK && !suser()) 
			return(-EACCES);
	}
	//下面则进行ip地址绑定
	//检查地址是否是一个本地接口地址
	chk_addr_ret = ip_chk_addr(addr->sin_addr.s_addr);
	//如果指定的地址不是本地地址，并且也不是一个多播地址，则错误返回
	if (addr->sin_addr.s_addr != 0 && chk_addr_ret != IS_MYADDR && chk_addr_ret != IS_MULTICAST)
		return(-EADDRNOTAVAIL);	/* Source address MUST be ours! */
	//如果没有指定地址，则系统自动分配一个本地地址  	
	if (chk_addr_ret || addr->sin_addr.s_addr == 0)
		sk->saddr = addr->sin_addr.s_addr;//本地地址绑定
	
	if(sock->type != SOCK_RAW)
	{
		/* Make sure we are allowed to bind here. */
		cli();
	
		//for循环主要是检查检查有无冲突的端口号以及本地地址，有冲突，但不允许地址复用，肯定错误退出
		//成功跳出for循环时，已经定位到了哈希表sock_array指定索引的链表的末端
		for(sk2 = sk->prot->sock_array[snum & (SOCK_ARRAY_SIZE -1)];
					sk2 != NULL; sk2 = sk2->next) 
		{
		/* should be below! */
			if (sk2->num != snum) //没有重复，继续搜索下一个
				continue;//除非有重复，否则后面的代码将不会被执行
			if (!sk->reuse)//端口号重复，如果没有设置地址复用标志，退出
			{
				sti();
				return(-EADDRINUSE);
			}
			
			if (sk2->num != snum) 
				continue;		/* more than one */
			if (sk2->saddr != sk->saddr) //地址和端口一个意思
				continue;	/* socket per slot ! -FB */
			//如果状态是LISTEN表明该套接字是一个服务端，服务端不可使用地址复用选项
			if (!sk2->reuse || sk2->state==TCP_LISTEN) 
			{
				sti();
				return(-EADDRINUSE);
			}
		}
		sti();
 
		remove_sock(sk);//将sk sock结构从其之前的表中删除，inet_create中 put_sock，这里remove_sock
		put_sock(snum, sk);//然后根据新分配的端口号插入到新的表中。可以得知系统在维护许多这样的表
		sk->dummy_th.source = ntohs(sk->num);//tcp首部，源端口号绑定
		sk->daddr = 0;//sock结构所代表套接字的远端地址
		sk->dummy_th.dest = 0;//tcp首部，目的端口号
	}
	return(0);
}
```

inet_bind 函数即为bind函数的最底层实现，该函数实现了本地地址和端口号的绑定，其中还针对上层传过来的地址结构进行校验，检查是否冲突可用。需要清楚的是 sock_array数组，这其实是一个链式哈希表，里面保存的就是各个端口号的sock结构，数组大小小于端口号，所以采用链式哈希表存储。

bind 函数的各层分工很明显，主要就是inet_bind函数了，在注释里说的很明确了，bind 是绑定本地地址，它不负责对端地址，一般用于服务器端，客户端是系统指定的。

一般是服务器端调用这个函数，到了这一步，服务器端套接字绑定了本地地址信息（ip地址和端口号），但是不知道对端（客户端）的地址信息。

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/549864166