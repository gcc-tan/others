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

### c++的vector

今天写代码使用了一下vector，结果在使用vector的size函数的时候发现了一个问题——size函数返回值是unsigned int类型的，是一个无符号的整数，与整数进行减法运算的时候千万要注意溢出的问题。出错的地方像下面一样，大概是这个样子的。解决方法要么就进行转换，要么就是移项减法变成加法。

```
#include<iostream>
#include<vector>
using namespace std;
int main()
{
	vector<int> v(2,0);
	unsigned char t = 2;
	cout<<t - 3<<endl;//这个没有溢出是因为常量是int型，t被直接转换成int了
	cout<<v.size() - 3<<endl;
	return 0;
}
```
运行结果是

```
tan@tan:~/Documents/git_dir/leetcode$ ./a.out 
-1
18446744073709551615
```

### 溢出与相反数

先看一段代码

```
#include<stdio.h>
int main()
{
	int a = 1 << 31;
	long long b = a;
	a = -a;
	b = -b;
	printf("a:%d\n", a);
	printf("b:%lld\n", b);
	return 0;
}
```
```
tan@tan:~/Documents/git_dir/leetcode$ ./a.out 
a:-2147483648
b:2147483648
```
上面的代码很简单，就是大家常用的求相反数的过程，运行的结果却是很让人吃惊。其实问题就在溢出这块，a乘上-1之后的值由于补码的原因不能表示，溢出了。因此常见的等式是INT_MIN = -INT_MAX - 1这个不光是int类型，其他类型也一样。


### Linux终端下后台运行的程序被stopped

今天写一个使用I/O复用的tcp服务器时碰到了一个很有意思的问题。tcp服务器完成之后，使用nc命令测试连接情况，该命令是`nc localhost 8888 &`。第一个次执行没有问题，第二次执行之后terminal提示之前的命令已经stopped，第三次执行告诉你第二次执行的命令被stopped。然后我就很奇怪，以为是代码问题，找了好久，也没有找到错误，然后在网上找资料，**才发现在《Unix 环境高级编程》第9.8节作业控制中讲到，“如果后台程序试图读取终端，这并不是一个错误，但是终端驱动程序将检测这种情况，并向后台作业发送一个特定信号SIGTTIN，该信号会停止此后台程序，并向用户发送通知”。**


为了验证上面的理论我写了一个很简单的代码

```
#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
void handler(int signo)
{
	printf("catch SIGTTIN\n");
	exit(1);
}
int main()
{
	char c;
	struct sigaction act;

	act.sa_handler = handler;
	act.sa_flags = 0;
	sigemptyset(&act.sa_mask);

	sigaction(SIGTTIN, &act, 0);

	while(1)
	{
		scanf("%c", &c);
		if(c == 'q') break;
	}
	return 0;
}

```
下面是上面代码的输出

```
tan@tan:~/Documents/code/net$ ./a.out &
[1] 20288
tan@tan:~/Documents/code/net$ catch SIGTTIN
```
面对这种情况，解决方案也很简单

+ 在程序中去除读终端的相关代码；
+ 将标准输入重定向
+ 使用nohup，nohup将忽略该程序的输入，并将输出追加到nohup.out。

结论是APUE真的是一本神书啊，平时遇到的好多问题都能在上面找到结果。








