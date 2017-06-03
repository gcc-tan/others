### gdb调试常用命令
#### 启动gdb
```
$gdb <program>

$gdb
```
program是要调试的程序，程序编译的时候记得带上-g参数，这样gdb调试的时候才有调试信息

####运行
+ run : 简记为 r ，其作用是运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令
+ continue （简写c ）：继续执行，到下一个断点处（或运行结束）
+ next：（简写 n）单步跟踪程序，当遇到函数调用时，也不进入此函数体；此命令同 step 的主要区别是，step 遇到用户自定义的函数，将步进到函数中去运行，而 next 则直接调用函数，不会进入到函数体内。
+ step （简写s）：单步调试如果有函数调用，则进入函数；与命令n不同，n是不进入调用的函数的
+ until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体。
+ until+行号： 运行至某行，不仅仅用来跳出循环
+ finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。
+ call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)
+ quit：简记为 q ，退出gdb


#### 设置断点
+ break n （简写b n）:在第n行处设置断点（可以带上代码路径和代码名称： b test.cpp:578）
+ b [line or function] if a＞b：条件断点设置，满足条件的时候才被激活
+ break func（break缩写为b）：在函数func()的入口处设置断点，如：break cb_button
+ delete 断点号n：删除第n个断点
+ disable 断点号n：暂停第n个断点
+ enable 断点号n：开启第n个断点
+ clear [line or function] ：清除行号或者函数上面的断点
+ info b （info breakpoints） ：显示当前程序的断点设置情况
+ delete breakpoints：清除所有断点

#### 查看源代码
+ list ：简记为 l ，其作用就是列出程序的源代码，默认每次显示10行。
+ list 行号：将显示当前文件以“行号”为中心的前后10行代码，如：list 12
+ list 函数名：将显示“函数名”所在函数前后源代码，如：list main
+ list ：不带参数，将接着上一次 list 命令的，输出下边的内容。
+ list - : 表示查看上次显示的前10行代码

#### 打印表达式
+ print 表达式：简记为 p ，其中“表达式”可以是任何当前正在被测试程序的有效表达式，比如当前正在调试C语言的程序，那么“表达式”可以是任何C语言的有效表达式，包括数字，变量甚至是函数调用。
```
print [/format] <expr>
format
    x 按十六进制格式显示变量
    d 按十进制格式显示变量
    u 按十六进制格式显示无符号整型
    o 按八进制格式显示变量
    t 按二进制格式显示变量
    a 按十六进制格式显示变量
    c 按字符格式显示变量
    f 按浮点数格式显示变量
```
```
int a[5] = {1,2,3,4,5}
(gdb) print/x  a
(gdb) print *a@5 //显示一个数组的方法a表示的是开始地址
$4 = {1, 2, 3, 4, 5}
```

+ display 表达式：在单步运行时将非常有用，使用display命令设置一个表达式后，它将在每次单步进行指令后，紧接着输出被设置的表达式及值。如： display a（如果不想显示的话用undisplay number 这个number就是被显示表达式前面的number）
+ watch 表达式：设置一个观察点，一旦被观察的“表达式”的值改变（会打印出改变前后的值），gdb将暂停到值改变的地方。如： watch a，类似的命令还有awatch，rwatch，awatch是变量被读写的时候，rwatch是变量被读的时候 。感觉就像条件断点break if change吗。清除的办法很简单delete number，number可以用info watchpoints命令
+ whatis ：查询变量或函数
+ info locals： 显示当前堆栈页的所有变量


####查看运行信息
+ where/bt ：当前运行的堆栈列表；
+ bt backtrace 显示当前调用堆栈
+ up/down 改变堆栈显示的深度
+ set args 参数:指定运行时的参数
+ show args：查看设置好的参数
+ info program： 来查看程序的是否在运行，进程号，被暂停的原因。

#### 调试信号
在使用gdb调试程序时，缺省情况下信号会被gdb截获，导致要调试的程序无法接收到信号，我们可以使用info handle来查看信号的缺省处理方式，同样info signals可以查看接受到的信号。要想在调试的程序中使用信号，我们需要使用gdb中handle这个命令，具体用法如这个形式   ：`handle  signal keywords`。keywords的取值如下：
 
|keywords 	|说明 	|keywords 	|说明
-------------------|------------|------------------|-----
stop 	|当GDB收到signal，停止被调试程序的执行 	|nostop| 	当GDB收到指定的信号，不会应用停止程序的执行，只会打印出一条收到信号的消息
print 	|如果收到signal，打印出一条信息 	|noprint 	|不会打印信息
pass 	|如果收到signal，把该信号通知给被调试程序 	|nopass 	|不会告知被调试程序收到signal
ignore |	同nopass 	|noignore |	同pass

今天就是因为一个类似这样的代码，浪费了很久的时间，在gdb调试阻塞在select处，这是后发送一个SIGINT信号，gdb有一行的输出说收到了一个信号但是next，没有反映，还是阻塞在select，我居然无知的以为这是select被信号中断后自动重启，试了好久不自动重启的方法都不行，根本原因原来是gdb阻塞了被调试程序的信号接收......，这里特别感谢“gdb信号”链接的作者
```
#include<sys/select.h>
#include<signal.h>
#include<stdio.h>
#include<string.h>
static void sig_int(int sig)
{
	printf("signo:%d\n",sig);
}
int main()
{
	struct sigaction sa;
	fd_set rset;

	memset(&sa,0,sizeof(sa));
	sa.sa_handler = sig_int;
	sigaction(SIGINT,&sa,0);

	
	select(1,&rset,0,0,0);
	return 0;

}

```
#### 调试多线程
+ `info threads `显示当前可调试的所有线程，每个线程会有一个GDB为其分配的ID，后面操作线程的时候会用到这个ID。 前面有*的是当前调试的线程。
+ `thread ID` 切换当前调试的线程为指定ID的线程。
+ `break thread_test.c:123 thread all` 在所有线程中相应的行上设置断点
+ `thread apply ID1 ID2 command` 让一个或者多个线程执行GDB命令command。 
+ `thread apply all command`让所有被调试线程执行GDB命令command。
+ `set scheduler-locking off|on|step` 估计是实际使用过多线程调试的人都可以发现，在使用step或者continue命令调试当前被调试线程的时候，其他线程也是同时执行的，怎么只让被调试程序执行呢？通过这个命令就可以实现这个需求。off 不锁定任何线程，也就是所有线程都执行，这是默认值。 on 只有当前被调试线程会执行。 没看懂step和on有什么区别


内容出自:
[gdb调试多线程](http://www.cnblogs.com/xuxm2007/archive/2011/04/01/2002162.html)
[gdb信号](http://www.cnblogs.com/ittinybird/p/4845394.html)
[gdb core文件的调试](http://www.cnblogs.com/hazir/p/linxu_core_dump.html)