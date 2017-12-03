性能分析的命令有很多，perf，gprof等。
###GNU gprof命令
gprof的优点就是使用简单。

缺点是：
1. 使用受限：gprof不能测量在内核模式的时间(系统调用，等待CPU和I/O时间)。因此我认为gprof只适合分析CPU密集型的程序。
2. gprof统计的时间是通过采样获取的，受制于统计的不准确性。如果一个函数执行很小的一段时间，那么平均情况下应该对这个函数采样一次，很可能没有采样或者是采样两次。这个采用的误差是可以估计的，如果采样的时长是n个周期，误差的上界是$\sqrt{n}$


使用方法
1. 使用gcc编译源代码，加上编译参数-pg。`gcc a.c -pg`
2. 执行目标代码，会生成一个用于gprof分析的文件--gmon.out。`./a.out`
3. 使用gprof命令分析二进制文件和gprof生成的文件。`gprof -b a.out gmon.out`



下面举一个例子。测试的文件名字是8.c，gcc生成的二进制文件是8，执行8之后生成的是gmon.out。下面是8.c的内容：
```
#include<stdio.h>
#define TARGET 100000000
void foo1()
{
	int i;
	for (i = 0; i < TARGET; ++i)
		;
}
void foo2()
{
	int i;
	for (i = 0; i < TARGET; i += 2)
		;
}
void foo3()
{
	int i;
	for (i = 0; i < TARGET; i += 3)
		;
	foo2();
}
int main()
{
	foo1();
	foo2();
	foo3();
	return 0;
}
```

从上面的代码来看，foo1()执行一次，foo2()执行两次。foo1()执行一次的时间是foo2()的两倍，是foo3()执行一次时间的三倍。下面是gprof的输出
```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 46.28      0.16     0.16        2    80.99    80.99  foo2
 34.71      0.28     0.12        1   121.48   121.48  foo1
 14.46      0.33     0.05        1    50.62   131.60  foo3


			Call graph


granularity: each sample hit covers 2 byte(s) for 2.99% of 0.33 seconds

index % time    self  children    called     name
                                                 <spontaneous>
[1]    100.0    0.00    0.33                 main [1]
                0.05    0.08       1/1           foo3 [3]
                0.12    0.00       1/1           foo1 [4]
                0.08    0.00       1/2           foo2 [2]
-----------------------------------------------
                0.08    0.00       1/2           foo3 [3]
                0.08    0.00       1/2           main [1]
[2]     48.5    0.16    0.00       2         foo2 [2]
-----------------------------------------------
                0.05    0.08       1/1           main [1]
[3]     39.4    0.05    0.08       1         foo3 [3]
                0.08    0.00       1/2           foo2 [2]
-----------------------------------------------
                0.12    0.00       1/1           main [1]
[4]     36.4    0.12    0.00       1         foo1 [4]
-----------------------------------------------


Index by function name

   [4] foo1                    [2] foo2                    [3] foo3

```

**gprof输出有两大类，一个是flat profile，另外一个是call graph**。

flat profile输出字段：

+ % time
程序执行函数的时间占总的执行时间的比例，这一列加起来等于100%

+ cumulative seconds
是一个累计的时间，表示用于执行这个函数的秒数，加上上面所有函数的执行秒数

+ self seconds
这个函数单独执行的秒数(不包括调用其他函数的时间)。flat profile列表就是首先按照这个数字排序的。

+ calls
函数被调用的次数。如果函数从来没有被调用，或者是被调用的次数不能确定(可能函数在编译时没有打开profiling标志)，这两种情况调用次数为空

+ self ms/call
表示这个函数平均每次调用花费的毫秒(不包括调用其他函数)

+ total ms/call
平均每次调用花费在这个函数(包括这个函数调用其他函数)的毫秒数。

+ name
函数的名字。如果flat profile列表中self seconds和calls字段相同就按照那么字段排序。

call graph：

call graph中用短横线分隔的就是一个entry。一个entry中有三种类型的行：
+ **primary line**
一个entry中index列不为空的那行就是这个entry中的primary line。self字段是函数自己的执行时间(和flat profile中的self seconds相等)，children字段是这个函数调用其他函数执行的时间。called表示这个函数被调用的次数，如果函数有递归调用那么就会有两个数，之间用’+‘分隔，第一个参数是没有递归的调用次数，另外一个是递归的调用次数。
+ **Line for Function's Callers**
在一个entry中，primary line前面的行表示primary line代表函数的调用者。这些字段的意义和上面的primary line中有一些不同。self表示调用者调用primary line行的函数这个函数花费在自身的时间。children表示调用者调用primary line函数这个函数花费在子函数上的时间。called由两个数字组成，斜线前的数字是调用调用primary line函数的次数，后面是这个函数所有的非递归调用次数。
+ **Lines for a Function’s Subroutines**
在一个entry中，primary line后面的行表示primary line代表函数的子函数(被该函数调用的函数)，self字段是primary line函数调用子函数时，子函数自己花费的时间。children表示这个子函数调用其他函数花费的时间。两者之和表示primary line函数调用该子函数花费的总时间。called由两个数字构成，第一个是primary line函数调用该函数的次数，第二个是这个函数被调用的总次数。

下面还有一段是gprof的实现原理，懒得翻译了，有时间再看看：
```
Profiling works by changing how every function in your program is compiled so that when it is called, it will stash away some information about where it was called from. From this, the profiler can figure out what function called it, and can count how many times it was called. This change is made by the compiler when your program is compiled with the ‘-pg’ option, which causes every function to call mcount (or _mcount, or __mcount, depending on the OS and compiler) as one of its first operations.

The mcount routine, included in the profiling library, is responsible for recording in an in-memory call graph table both its parent routine (the child) and its parent’s parent. This is typically done by examining the stack frame to find both the address of the child, and the return address in the original parent. Since this is a very machine-dependent operation, mcount itself is typically a short assembly-language stub routine that extracts the required information, and then calls __mcount_internal (a normal C function) with two arguments—frompc and selfpc. __mcount_internal is responsible for maintaining the in-memory call graph, which records frompc, selfpc, and the number of times each of these call arcs was traversed.

GCC Version 2 provides a magical function (__builtin_return_address), which allows a generic mcount function to extract the required information from the stack frame. However, on some architectures, most notably the SPARC, using this builtin can be very computationally expensive, and an assembly language version of mcount is used for performance reasons.

Number-of-calls information for library routines is collected by using a special version of the C library. The programs in it are the same as in the usual C library, but they were compiled with ‘-pg’. If you link your program with ‘gcc … -pg’, it automatically uses the profiling version of the library.

Profiling also involves watching your program as it runs, and keeping a histogram of where the program counter happens to be every now and then. Typically the program counter is looked at around 100 times per second of run time, but the exact frequency may vary from system to system.

This is done is one of two ways. Most UNIX-like operating systems provide a profil() system call, which registers a memory array with the kernel, along with a scale factor that determines how the program’s address space maps into the array. Typical scaling values cause every 2 to 8 bytes of address space to map into a single array slot. On every tick of the system clock (assuming the profiled program is running), the value of the program counter is examined and the corresponding slot in the memory array is incremented. Since this is done in the kernel, which had to interrupt the process anyway to handle the clock interrupt, very little additional system overhead is required.

However, some operating systems, most notably Linux 2.0 (and earlier), do not provide a profil() system call. On such a system, arrangements are made for the kernel to periodically deliver a signal to the process (typically via setitimer()), which then performs the same operation of examining the program counter and incrementing a slot in the memory array. Since this method requires a signal to be delivered to user space every time a sample is taken, it uses considerably more overhead than kernel-based profiling. Also, due to the added delay required to deliver the signal, this method is less accurate as well.

A special startup routine allocates memory for the histogram and either calls profil() or sets up a clock signal handler. This routine (monstartup) can be invoked in several ways. On Linux systems, a special profiling startup file gcrt0.o, which invokes monstartup before main, is used instead of the default crt0.o. Use of this special startup file is one of the effects of using ‘gcc … -pg’ to link. On SPARC systems, no special startup files are used. Rather, the mcount routine, when it is invoked for the first time (typically when main is called), calls monstartup.

If the compiler’s ‘-a’ option was used, basic-block counting is also enabled. Each object file is then compiled with a static array of counts, initially zero. In the executable code, every time a new basic-block begins (i.e., when an if statement appears), an extra instruction is inserted to increment the corresponding count in the array. At compile time, a paired array was constructed that recorded the starting address of each basic-block. Taken together, the two arrays record the starting address of every basic-block, along with the number of times it was executed.

The profiling library also includes a function (mcleanup) which is typically registered using atexit() to be called as the program exits, and is responsible for writing the file gmon.out. Profiling is turned off, various headers are output, and the histogram is written, followed by the call-graph arcs and the basic-block counts.

The output from gprof gives no indication of parts of your program that are limited by I/O or swapping bandwidth. This is because samples of the program counter are taken at fixed intervals of the program’s run time. Therefore, the time measurements in gprof output say nothing about time that your program was not running. For example, a part of the program that creates so much data that it cannot all fit in physical memory at once may run very slowly due to thrashing, but gprof will say it uses little time. On the other hand, sampling by run time has the advantage that the amount of load due to other users won’t directly affect the output you get.
```
**内容来自**

[Interpreting gprof’s Output](https://sourceware.org/binutils/docs/gprof/Output.html#Output)