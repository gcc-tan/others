
### vtun源代码
vtun的源代码进行编译之后会生成vtund的一个可执行文件（编译的方法比较简单，先读README然后里面给出了依赖的软件，确保已经安装。然后执行configure的shell脚本 可以带上--disable-xxx的参数，为了了解核心功能可以--disable-shaper， --disable-zlib等等。执行完毕之后会提示config正确与否，然后再执行config.status就会生成Makefile了，再执行make vtund就行了）

别着急看代码，先了解怎么用对看代码有帮助，vtund可以在两种模式下运行,这是官网直接拷贝过来的英文的意思已将相当简单了，不需要我这个蹩脚的翻译


	-s
	    Run as the server. 
	-i
	    Run as the inetd server. 
	-P port
	    Listen for connection on the specified port By default vtund listens on TCP port 5000. This options is equivalent to the 'port' option of config file. 

	 
	Client mode:

	-P port
	    Connect to the server on the specified port By default vtund connects to TCP port 5000. This options is equivalent to the 'port' option of config file. 
	-p
	    Reconnect to the server after connection termination. By default vtund will exit if connection has been terminated. This options is equivalent to the 'persist' option of config file. 
	-t timeout
	    Connect timeout Default is 30 seconds. This options is equivalent to the 'timeout' option of config file. 
	host
	    Host to authenticate with the server. Password and other information of this host will be taken from the config file. 
	server
	    Address of the server to connect to. Either IP address or domain name can be specified. 

	 
	FILES

	/etc/vtund.conf
	    Main configuration file with hosts and other information. See vtund.conf example provided with distribution and vtund.conf(5) for more information. 
	/var/lock/vtund/
	    Session lock files. 
	/var/log/vtund/
	    Connection statistic log files.
	    Format:
	       Date Uncomp_In Uncomp_Out Comp_In Comp_Out 

	 
	SIGNALS

	SIGHUP
	    This signal causes vtund(server mode) to reread config file. 
	SIGUSR1
	    This signal causes vtund to reset statistic counters. 

下面是一个配置文件的例子
serv.conf

	default
	{
		type tun;
		proto udp;
		keepalive yes;
	}
	conn
	{
		passwd tan5;
		up
		{
			ifconfig "%% 192.168.0.2 pointtopoint 192.168.0.1";
			route "add -net 192.168.0.0 netmask 255.255.255.0 gw 192.168.1.0";
		};
	}
cli.conf

	conn
	{
		pass tan5;
		up
		{
			ifconfig "%% 192.168.0.1 pointtopoint 192.168.0.2";
			route "add -net 192.168.0.0 netmask 255.255.255.0 gw 192.168.0.2";
		};

	}
然后运行服务器的话`./vtund conn -n -s -f conffile/serv.conf`。运行客户端`./vtund  conn 127.0.0.1 -n -f conffile/cli.conf`。这里有个小技巧，就是加上-n参数运行，这样进程就不会变成守护进程，日志信息的输入输出就就会出现在终端。其中conn的话是指明配置文件中的conn这一项表示指定连接，然后具体的可以参考[这里](http://vtun.sourceforge.net/conf.man.html)。配置文件主要分成两个部分，一个是options{}一个是default{} options是对vtund的全局的配置，default是针对每个连接所进行的默认的配置，然后这个配置会被运行时明确的指定的连接配置覆盖。例如前面例子的conn。这里的ifconfig就是设置本地的一个ip地址，以及对端的一个ip，这是虚拟的ip所以要用接下来的route命令添加一个路由表项

这里介绍两个很重要的结构体，从刚刚的配置文件中就可一看出来对应关系，vtun_host就是用来描tunnel连接用的，vtun_opts就是对应配置文件中vtund的一些全局的配置然后具体的作用可以见代码中的注释

	struct vtun_host 
	{
	   char *host;//本次连接配置文件的节点名，上面例子中的conn一样
	   char *passwd;//连接的密码，客户端的配置必须和服务器配置的一样
	   char *dev;//设备的名字,根据不同类型的tunnel打开不同的设备(pty，tun/tap)

	   llist up;//连接建立后执行的命令集合
	   llist down;//连接终止后执行的命令集合

	   int  flags;//各种状态标记位
	   int  timeout;
	   int  spd_in;
	   int  spd_out;
	   int  zlevel;
	   int  cipher;

	   int  rmt_fd;//代表远端连接的描述符号
	   int  loc_fd;//本地打开用于tunnel的描述符

	   /* Persist mode */
	   int  persist;

	   /* Multiple connections */
	   int  multi;

	   /* Keep Alive */
	   int ka_interval;//连接的超时时间
	   int ka_maxfail;

	   /* Source address */
	   struct vtun_addr src_addr;

	   struct vtun_stat stat;//本次连接的统计信息

	   struct vtun_sopt sopt;
	};

	
	/* Global options */
	struct vtun_opts 
	{
	   int  timeout;
	   int  persist;

	   char *cfg_file;

	   char *shell; 	 /* Shell */
	   char *ppp;		 /* Command to configure ppp devices */
	   char *ifcfg;		 /* Command to configure net devices */
	   char *route;		 /* Command to configure routing */
	   char *fwall; 	 /* Command to configure FireWall */
	   char *iproute;	 /* iproute command */

	   char *svr_name;       /* Server's host name */
	   char *svr_addr;       /* Server's address (string) */
	   struct vtun_addr bind_addr;	 /* Server should listen on this address */
	   int  svr_type;	 /* Server mode */
	   int  syslog; 	 /* Facility to log messages to syslog under */
	   int  quiet;		 /* Be quiet about common errors */
	};


在main.c中的主函数的主要函数调用图

	
	int main()
	{
		/*
		* 初始化vtun，和default_host ,并根据命令行参数填入vtun
		*/
		reread_config(int 0);
		clear_nat_hack_flags(int svr);
		if(!svr)
		{
			find_host(char *hst);
		}
		if(daemon)
		{
			//让进程变成守护进程
		}
		if(svr)
		{
			void server(int sock)
			{
				//服务器运行的这两种方式对应的就是-s -i 指定的模式
				if(vtun.svr_type == VTUN_STAND_ALONE)
				{
					listener();
				}
				else if(vtun.svr_type == VTUN_INETD)
				{
					connection(int sock);
				}
			}
		}
		else
		{
			client(struct vtun_host *host);
		}
	}
#### 服务器的大概流程	
然后再看看listener函数中做了什么

	void listener(void)
	{
		/* Set address by interface name, ip address or hostname */
		generic_addr(struct sockaddr_in *addr, struct vtun_addr *vaddr);
		socket(...);
		setsockopt(...);
		bind(...);
		listen(...);
		while(...)
		{
			s1 = accept(...);
			switch(fork())
			{
				case 0:
					connection(s1);
					break;
				default:
					...
			}
		}
	}


很明显listener所干的事情就是典型的tcp服务器的工作流程，然后实际去处理请求的还是子进程执行的connection函数里面。那接下来重点看connection函数

	void conection(int sock)
	{
		getpeername(sock, (struct sockaddr *) &cl_addr, &opt);
		getsockname(sock, (struct sockaddr *) &my_addr, &opt);
		io_init();
	
		if(struct vtun_host *host = auth_server(sock))//这个host是描述客户端的结构体
		{
			/*
			*host加入下面的初始化代码
			*host->rmt_fd = sock
			*host->sopt.laddr = strdup(inet_ntoa(my_addr.sin_addr));
			*host->sopt.lport = vtun.bind_addr.port;
			*host->sopt.raddr = strdup(ip);
			*/
			tunnel(host);
	
			/* Unlock host. (locked in auth_server) */	
			unlock_host(host);
		}
	}

下面是tunnel.c中tunnel函数的内容

	int tunnel(struct vtun_host *host)
	{
		if( !interface_already_open)
		{
			switch(host->flag & VTUN_TYPE_MASK)
			{
				case VTUN_TTY:
					pty_open(char *dev);
					break;
				case VTUN_PIPE:
					pipe_open(int fd);
					break;
				case VTUN_ETHER:
					tap_open(char *dev)
					break;
				case VTUN_TUN:
					tun_open(char *dev);
			}
		}
		switch( host->flags & VTUN_PROT_MASK )
		{
			case VTUN_TCP:
				//初始化proto_write,proto_read两个函数指针
				break;
			case VTUN_UDP:
				udp_session(host);
				//初始化proto_write,proto_read两个函数指针
		}
		/*
		*这里还有一段fork，父进程继续执行，子进程执行llist_trav(&host->up, run_cmd, &host->sopt);
		*这个函数是是执行配置文件中up里面所有的命令
		*/
		
		/*
		*根据 host->flags & VTUN_TYPE_MASK 初始化函数指针dev_read,dev_write	
		*/
		
		
		linkfd(struct vtun_host *host);
		
		/*
		*结尾工作，执行配置文件中down配置的命令，然后关掉对应的描述符
		*/
	}

再接着介绍linkfd.c中函数linkfd

	int linkfd(struct vtun_host *host)
	{
		/* Build modules stack */
	     if(host->flags & VTUN_ZLIB)
	     	lfd_add_mod(lfd_mod *&lfd_zlib);
	     /*
	     *类似于上面模块加载的代码就不贴出来了，都是通过判断host的flag标志，然后将配置的模块添加进入一个双向链表，一共有zlib,lzo,encrypt|legacy_encrypt,shaper，前两个是压缩，第三个是加密，第四个是流量控制
	     */
	     lfd_alloc_mod(host);//初始化每个模块 
	     
	     void lfd_linker()
	     {
	     	int fd1 = lfd_host->rmt_fd;
     		int fd2 = lfd_host->loc_fd; 
	     	buf = lfd_alloc(VTUN_FRAME_SIZE + VTUN_FRAME_OVERHEAD);
	     	while(!linker_term)
	     	{
	     		select(maxfd, &fdset, NULL, NULL, &tv);
	     		/*
	     		*主要就是从网络(fd1)读取数据，解密，然后写入本机的设备(fd2)
	     		*/
	     		if ( FD_ISSET(fd1, &fdset) && lfd_check_up() )
	     		{
	     			len=proto_read(fd1, buf);
	     			len = len & VTUN_FSIZE_MASK;
	     			/*
	     			*fl = len & ~VTUN_FSIZE_MASK;
	     			*然后vtun通过这个fl判断收到的数据到底是什么类型的，然后做不同的操作，一共5种类型，VTUN_BAD_FRAME，VTUN_ECHO_REQ，VTUN_ECHO_REP，VTUN_CONN_CLOSE和正常的数据包
	     			*/
	     			len=lfd_run_up(len,buf,&out);
	     			dev_write(fd2,out,len);
	     		}
	     		//和上面的代码的过程很相似，就不重复了
	     		/* Read data from the local device(fd2), encode and pass it to  the network (fd1) */
	     		
	     	}
	     }
	     lfd_free(buf);
	}

这个代码真的写得很厉害，把这个附加的功能(注释里有)抽象成一个结构体，然后结构体里面有函数指针，并且把这些结构体串成一个双向链表，每当有读入的时候就从尾到头遍历链表，然后执行对应的函数指针去处理读进来的数据，然后如果有数据写出的时候就反向遍历该链表

#### 客户端的大概流程

	void client(struct vtun_host *host)
	{
		while()
		{
			socket(AF_INET,SOCK_STREAM,0);
			bind(s,(struct sockaddr *)&my_addr,sizeof(my_addr);
			if(connect_t(s,(struct sockaddr *) &svr_addr, host->timeout))//这个是非阻塞的连接
			{
			}
			else
			{
				if( auth_client(s, host) )
				{
					client_term = tunnel(host);
				}
			}
		}
	}

代码看到这里然后先提出一下目前心中的问题：

1. 客户端与服务器的工作流程到底是什么呢？
这个首先是客户端创建tcp非阻塞连接去连接服务器指定端口，然后服务器有两种工作模式，一种工作模式是常规的监听某个端口，有请求的时候创建一个子进程，子进程去处理请求。另一种是inet守护进程模式，就是修改xx的配置文件，向inetd守护进程注册，然后监听的工作由inetd完成，然后客户端请求到来的时候由inetd去exec执行服务器代码。服务器接收连接后会对客户端身份进行认证，认证成功后，根据配置文件会打开用于tunnel的设备，设置读取网络数据的函数(tcp|udp),执行配置文件中配置的启动命令，然后在select模式下循环(等待描述符可读，如果网络可读，从网络中读取数据，处理（解密等）后写入设备;如果设备可读，从设备中读取，处理(加密等)写入网络)，循环推出后执行配置文件中关闭的命令。客户端连接成功后也是执行与服务器一模一样的select的循环

2.  客户端与服务器的认证过程？
这个过程比较简单，认证的工作在auth_server和auth_client里面。首先由服务器发送`“VTUN server ver 3.X 03/22/2017\n”`版本信息的字符串，然后客户端读取并且回应`"HOST: conn\n"`conn就是配置文件中代表本次连接的名字，接着服务器发送`“OK CHAL: xxxx\n”`其中的xxxx是用随机数随机一个字节数组，然后将字节数组用一个简单的算法转成的字符数组。客户端接收到了这个数据然后将xxxx又转换成字节数组，然后自己的密码(配置文件中的passwd)加密这个字节数组，并且用之前的算法转换成字符串yyyy发送`"CHAL: yyyy\n"`。最后服务器收到yyyy后转换成字节数组，用之前的连接的名字(conn)找到自己配置文件中与之对应的项取出passwd，然后用passwd解密这个字节数组，并且比较之前发送的字节数组与接受到解密后的是否相同，相同认证完成

3. 


















