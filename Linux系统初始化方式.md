### 问题
今天在使用ssh时候碰到了一个很有意思的问题。下面的输出表示ssh在正常运行没有问题。这个时候我想停止这个服务（不知道为什么`/etc/init.d/ssh start|stop`这个脚本不太好用），所以我就根据pid，kill掉这个进程，我以为万事大吉，准备收工。结果就尴尬了，首先的那个进程虽然是被结束了，但是被结束了之后又被立即创建了。这就让我很上火，然后在网上搜索了半天才弄出一个结果。**仔细看看就知道这个sshd进程的父进程pid=1,它是一个init进程的子进程，这个进程肯定是init进程给重新启动的，但是这个过程究竟是怎么回事呢？具体的东西就可以看接下来的upstart分析**
```
tan@tan:~$ ps -ef | grep ssh
root     21846 21657  0 14:51 pts/0    00:00:00 vi /etc/init/ssh.conf
root     23622     1  0 16:25 ?        00:00:00 /usr/sbin/sshd -D
tan      23647 23340  0 16:32 pts/11   00:00:00 grep --color=auto ssh
tan@tan:~$ sudo kill -s SIGTERM 23622
tan@tan:~$ ps -ef | grep ssh
root     21846 21657  0 14:51 pts/0    00:00:00 vi /etc/init/ssh.conf
root     23652     1  0 16:32 ?        00:00:00 /usr/sbin/sshd -D
tan      23655 23340  0 16:32 pts/11   00:00:00 grep --color=auto ssh
```


现在Linux系统init过程中主要有三种技术

+ sysvinit(快要淘汰了)
+ upstart(接下来需要淘汰的)
+ systemd(新技术)

### sysvinit
Linux 操作系统的启动首先从 BIOS 开始，接下来进入 boot loader，由 bootloader 载入内核，进行内核初始化。内核初始化的最后一步就是启动 pid 为 1 的 init 进程。这个进程是系统的第一个进程。它负责产生其他所有用户进程。init 以守护进程方式存在，是所有其他进程的祖先。init 进程非常独特，能够完成其他进程无法完成的任务。Init 系统能够定义、管理和控制 init 进程的行为。它负责组织和运行许多独立的或相关的始化工作(因此被称为 init 系统)，从而让计算机系统进入某种用户预订的运行模式。

sysvinit巧妙地运用脚本，文件命名规则和软链接来实现不同的runlevel。首先sysvinit需要读取/etc/inittab文件的内容，它获取以下的配置信息：

+ 系统需要进入的 runlevel
+ 捕获组合键的定义
+ 定义电源 fail/restore 脚本
+ 启动 getty 和虚拟控制台

得到这些信息之后，sysvinit顺序地执行以下的步骤，从而将系统初始化为runlevel X：

1. /etc/rc.d/rc.sysinit
2. /etc/rc.d/rc 和/etc/rc.d/rcX.d/ (X 代表运行级别 0-6)
3. /etc/rc.d/rc.local
4. X Display Manager（如果需要的话）

### upstart
**在现在的ubuntu系统中已经找不到/etc/inittab文件了，ubuntu使用了一种upstart的init系统**

upstart采用事件驱动模型，upstart可以：

+ 更快地启动系统
+ 当硬件被发现或者拔出时候动态地启动和停止服务

这些特点使得upstart可以更好地应用在桌面或者是便携的系统中。

在upstart中主要概念是job和event。job是一个工作单元，用来完成一件工作，例如启动一个后台服务，或者运行一个配置命令。每个 Job 都等待一个或多个事件，一旦事件发生，upstart 就触发该 job 完成相应的工作。

####job

Job 就是一个工作的单元，一个任务或者一个服务。可以理解为 sysvinit 中的一个服务脚本。有三种类型的工作：
+ task job；
+ service job；
+ abstract job；
task job 代表在一定时间内会执行完毕的任务，比如删除一个文件；
service job 代表后台服务进程，比如 apache httpd。这里进程一般不会退出，一旦开始运行就成为一个后台精灵进程，由 init 进程管理，如果这类进程退出，由 init 进程重新启动，它们只能由 init 进程发送信号停止。它们的停止一般也是由于所依赖的停止事件而触发的，不过 upstart 也提供命令行工具(ubuntu可以使用`sudo service servicename start|stop|restart`，servicename可以通过`service --status-all`这个命令来查询。但是我还是没搞清楚这个和`initctl`命令的区别)，让管理人员手动停止某个服务（这里就是为什sshd这个进程杀死之后又会被init进程启动的原因）；

####job生命周期

Upstart 为每个工作都维护一个生命周期。一般来说，工作有开始，运行和结束这几种状态。为了更精细地描述工作的变化，Upstart 还引入了一些其它的状态。比如开始就有开始之前(pre-start)，即将开始(starting)和已经开始了(started)几种不同的状态，这样可以更加精确地描述工作的当前状态。
工作从某种初始状态开始，逐渐变化，或许要经历其它几种不同的状态，最终进入另外一种状态，形成一个状态机。在这个过程中，当工作的状态即将发生变化的时候，init 进程会发出相应的事件（event）。


####事件event

顾名思义，Event 就是一个事件。事件在 upstart 中以通知消息的形式具体存在。一旦某个事件发生了，Upstart 就向整个系统发送一个消息。没有任何手段阻止事件消息被 upstart 的其它部分知晓，也就是说，事件一旦发生，整个 upstart 系统中所有工作和其它的事件都会得到通知。
Event 可以分为三类: signal，methods 或者 hooks

+ Signals
Signal 事件是非阻塞的，异步的。发送一个信号之后控制权立即返回。
+ Methods
Methods 事件是阻塞的，同步的。
+ Hooks
Hooks 事件是阻塞的，同步的。它介于 Signals 和 Methods 之间，调用发出 Hooks 事件的进程必须等待事件完成才可以得到控制权，但不检查事件是否成功。

####job和event合作

Upstart 就是由事件触发工作运行的一个系统，每一个程序的运行都由其依赖的事件发生而触发的。

系统初始化的过程是在工作和事件的相互协作下完成的，可以大致描述如下：系统初始化时，init 进程开始运行，init 进程自身会发出不同的事件，这些最初的事件会触发一些工作运行。每个工作运行过程中会释放不同的事件，这些事件又将触发新的工作运行。如此反复，直到整个系统正常运行起来。

究竟哪些事件会触发某个工作的运行？这是由工作配置文件定义的。

####工作配置文件

任何一个工作都是由一个工作配置文件（Job Configuration File）定义的。这个文件是一个文本文件，包含一个或者多个小节（stanza）。每个小节是一个完整的定义模块，定义了工作的一个方面，比如 author 小节定义了工作的作者。工作配置文件存放在/etc/init 下面，是以.conf 作为文件后缀的文件。ubuntu系统的工作配置文件目录是`/etc/init/xxx.conf`，下面是一个ssh服务的配置文件的例子
```
# ssh - OpenBSD Secure Shell server
#
# The OpenSSH server provides secure shell access to the system.

description	"OpenSSH server"

start on runlevel [2345]
stop on runlevel [!2345]

#可以注释下面这两行，重启服务之后，再kill sshd进程就发现init进程不会重启sshd了
respawn
respawn limit 10 5
umask 022

env SSH_SIGSTOP=1
expect stop

# 'sshd -D' leaks stderr and confuses things in conjunction with 'console log'
console none

pre-start script
    test -x /usr/sbin/sshd || { stop; exit 0; }
    test -e /etc/ssh/sshd_not_to_be_run && { stop; exit 0; }

    mkdir -p -m0755 /var/run/sshd
end script

# if you used to set SSHD_OPTS in /etc/default/ssh, you can change the
# 'exec' line here instead
exec /usr/sbin/sshd -D

```

+ `start on EVENT [[KEY=]VALUE]... [and|or...] `可以在 start on 中指定多个事件，表示该工作的开始需要依赖多个事件发生。多个事件之间可以用 and 或者 or 组合，"表示全部都必须发生"或者"其中之一发生即可"等不同的依赖条件。
+ `stop on`和上面的类似
+ `exec` 表示配置工作需要运行的命令
+ `script` 定义了需要运行的脚本



**内容出处**
[浅析Linux的初始化init系统，第一部分：sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)

[浅析Linux的初始化init系统，第二部分：upstart](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/index.html)

[浅析Linux的初始化init系统，第三部分：systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)

[How To Configure a Linux Service to Start Automatically After a Crash or Reboot – Part 1: Practical Examples](https://www.digitalocean.com/community/tutorials/how-to-configure-a-linux-service-to-start-automatically-after-a-crash-or-reboot-part-1-practical-examples)













