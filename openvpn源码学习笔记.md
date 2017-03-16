## openvpn 源码阅读
###工作流程


openvpn原理如下：

openvpn驱动部分实现了网卡处理和字符设备。网卡处理网络数据，字符设备完成与应用层的数据交互。
使用openvpn必须修改路由表
工作过程
 发送数据:
应用程序发送网络数据
网络数据根据修改后的路由表把数据路由到虚拟网卡
虚拟网卡把数据放到数据队列中
字符设备从数据队列中取数据，然后送给应用层
应用层把数据转发给物理网卡
物理网卡发送数据
接收过程：
物理网卡接受到数据，并传到应用空间
应用守护程序通过字符设备，把数据传给驱动网卡
数据通过虚拟网卡重新进入网络堆栈
网络堆栈把数据传给上层真实的应用程序。
 
基本上所有的VPN都是这么实现的：

1.  更改路由表，通过应用程序调用API函数更改和添加路由表。将数据包都先发送到虚拟网卡。

2.  虚拟网卡得到数据包，先判断该数据包是什么类型的数据包。因为本身是一个物理的虚拟网卡，它必须去处理ARP，DHCP等一些数据包，用于欺骗上层网络，否则该网卡将不能工作，这一点比IMD的驱动要麻烦一些。
3. 虚拟网卡判断得到的数据包是普通的Tcp和Udp数据包后，将该数据包压入一个堆栈。该堆栈是一个和应用程序（通常这个应用程序被称为守护程序）共享的内存块。然后驱动Set一个Event。上层应用程序Wait这个Event。并从这个堆栈中取得这个数据包。
4. 上层应用程序得到这个数据包后，修改修改这个数据包的内容，然后建立一个Socket把该数据包重新封装成TCP或者UDP包后从物理网卡发送到Vpn的服务器上，由服务器再进行处理。转发到最终的目的IP。

假设我们要打开一个IE，那么数据包的路线如下所示。
ie－vnic－vpn client－tcp/udp－vpn网关－route－web服务器
  接收的时候正好相反。

### 主要结构体

+ `struct context` 存储与其相关的VPN tunnel的状态信息和进程配置信息的数据
+ 

### 文件的大概功能

+ perf.c 文件中定义了一个perf的结构体和perf_set的结构体，perf里面包含了时间和状态等统计信息，perf_set包含了一个栈结构和一个perf数组，**这个估计是对整个程序的 性能和状态的记录**
+ openvpn.c 整个程序的入口文件，可运行在不同的模式下，客户端模式和服务器模式


### 函数的调用结构
客户端模式的处理函数,只是列出了关键的参数
	
	tunnel_point_to_point(struct context *c)
	{
		perf_push();
		
		io_wait(c)
		{
			io_wait_dowork()
			{
				socket_set();
				tun_set();
			}
		}
		
		process_io(c)
		{
			process_outgoing_link()
			{
				link_socket_write()
				{
					link_socket_write_udp();
					
					link_socket_write_tcp();
				}
			}
			
			process_outgoing_tun()
			{
				write_tun_buffered()
				write_tun()
			}
			
			read_incoming_link()
			process_incoming_link()
			{
				process_incoming_link_part1()
				{
					tls_pre_decrypt();
					
					openvpn_decrypt();
					
					
				}
				process_incoming_link_part2()
				{
					fragment_incoming();
				}
				
			}
			
			read_incoming_tun()
			process_incoming_tun()
			{
				encrypt_sign()
			}
		}
		
		perf_pop();
	}

首先来看process_io里面的几个主要的读写函数，这个肯定是整个代码的核心

+ `process_outgoing_link`主要就是根据context的值，调用udp或者是tcp进行写缓冲的数据

+ `process_out_going_tun`主要就是对虚拟网卡设备进行操作，大多数的操作系统都支持虚拟网卡，但是每个系统的操作没有形成统一的标准，那么可以看到tun.c文件里里面对write_tun的unix系统有很多的版本
+ `process_incoming_link`首先`tls_pre_decrypt()`来确定数据包还是控制包。如果是数据包，相关的安全参数会被加载，如果是控制包，该包的缓冲区的长度会被置零，之后对数据的处理会过程都会被忽略

整个程序的入口文件

	openvpn_main()
	{
		
	}
