###uint32_t 转换成uint8_t的血案

之前一直以为指针的强制转换只要注意点应该是不会有坑的，今天写了个发现被坑大发了。
```
	uint16_t src[] = {0x4500,0x003c,0x6cf2,0x0000,0x0111,0x0392,0x6442,0x03ef,
0xe000,0x00fc,0xe58b,0x14eb,0x0028,0x8111,0x7ffb,0x0000,
0x0001,0x0000,0x0000,0x0000,0x0e44,0x7574,0x7068,0x6f6e,
0x656c,0x6162,0x2d70,0x6300,0x001c,0x0001};//这是一个ip header(20)+upd header(8)+data(22)的报文，是tcpdump截取之后用sed，awk处理了才的出来的。这就是原始的内存中存放的数据
	uint8_t *data = (uint8_t *)src;

```
其他部分的代码就不贴出来了不是关键的部分，按照常理觉得前两个字节是0x4500,那么转换之后data的第一个字节应该是0x45，第二个字节是0x00。很不巧，在ubuntu的系统下结果恰好是相反的，因为linux的系统是小端的字节序，所谓小端字节序列大家应该很清楚，就是高地址位置存放的是MSB(most significant bit),所以`data[0] = 0x00 data[1] = 0x45`,而tcp/ip的字节序是大端的所以结果恰好相反


就是一个这么简单的问题，呵呵运行结果错得离谱


### inet_ntoa函数的血案
今天写了个类似于这个样子的
```
#include<stdio.h>
#include<netinet/in.h>
#include<arpa/inet.h>
int main(int argc,char **argv)
{
	struct in_addr addr,mask;
	char *s1,*s2;
	addr.s_addr = 1;
	mask.s_addr = 2;
	printf("s1:%s s2:%s\n",inet_ntoa(addr),inet_ntoa(mask));

	return 0;
}

```
运行结果是这样的
```
tan@ttt:~/Desktop$ ./a.out 
s1:1.0.0.0 s2:1.0.0.0
```
为什么不是一个是1.0.0.0，一个是2.0.0.0呢，没看inet_ntoa的代码实现，估计是函数静态分配的一个缓冲区域，所以每次函数调用返回值是指向同一个缓冲区，而调用printf的时候，参数从右边向左入栈，入栈的过程中发现函数调用，调用函数后在栈内存储相应的结果

真是迷指错误

#### 向未绑定的udp端口写数据
今天写代码的时候，创建了一个udp套接字，然后向connect对端的一个ip和端口，然后开始写数据，写数据这块没什么问题出现，然后读返回数据的时候读出了一个errmsg "Connnection refused",这里其实有一个问题值得思考。udp协议是没有出错控制的信息的，那么这个信息是怎么产生的。这是通过icmp协议产生的，目的主机没有这个服务端口就通过icmp错误消息返回，当源主机收到这个消息，就找到对应连接的套接字返回这个错误。




















