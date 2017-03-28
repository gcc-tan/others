## Linux网络命令

###netstat
netstat用于显示网络中网络连接，接口状态，路由表等信息

参数：

+ a (all)显示所有选项，默认不显示LISTEN相关
+ -t (tcp)仅显示tcp相关选项
+ -u (udp)仅显示udp相关选项
+ -n 拒绝显示别名，能显示数字的全部转化成数字。
+ -l 仅列出有在 Listen (监听) 的服務状态

+ -p 显示建立相关链接的程序名和进程号
+ -r 显示路由信息，路由表
+ -e 显示扩展信息，例如uid等
+ -s 按各个协议进行统计
+ -c 每隔一个固定时间，执行该netstat命令。

提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到


netstat的输出

	Active Internet connections (w/o servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State
	tcp        0      1 100.66.3.146:58506      chatenabled.mail.:https SYN_SENT
	tcp        0      0 100.66.3.146:37654      114.55.49.182:http      ESTABLISHED
	tcp        0      0 100.66.3.146:45072      121.194.7.171:http      TIME_WAIT
	tcp        0      0 100.66.3.146:57338      47.89.29.3:http         ESTABLISHED
	tcp        0      1 100.66.3.146:58508      chatenabled.mail.:https SYN_SENT
	Active UNIX domain sockets (w/o servers)
	Proto RefCnt Flags       Type       State         I-Node   Path
	unix  19     [ ]         DGRAM                    8718     /dev/log
	unix  3      [ ]         STREAM     CONNECTED     14973
	unix  3      [ ]         STREAM     CONNECTED     15414
	unix  3      [ ]         STREAM     CONNECTED     71880
	unix  3      [ ]         STREAM     CONNECTED     13798    /var/run/dbus/system_bus_socket

从整体上看，netstat的输出结果可以分为两个部分：

一个是Active Internet connections，称为有源TCP连接，其中"Recv-Q"和"Send-Q"指%0A的是接收队列和发送队列。这些数字一般都应该是0。如果不是则表示软件包正在队列中堆积。这种情况只能在非常少的情况见到。

另一个是Active UNIX domain sockets，称为有源Unix域套接口(和网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。
Proto显示连接使用的协议,RefCnt表示连接到本套接口上的进程号,Types显示套接口的类型,State显示套接口当前的状态,Path表示连接到套接口的其它进程使用的路径名。

###重启网络
ubuntu的环境下sudo service network-manager restart，因为ubuntu使用了network-manager代替了传统的linux的网络模型，或者用ifup/down

###route
