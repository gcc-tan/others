### diff
`diff [option] files` 
常用参数

    -c 上下文模式
    -u 合并模式

+ diff输出格式(默认) 
n1 a n3,n4 表示在文件1的n1行后面添加n3到n4行 
n1,n2 d n3 表示在文件1 n1到n2行之间删除 变成文件2的n3行 
n1,n2 c n3,n4 表示把n1,n2行用n3,n4行替换掉 
字母a：表示附加（add） 
字符c：表示修改（change） 
字符d：表示删除（delete） 
字母前的是源文件，字母后是目标文件。Nx表示行号 
以"<"打头的行属于第一个文件，以”>”打头的行属于第二个文件

+ 上下文模式
这种方式在开头两行作了比较文件的说明，这里有三中特殊字符： 
“+” 比较的文件的后者比前着多一行 
“-” 比较的文件的后者比前着少一行 
“!” 比较的文件两者有差别的行 
			
		
			tan@ttt:~/Desktop$ cat a
			h
			b
			c
			d
			tan@ttt:~/Desktop$ cat b
			a
			b
			tan@ttt:~/Desktop$ diff a b
			1c1
			< h
			---
			> a
			3,4d2
			< c
			< d
			
			tan@ttt:~/Desktop$ diff -c a b
			*** a	2017-03-07 16:05:52.312919199 +0800
			--- b	2017-03-07 16:07:08.248921611 +0800
			***************
			*** 1,4 ****
			! h
			  b
			- c
			- d
			--- 1,2 ----
			! a
			  b
			  这里的1，4和1，2就是文件的指定范围发生了改变

+ 合并模式

		tan@ttt:~/Desktop$ diff -u a b 
		--- a	2017-03-07 16:05:52.312919199 +0800
		+++ b	2017-03-07 16:07:08.248921611 +0800
		@@ -1,4 +1,2 @@
		-h
		+a
		 b
		-c
		-d
		这里的1，4和1，2都是从第1行开始4行和连续2行的意思，和上下文模式不同

###bc命令
bc是支持任意精度的数字和交互执行语句的语言。语法和C语言类似。bc按照文件在命令行被列出的顺序执行，在所有的文件被读取完成之后，bc从标准输入读入。所有的语句都在它被读入时执行。（如果在文件中包含一个命令暂停了解释器，bc就不会从标准输入读取了）

**语法**

```
bc [ -hlwsqv ] [long-options] [  file ... ]
```

>file是用bc语言定义的语句

**选项**

+ -i。强制使用交互模式
+ -l。定义标注的数学库
+ -s。只处理POSIX标准定义的bc语言
+ -q。不打印GNU的bc欢迎语句

在bc中，有几个很重要的概念。

**数（Numbers）**
数是bc里面最基本的元素。这里面数是任意精度的。这里的精度包括整数部分和小数部分。所有的数都是以10进制表示并且是10进制计算的。（这个版本在除法和乘法操作过程中产生截断误差）。数有两个属性，length和scale。length是一个数中所有的有意义的数字的位数。scale是小数点后数字的位数。例如：.000001的length是6,scale是6。1935.000的length是7，scale是3。

**变量（Variables）**
数字存储在两种类型的变量中，简单变量和数组。变量的命名规则是以字母开头，后面接任何的字母，数字，下划线。所有的字母都应该是小写的（在POSIX标准的bc中，变量名字是一个小写字母）。通过变量名字后边有中括号（`[]`）能够判断是数组变量。

除了自定义的变量外，bc中定义了4个特殊的变量。

+ scale。定义某些操作中怎么样使用小数点后的数字。合法值是0～c语言定义的最大的int
+ ibase。定义输入中数的进制。默认是10，合法值是2～16.
+ obase。定义输出中数的进制。默认是10
+ last。（扩展，POSIX标准没有）。存储最后打印输出的变量的值。

**注释（comments）**
使用类似于C语言的形式的多行注释：`/* comment content */`

单行注释使用井号`#`开头，这是一个扩展，POSIX标准不支持。

>以后说扩展，就是说POSIX标准的bc是不支持的。

**表达式（Experssions）**
数在表达式和语句中被使用。bc语言是解释执行的，没有main函数。

最简单的表达式就是一个常数。bc将输入常数q使用ibase定义的进制转换成内部使用的10进制。输入的数字包括0-9和A-F。（A-F必须大写，小写是变量名）。 Single digit numbers always have the value of the digit regardless of the value of ibase. (i.e. A = 10.)   For multi-digit numbers, bc changes all input digits greater or equal to ibase to the value of ibase-1.  This makes the number FFF always be the largest 3  digit  number  of  the input base.

复杂的表达式和其他高级语言的组成类似。下面是一个表达式的表格和每个表达式对应的含义。expr表示一个表达式。var表示一个简单变量（name）或者数组变量（name[expr]）。每个表达式都有scale，这个scale可能是数的scale中继承的，或者是运算中获得的，或者是scale变量指示的。除非特别说明，整个表达式的scale是表达式中最大的sacle。

> 这里举例子说明scale的问题，在bc中，假设scale变量为默认值0。那么1.5 + 2.222的值为3.722。很好理解，取常数1.5与2.222的scale中最大的scale。而表达式1.5 / 2.222为0。这是因为特别指定了除法运算的scale是scale变量定义的值0，那么所有小数部分全都被抹去，结果为0。因此为了避免这种问题，可以将scale设定成一个满意的精度

| 表达式 | 含义 |
| ------ |---- |
|- expr | 相反数 |
| ++ var | 前置自增1，表达式的值就是自增之后的值 |
| -\- var | 前置自减1，表达式的值是自减之后的值 |
| var ++ | 后置自增1，表达式的值是var的值，var自增 |
| var -\- | 后置自减1，表达式的值是var的值，var自减 |
| expr + expr | 加 |
| expr - expr | 减 |
| expr * expr | 乘 |
| expr / expr | 除， scale为scale变量定义的值 |
| expr % expr | 取余，如果scale为0，并且两个表达式就是整数，那么和普通的取余运算一样。a % b的计算过程为，首先计算a / b使用scale变量指定的位数进行截断，然后利用这个结果计算a - (a / b) * b，这个就是a % b的结果，要进行截断，截断精度是max(scale + scale(b), scale(a))。其中(a / b)就是第一步计算的值 |
| expr ^ expr | 指数，要求第二个expr是整数。如果expr是负数，那么表达式的scale就是变量scale定义的值，否则是scale(a^b) = min(scale(a)*b, max( scale, scale(a))) |
| (expr) | 括号表达式，先计算expr |
| var = expr | 赋值表达式 |
| var <op>= expr | 等价于 var = var <op> expr |

除了上面的表达式外还有关系运算表达式，他们的值不是0就是1，0代表false，1代表true。这些表达式可能出现在前面的表达式中（POSIX标准要求只能出现在if，while，for等语句中）

```
#关系表达式
expr1 < expr2
expr1 <= expr2
expr1 > expr2
expr1 >= expr2
expr1 == expr2
expr1 != expr2
#bool运算，值为0，1。POSIX标注不支持
!expr
expr && expr
expr || expr
```

运算符的优先级可以看man文档，和C语言的基本一样，但是有一个要特别注意的地方就是赋值运算的优先级提高了。导致`a = 3 < 5`的表达式不是先比较3和5,而是先赋值。

bc还提供了能够直接调用的函数。包括length(expression)，read()，scale(expression)，sqrt(expression)等。


**语句（statements）**
语句包括方括号的内容是可选的：

+ expression。如果表达式是以变量后接着等号开始，那么表达式被认为是赋值语句。如果不是赋值语句，那么表达式的值将会被计算输出。e.g. a = 1是赋值语句不输出任何结果，(a = 1)不是赋值语句输出表达式的值1。

+ string。用双引号引用，直接原样输出。输出时字符串后面不加回车。
+ print list。print是一个扩展，提供了另外一种输出的方式。list是字符串和表达式构成的，他们之间使用逗号分割。字符串可以使用某些转义字符。e.g. print 2.2 * 3, "\n"
+ { statement_list }。语句块。
+ if ( expression ) statement1 [else statement2]。else是扩展。
+ while ( expression ) statement。
+ for ( [expression1] ; [expression2] ; [expression3] ) statement。for里面可选是扩展，POSIX要求三个必须填上
+ break。
+ continue。扩展。
+ halt。停止执行，退出bc。扩展。
+ return [ (expression) ]。从函数返回表达式的值。没有方括号里面的是返回0。也可以不使用()，但是属于扩展，POSIX标准要求。

**pseudo statements**

主要有limits（扩展，输出bc版本限制），quit（无条件退出），warranty（开源权限信息）

**函数（functions）**
定义函数的方法：
```
define name ( parameters ) { newline
                  auto_list   statement_list }
```

name是函数的名字，parameters是变量名，可以定义数组变量和普通变量，使用逗号分隔。auto_list是一个可选的自定义变量列表，使用`auto name, ... ;`的形式定义。statement_list是上面出现的所有合法的语句。关于参数和auto变量需要注意的是bc中函数定义的参数和auto变量在其他函数中是可见的，例如函数A调用函数B，在函数中定义了auto a，那么在B中能够访问a的值。如果B中重新定义了a，那么对a的操作就是B中的a，原来的a被压栈。然后当函数B调用返回时，a弹栈，此时a的值又是A函数定义的auto a了。

函数的返回值由return语句确定，默认的返回值是0。

在函数中的所有常数会根据函数调用时的ibase值进行转换，转换成内部使用的10进制。

函数定义的格式在POSIX标准中左括号必须和define关键字在一行，但是这个版本没有这个要求。除此之外，函数可以定义为void类型，只要在define关键字后面加上void关键字，表示函数没有返回值，意味着函数做外单独一行调用时不会产生输出。

**数学库（math library）**
使用-l参数能够加载bc的函数库，函数库默认的scale是20。数学函数会根据他们被调用时的scale变量的值设置他们的结果。函数库的函数有：

+ s(x)。sin(x)
+ c(x)。cos(x)
+ a(x)。arctan(x)
+ l(x)。ln(x)
+ e(x)。e^x
+ j(n, x)。Bessel function。
