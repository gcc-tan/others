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

#### 调试正在运行的程序

