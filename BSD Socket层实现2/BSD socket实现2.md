内核缓冲区与用户缓冲区之间的数据复制：

```c
1、move_addr_to_kernel
2、move_addr_to_user
//这两个函数主要通过调用底层函数memcpy_tofs和memcpy_fromfs实现
```



```c
/*下面两个函数实现地址用户空间和内核空间地址之间的相互移动*/
//从uaddr拷贝ulen大小的数据到kaddr
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
//注意的是，从内核拷贝数据到用户空间是值-结果参数
//ulen这个指向某个整数变量的指针，当函数被调用的时候，它告诉内核需要拷贝多少
//函数返回时，该参数作为一个结果，告诉进程，内核实际拷贝了多少信息
static int move_addr_to_user(void *kaddr, int klen, void *uaddr, int *ulen)
{
	int err;
	int len;
 
	//判断ulen指向的存储块是否可写，就是判断ulen是否可作为左值	
	if((err=verify_area(VERIFY_WRITE,ulen,sizeof(*ulen)))<0)
		return err;
	len=get_fs_long(ulen);//len = *ulen，ulen作为值传入，告诉要拷贝多少数据
	if(len>klen)
		len=klen;//供不应求，按供的算。实际拷贝的数据
	if(len<0 || len> MAX_SOCK_ADDR)
		return -EINVAL;
	if(len)
	{
	//判断uaddr用户空间所指的存储块是否可写
		if((err=verify_area(VERIFY_WRITE,uaddr,len))<0)
			return err;
		memcpy_tofs(uaddr,kaddr,len);//实质是调用memcpy
	}
 	put_fs_long(len,ulen);//*ulen = len，作为结果返回，即实际拷贝了多少数据
 	return 0;
}

```





```c
/*
socki_lookup与sockfd_lookup函数
函数功能：socket结构查询                                                                            
*/
inline struct socket *socki_lookup(struct inode *inode)
{
    return &inode->u.socket_i
}

/*socketfd_lookup函数与socki_lookup相同，只不过socketfd_lookup函数是通过文件描述符得到对应的file结构，进而得到inode结构再调用socki_lookup函数
*/
```





```c
/*
  sock_socket函数是socket系统调用BSD socket层对应的底层实现函数
  主要完成工作：
      1、分配socket、sock结构，这两个结构在网络栈的不同层次表示一个套接字连接
      2、分配inode、file结构用于普通文件
      3、分配一个文件描述符并返回给应用程序
*/

static int sock_socket(int family, int type, int protocol)
{
	int i, fd;
	struct socket *sock;
	struct proto_ops *ops;

	/* Locate the correct protocol family. */
    //匹配应用程序调用socket()函数时指定的协议
	for (i = 0; i < NPROTO; ++i) 
	{
		if (pops[i] == NULL) continue;
		if (pops[i]->family == family) 
			break;
	}
    //没有匹配到
	if (i == NPROTO) 
	{
  		return -EINVAL;
	}
    //根据family输入参数决定域操作函数集，用于ops字段的赋值
    //操作函数集与域相关，不同的域对应不同的操作函数集
	ops = pops[i];

/*
 *	Check that this is a type that we know how to manipulate and
 *	the protocol makes sense here. The family can still reject the
 *	protocol later.
 */
  。//套接字类型检查
	if ((type != SOCK_STREAM && type != SOCK_DGRAM &&
		type != SOCK_SEQPACKET && type != SOCK_RAW &&
		type != SOCK_PACKET) || protocol < 0)
			return(-EINVAL);

/*
 *	Allocate the socket and allow the family to set things up. if
 *	the protocol is 0, the family is instructed to select an appropriate
 *	default.
 */
   //sock_alloc分配socket结构体并初始化
	if (!(sock = sock_alloc())) 
	{
		printk("NET: sock_socket: no more sockets\n");
		return(-ENOSR);	/* Was: EAGAIN, but we are out of
				   system resources! */
	}
   //socket类型与函数操作集赋值
	sock->type = type;
	sock->ops = ops;
    //socket是通用套接字结构体，而sock与具体使用的协议相关
    //调用下层函数create分配下层sock结构
	if ((i = sock->ops->create(sock, protocol)) < 0) 
	{
		sock_release(sock);
		return(i);
	}
    //调用get_fd()分配一个文件描述符并返回
    //SOCK_INODE是根据
	if ((fd = get_fd(SOCK_INODE(sock))) < 0) 
	{
		sock_release(sock);
		return(-EINVAL);
	}

	return(fd);
}
```





```c
/*
socket函数并没有为套接字绑定本地地址和端口号，对于服务器端则必须显性绑定地址和端口号，bind函数主要是服务器使用，把一个本地协议地址赋值给套接字
bind函数的主要功能就是将socket套接字绑定指定的地址
*/
//fd就是套接字描述符，umyaddr表示需要绑定的地址结构，addrlen表示该结构体的长度
static int sock_bind(int fd, struct sockaddr *umyaddr, int addrlen)
{
	struct socket *sock;
	int i;
	char address[MAX_SOCK_ADDR];
	int err;

	if (fd < 0 || fd >= NR_OPEN || current->files->fd[fd] == NULL)
		return(-EBADF);
	   //获取fd对应的socket结构
	if (!(sock = sockfd_lookup(fd, NULL))) 
		return(-ENOTSOCK);
       //将地址从用户缓存区复制到内核缓存区，umyaddr->address
	if((err=move_addr_to_kernel(umyaddr,addrlen,address))<0)
	  	return err;
     //转调用bind指向的函数，也就是下层函数(inet_bind)
	if ((i = sock->ops->bind(sock, (struct sockaddr *)address, addrlen)) < 0) 
	{
		return(i);
	}
	return(0);
}
```







```c
/*
客户端，表示客户端向服务端发起连接请求
首先将要连接的源端地址从用户缓存区复制到内核缓冲区，之后根据套接字目前所处状态采取相应的措施
如果状态有效，则转调用connect函数
*/
//fd是客户端调用socket函数返回的套接字描述符
static int sock_connect(int fd, struct sockaddr *uservaddr, int addrlen)
{
	struct socket *sock;
	struct file *file;
	int i;
	char address[MAX_SOCK_ADDR];
	int err;

	if (fd < 0 || fd >= NR_OPEN || (file=current->files->fd[fd]) == NULL)
		return(-EBADF);
    //给定文件描述符返回socket结构以及file结构体指针
	if (!(sock = sockfd_lookup(fd, &file)))
		return(-ENOTSOCK);
    //用户地址空间数据拷贝到内核地址空间
	if((err=move_addr_to_kernel(uservaddr,addrlen,address))<0)
	  	return err;
  
	switch(sock->state) 
	{
		case SS_UNCONNECTED:
			/* This is ok... continue with connect */
			break;
		case SS_CONNECTED:
			/* Socket is already connected */
			if(sock->type == SOCK_DGRAM) /* Hack for now - move this all into the protocol */
				break;
			return -EISCONN;
		case SS_CONNECTING:
			/* Not yet connected... we will check this. */
		
			/*
			 *	FIXME:  for all protocols what happens if you start
			 *	an async connect fork and both children connect. Clean
			 *	this up in the protocols!
			 */
			break;
		default:
			return(-EINVAL);
	}
    //调用下一层函数inet_connect()
	i = sock->ops->connect(sock, (struct sockaddr *)address, addrlen, file->f_flags);
	if (i < 0) 
	{
		return(i);
	}
	return(0);
}
```















```c
 /*
 listen函数仅供服务器调用，把一个未连接的套接字转换成一个被动套接字，指示内核应接受指向套接字的连接请求*/

//fd表示bind后的套接字文件描述符 ,backlog表示排队的最大连接个数
static int sock_listen(int fd, int backlog)
{
	struct socket *sock;

	if (fd < 0 || fd >= NR_OPEN || current->files->fd[fd] == NULL)
		return(-EBADF);
    //根据给定的套接字文件描述符获取socket结构体
	if (!(sock = sockfd_lookup(fd, NULL))) 
		return(-ENOTSOCK);
    //前提是没有建立连接，该套接字已经建立连接就不需要了
	if (sock->state != SS_UNCONNECTED) 
	{
		return(-EINVAL);
	}
    //调用底层实现函数(inet_listen)
	if (sock->ops && sock->ops->listen)
		sock->ops->listen(sock, backlog);
    //设置标识字段，表示正在监听
	sock->flags |= SO_ACCEPTCON;
	return(0);
}
```





```c
/*
用于服务器接收一个客户端的连接请求
*/
//fd为监听后套接字，upeer_sockaddr用来返回已连接客户的协议地址
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
	if (sock->state != SS_UNCONNECTED) 
	{
		return(-EINVAL);
	}
	if (!(sock->flags & SO_ACCEPTCON))//没有listen
	{
		return(-EINVAL);
	}
    //分配一个新的套接字，用于后面进行通信的套接字
	if (!(newsock = sock_alloc())) 
	{
		printk("NET: sock_accept: no more sockets\n");
		return(-ENOSR);	/* Was: EAGAIN, but we are out of system
				   resources! */
	}
    //初始化进行通信的套接字，初始化主要来自监听套接字中原有的信息
	newsock->type = sock->type;
	newsock->ops = sock->ops;
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
    //如果函数传入的upeer_sockaddr指针参数不为NULL,表示应用程序需要返回通信远端的地址
	if (upeer_sockaddr)
	{
        //调用sock-ops-getname函数从请求数据包中取得远端地址
        //并使用move_addr_to_user函数将地址复制到用户缓存区中
		newsock->ops->getname(newsock, (struct sockaddr *)address, &len, 1);
		move_addr_to_user(address,len, upeer_sockaddr, upeer_addrlen);
	}
	return(fd);
}
```



```c
/*
发送数据给指定的远端地址，主要用于udp协议
向指定地址的远端发送数据包，将buff缓冲区中len大小的数据发送给addr指定的远端套接字
*/
//第一个参数是套接字描述符，第二个参数指向缓冲区的指针
//第三个参数读写字节数，addr指向一个含有数据包接收者的协议地址的套接字地址结构

static int sock_sendto(int fd, void * buff, int len, unsigned flags,
	   struct sockaddr *addr, int addr_len)
{
	struct socket *sock;
	struct file *file;
	char address[MAX_SOCK_ADDR];
	int err;
	
	if (fd < 0 || fd >= NR_OPEN || ((file = current->files->fd[fd]) == NULL))
		return(-EBADF);
    //找到给定文件描述符对应的socket结构
	if (!(sock = sockfd_lookup(fd, NULL)))
		return(-ENOTSOCK);

	if(len<0)
		return -EINVAL;
	err=verify_area(VERIFY_READ,buff,len);
	if(err)
	  	return err;
  	//从addr拷贝add_len大小的数据到address
	if((err=move_addr_to_kernel(addr,addr_len,address))<0)
	  	return err;
    //调用下层函数sendto(inet_sendto)
	return(sock->ops->sendto(sock, buff, len, (file->f_flags & O_NONBLOCK),
		flags, (struct sockaddr *)address, addr_len));
}

```







```c
/*
从指定的远端地址接收数据，主要用于udp协议
从addr指定的源端接收len大小的数据，而后缓存到buff缓冲区
函数返回远端地址信息，存放在addr指定的地址结构中
*/


static int sock_recvfrom(int fd, void * buff, int len, unsigned flags,
	     struct sockaddr *addr, int *addr_len)
{
	struct socket *sock;
	struct file *file;
	char address[MAX_SOCK_ADDR];
	int err;
	int alen;
	if (fd < 0 || fd >= NR_OPEN || ((file = current->files->fd[fd]) == NULL))
		return(-EBADF);
    //通过文件描述符找到对应的socket结构
	if (!(sock = sockfd_lookup(fd, NULL))) 
	  	return(-ENOTSOCK);
	if(len<0)
		return -EINVAL;
	if(len==0)
		return 0;
   //检查缓冲区是否可写
	err=verify_area(VERIFY_WRITE,buff,len);
	if(err)
	  	return err;
   //调用下层函数inet_recvfrom
	len=sock->ops->recvfrom(sock, buff, len, (file->f_flags & O_NONBLOCK),
		     flags, (struct sockaddr *)address, &alen);

	if(len<0)
	 	return len;
    //将发送端的地址信息填充到addr中
	if(addr!=NULL && (err=move_addr_to_user(address,alen, addr, addr_len))<0)
	  	return err;

	return len;
}

```


