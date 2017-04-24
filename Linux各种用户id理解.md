### Linux各种用户ID的理解
+ 实际用户ID(RUID):用来标志一个系统中用户的身份，实际用户ID是在用户登录的时候由login程序来设置的，一般不会改变
+ 有效用户ID(EUID):用来决定用户对系统资源的访问权限。系统判断用户有没有权限都是通过有效用户ID来判断的。登录之后有效用户ID是和实际用户ID是相同的，如果要进行某些特权操作，就必须改变有效用户ID。
+ SUID:用于对外的权限开放，这个是文件的一个标志位（只是对可执行文件件才有意义），如果设置了该位，执行该文件的时候，EUID就会被设置成文件所有者的id
+ SGID:用于对外的权限开放，同上，如果设置了该位，执行该文件的时候EGID，被设置成文件所有者所在组的权限

为什么要有这两个位呢，有个非常经典的例子。大叫都知道passwd命令来更改帐号的密码，而密码要更改必须修改/etc/shadow文件，很不巧，这个文件肯定只有root帐号才有权限修改。这个就很难受了，总不能一个普通帐号改自己的密码，还得找root帐号去执行吧。所以SUID的作用就体现了。当其他用户执行这个程序的时候，EUID就会变成文件所有者的uid，然后passwd的文件所有者肯定是root，然后该程序就会获得权限修改这个只有root帐号能够修改的文件了

给文件加SUID和SUID的命令如下：
```
chmod u+s filename 设置SUID位
chmod u-s filename 去掉SUID设置
chmod g+s filename 设置SGID位
chmod g-s filename 去掉SGID设置 
```
其实也可以用8进制数字表示，在unix实现中，文件权限用12个二进制位表示
12 11 10 9 8 7 6 5 4 3 2 1
12是SUID，11是SGID，10是sticky位，9～1就是对应的文件所有者，同组，其他用户的读写执行的权限位了
这是上面设置之后的一个例子
```
tan@ttt:~/Documents/code/apue/8$ ll data
-rw-rw-r-- 1 tan tan 0 Apr  3 11:11 data
tan@ttt:~/Documents/code/apue/8$ sudo chmod 4664 data
[sudo] password for tan: 
tan@ttt:~/Documents/code/apue/8$ ll data
-rwSrw-r-- 1 tan tan 0 Apr  3 11:11 data
```
大写的S表示只设置了SUID(或者SGID)位，没有设置可执行权限，小写的s表示两者都设置了

这里给出简单的测试代码，可以编译出可执行文件，然后用chown改变文件的所有者去执行该文件，看看输出的id有没有什么改变
```
#include<stdio.h>
#include<unistd.h>
int main(int argc,char **argv)
{
	printf("euid:%d egid:%d\n",geteuid(),getegid());
	return 0;
}
```
