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

###iptables
iptable是配置防火墙使用的。这个命令主要是由table，chain，rule构成：

+ rule是规则，是数据包满足什么条件，执行什么动作。
+ chain中包含一系列的规则，chain的本质其实是内核实现的5个钩子函数，表示数据包在内核中经过的所有流程，所有的数据包的流经这些chain的一个或者多个，分别作用在不同的位置。PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING。
+ table包括了一个或者多个chain。命令中定义的表有filter,nat,mangle,raw,security。table的作用其实是定义了一组功能的集合。它表示完成这个功能一般经由的chain，nat(PREROUTING,POSTROUTING,OUTPUT)。很明显表之间是有优先级的，不同的表包含了相同的chain的时候，优先级高的表的那个chain定义的规则先执行。

这里有个很好的图表示链之间的关系
![chain和table的对应图](img/tables_traverse.jpg)

这里给出几个简单的例子，具体的参数意思可以查，这里就不罗嗦了。
```
root@tan:~# iptables -traw  -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
-t是指定了table的名字，-nL n参数是为了防止长时间的方向dns查找，L是显示指定的链的信息，没有就是默认全部。target就是满足规则后采取的动作这个target有很多常见的有ACCEPT,DROP,DNAT,SNAT,MASQUERADE,REJECT,RETURN，policy就是默认的动作，这个是修改某个表中某个链的默认动组
```
root@tan:~# iptables -F//清除当前表中所有chain中的规则
root@tan:~# iptables -X//清楚当前表中用户自定义的链
```


**DNAT,SNAT,MASQUERADE三种target**
dnat是改写数据包的目的地址。应用的场景是：你的Web服务器在LAN内部，而且没有可在Internet上使用的真实IP地址，那就可以使用这个dnat让防火墙把所有到它自己HTTP端口的包转发给LAN内部真正的Web服务器。

snat是改写数据包的源地址。应用的场景就是普通的网络地址转换，能够让内部的机器访问外部的服务器。

masquerade和snat类似，只是不用指定具体的ip，可以指定使用某个网络接口的ip
```
//将源地址为172.16.93.0/24的数据包的源地址改写为10.0.0.1
iptables -t nat -A POSTROUTING -s 172.16.93.0/24  -j SNAT --to-source 10.0.0.1
//将源地址为172.16.93.0/24的数据包指定外出接口为eth0,源地址改写为eth0的ip地址
ptables -t nat -A POSTROUTING -s 172.16.93.0/24 -o eth1 -j MASQUEREADE
//将目的地址为10.0.0.1的数据包地址改写为172.16.93.1
iptables -t nat -A PREROUTING -d 10.0.0.1 -j DNAT –-to-destination 172.16.93.1
```










