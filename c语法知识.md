### 函数指针
下面是函数指针及其使用的例子

	#include<stdio.h>
    void show(int param)
    {
    	printf("show:%d\n",param);
    }
    int main(int argc,char **argv)
    {
    	void (* fp1)(int);
        typedef void FP2(int);
        typedef void (* FP3)(int);
        fp1 = show;
        fp1(10);
        fp1 = &show;
        (* fp1)(20);
        
        FP2 *fp2 = show;
        fp2(30);
        fp2 = &show;
        (* fp2)(40);
        
        FP3 fp3 = show;
        fp3(50);
        fp3 = &show;
        (* fp3)(60);
        
        return 0;
    }
    
### static和extern
例如在a.c

	#include<stdio.h>
	int t = 10;
在b.c中
	
	#include<stdio.h>
	int main(int argc,char **argv)
	{
	    extern int t;
	    printf("%d\n",t);
	    return 0;
	}

result：10
在a.c中定义了一个全局变量t，然后在b.c中用extern声明了这个变量，并且引用了这变量 果是在main函数中进行声明的，则只能在main函数中调用，在其它函数中不能调用。其实要调用其它文件中的函数和变量，只需把该文件用#include包含进来即可


static的可以修饰变量，函数。static修饰变量和函数时表示这个变量和函数是该文件可见的，就是出了文件其他文件是访问不到的
### _()函数
_是一个gettext库中的函数用来国际化的。这个函数的作用是根据系统语言来替换给定的字符串

	fprintf(stderr, _("Usage: route [-nNvee] [-FC] [<AF>]           List kernel routing tables\n"));


### 预编译指令
预处理是编译器进行编译的第一步工作，主要是扫描源代码和对程序进行初步的转换
然后常用的指令这里介绍一下

**#include**
作用是包含源文件

	#include<xxx>
	#include"xxx" 

两者的区别就是第一个只会在系统默认的路径和括号指定的路径内寻找，第二个是在源文件的目录，没找到就去系统目录找

**#define**
定义宏

	#define PI 3.14 
	#define MAX(a,b) (((a) > (b)) ? (a) : (b)) //这里的参数之所以要加上括号是因为宏的展开是简单的替换，如果a，b里面含有其他操作符就会因为优先级等其他原因而引起意想不到的结果
	#define APPEND(x) "abc"#x 
	#define JOIN(a,b) a##b //#和##是特殊的宏，连接字符串用的
**#if #elif #else #endif**
这个就是条件编译指令，用来有条件的选择性编译,注意条件的选项必须是常量或者是已经定义过的标志符

	#define OPTION 2
	#if OPTION == 1
	printf("option is 1\n");
	#elif OPTION == 2
	printf("option is 2\n");
	#else
	printf("other option\n");
	#endif

**#ifdef #ifndef #endif**
这几个也是条件编译的语句，常常用来做头文件的guard word。为什么要这个guard word呢，是因为为了防止头文件被重复引用，头文件被重复引用指的是同一个.c文件中两次include同一个头文件,如果两次引用了那么就会引起变量或者函数多次定义的语法错误，注意是定义不是生命，多次声明是没有语法错误的。虽然不太推荐在头文件中定义函数和变量的
例如有a.h,b.h,c.h三个文件

	a.h
	int a = 10;
	
	b.h
	#include"a.h"
	int b = 10;
	
	c.c
	#include"a.h"
	#include"b.h"
就是出现redefinition of a的错误了

用底下条件编译语句就能解决问题了
    #ifndef A_H
      
    #define A_H
    
      
    #endif

### 头文件的作用
先总结第一点c语言中函数假如写了一个通用的add函数，其他文件想要调用这个函数有两种方式，在调用前声明，这样假如调用比较多，然就要维护很多生命，然后如果把声明写入头文件中，只要用#include一次就够了，避免函数出现改动的时候要改动很多地方