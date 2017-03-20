## Makefile 文件编写
makefile文件是make工具的输入文件，其中定义了一些文件依赖，编译参数等信息，可以让工程完成自动编译的效果，提高开发效率

### makefile的基本规则

	target : prerequisites
		command
要求command前面要有一个tab，然后如果cmd太长可以用\换行

1.  target可以是一个或多个，可以是Object File，也可以是执行文件，甚至可以是一个标签。
2. prerequisites 就是生成目标所需要的文件或target
3. command 就是生成目标所需要执行的命令

除了这种简单的基本规则外，还有一种特殊的规则模式规则，就是在target中出现一个模式字符%，包含这个字符的目标用来匹配一个文件名，%可表示任何非空的字符串，规则的依赖文件中同样可以使用%它的取值由对应目标中的%来决定的

	

一个简单的例子，写得可能比较多
文件的内容

	tan@ttt:~/Desktop$ cat hello.h
	void print();
	tan@ttt:~/Desktop$ cat hello.c
	#include<stdio.h>
	void print()
	{
		printf("hello world!\n");
	}
	tan@ttt:~/Desktop$ cat main.c
	#include"hello.h"
	int main(int argc,char **argv)
	{
		print();
		return 0;
	}

makefile文件的内容

	project : main.o hello.o
		cc -o project main.o hello.o
	main.o : main.c hello.h
		cc -c main.c
	hello.o : hello.h hello.c
		cc -c hello.c 
	.PHONY : clean
	clean :
		rm project *o
其中cc（unix的一个c编译器）命令在ubuntu的环境下可以看到是

	tan@ttt:~/Desktop$ which cc
	/usr/bin/cc
	tan@ttt:~/Desktop$ ll /usr/bin/cc
	lrwxrwxrwx 1 root root 20 Mar  1 16:43 /usr/bin/cc -> /etc/alternatives/cc*
	tan@ttt:~/Desktop$ vi /etc/alternatives/cc
	tan@ttt:~/Desktop$ ll /etc/alternatives/cc
	lrwxrwxrwx 1 root root 12 Mar  1 16:43 /etc/alternatives/cc -> /usr/bin/gcc*

其实就是gcc

### make的主要工作流程
1. 读入主Makefile (主Makefile中可以引用其他Makefile)
2. 读入被include的其他Makefile
3. 初始化文件中的变量
4. 推导隐晦规则, 并分析所有规则
5. 为所有的目标文件创建依赖关系链
6. 根据依赖关系, 决定哪些目标要重新生成
7. 执行生成命令


### 注释
 注释是`#`
 
###  路径查找
VPATH VPATH的功能就是当源文件和makefile文件不在同一个目录中时就在VPATH指定的路径中查找
```
vpath <directories>            当前目录中找不到文件时, 就从<directories>中搜索
vpath <pattern> <directories>  符合<pattern>格式的文件, 就从<directories>中搜索
vpath <pattern>                 清除符合<pattern>格式的文件搜索路径
vpath                          清除所有已经设置好的文件路径
# 示例1 - 当前目录中找不到文件时, 按顺序从 src目录 ../parent-dir目录中查找文件
VPATH src:../parent-dir

# 示例2 - .h结尾的文件都从 ./header 目录中查找
VPATH %.h ./header

# 示例3 - 清除示例2中设置的规则
VPATH %.h

# 示例4 - 清除所有VPATH的设置
VPATH
```

### 定义变量
用 = 和 :=都能够定义变量，两者的区别是=可以使用后面定义的变量，:=只能使用之前定义的变量
eg

	tan@ttt:~/Desktop$ cat makefile 
	v1 = $(v2) kdkdk
	v2 = yyy
	all :
		@echo $(v1)
	tan@ttt:~/Desktop$ make
	yyy kdkdk

### 变量追加值
变量追加值直接用+=
eg

	tan@ttt:~/Desktop$ cat makefile
	v1 = kdkdk
	v1 += go
	all :
		@echo $(v1)
	tan@ttt:~/Desktop$ make
	kdkdk go

### makefile 命令的前缀
不用前缀  输出执行的命令以及命令执行的结果, 出错的话停止执行
前缀 @   只输出命令执行的结果, 出错的话停止执行
前缀 -   命令执行有错的话, 忽略错误, 继续执行

### 伪目标
伪目标不是真正的目标不会像真正的目标一样生成目标文件提升了make的效率，因为make不会试图去创建一个伪目标 具体的功能等到用过的时候在去补充

	.PHONY 指定的目标不管重不重名都会被执行
	.DEFAULT
	.PRECIOUS
	.INTERMEDIATE
	.SECONDARY
	.DELETE_ON_ERROR
	.IGNORE
	.LOW_RESOLUTION_TIME
	.SILENT
	.EXPORT_ALL_VARIABLES
	.NOTPARALLEL


### 引用其他的makefile
`include <filename>` filename中可以包含路径和通配符信息
eg

	# makefile文件的内容
	all:
	    @echo "主 Makefile begin"
	    @make other-all
	    @echo "主 Makefile end"

	include ./other/Makefile

	# ./other/makefile 内容
	other-all:
	    @echo "other makefile begin"
	    @echo "other makefile end"


### make的退出码

    0  表示成功执行
    1  表示make命令出现了错误
    2  使用了 "-q" 选项, 并且make使得一些目标不需要更新



### make指定目标
make的默认文件名字是makefile或者Makefile，然后要用其他名字要写成make -f filename 指定目标了就会执行指定的目标，如果没有就找到第一个目标执行

### 命令参数

	ARFLAGS 	AR命令的参数
	CFLAGS 	C语言编译器的参数
	CXXFLAGS 	C++语言编译器的参数
就是左边的参数假如被赋值，然后执行命令的时候就是带上被赋的值


### 自动变量
模式规则中，规则的目标和依赖文件名代表了一类文件名；规则的命令是对所有这一类文件重建过程的描述，显然，在命令中不能出现具体的文件名，否则模式规则失去意义
```
$@ 	表示规则的目标文件名

$% 	当规则的目标文件是一个静态库文件时，代表静态库的一个成员名。例如，规则的目标是“foo.a(bar.o)”，那么，“ $%”的值就为“bar.o”，“ $@ ”的值为“foo.a”。如果目标不是静态库文件，其值为空。

$< 	第一个依赖目标. 如果依赖目标是多个, 逐个表示依赖目标。如果是一个目标文件使用隐含规则来重建，则它代表由隐含规则加入的第一个依赖文件

$? 	比目标新的依赖文件的集合

$^ 	所有依赖文件的集合, 会去除重复的依赖文件

$+ 	所有依赖文件的集合, 不会去除重复的依赖文件

$* 	这个是GNU make特有的, 其它的make不一定支持
```
例子

	tan@ttt:~/Desktop$ cat makefile 
	all : hold your fire
		@echo "build target:$@ First depenency is $<"
	hold :
		@echo "hold"
	your : 
		@echo "your"
	fire :
		@echo "fire"
	tan@ttt:~/Desktop$ make
	hold
	your
	fire
	build target:all First depenency is hold


### export传递参数

    export value = xxx
    export value := xxx
    export value += xxx

这样的被调用的makefile文件中就能访问这个value值了




### 定义包

	define <command-name>
	command
	...
	endef

这样通过command-name就能够执行一串命令了
eg

	# Makefile 内容
	define run-hello-makefile
	@echo -n "Hello"
	@echo " Makefile!"
	@echo "这里可以执行多条 Shell 命令!"
	endef

	all:
	    $(run-hello-makefile)


	# bash 中运行make
	$ make
	Hello Makefile!
	这里可以执行多条 Shell 命令!


### 条件判断
条件判断的关键字主要有 ifeq ifneq ifdef ifndef

	# Makefile 内容
	all:
	ifeq ("aa", "bb")
	    @echo "equal"
	else
	    @echo "not equal"
	endif

	# bash 中执行 make
	$ make
	not equal

### 执行shell命令
采用$()的形式可以执行shell指令

	$(shell if [ "$$PWD" != "" ]; then echo $$PWD; else pwd; fi)




[本文转自](http://www.cnblogs.com/wang_yb/p/3990952.html) 



