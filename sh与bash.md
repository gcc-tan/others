总结一下Bourne shell(/bin/sh)和Bourne Again shell中遇到的常见问题和概念。环境是ubuntu 14.04的版本

###交互式shell和非交互式shell
交互式shell（interactive shell）和非交互式shell。两者的区别很简单，一个是通过输入命令，给出响应的方式，这就是交互式的shell。另外一个是写在脚本文件中，单个或者批量执行的方式，这个是非交互式。在bash中能够通过`echo $-`查看输出中的i选项是否存在进行判断，如果有i选项，那么就是交互式shell。

###登录shell和非登录shell
登录shell（login shell）和非登录shell（non-login shell）两者最主要的区别是在设置环境变量上的差异。以sh为例，如果sh是登录shell，那么将会执行正常的Bourne shell的登录流程，也就是会执行/etc/profile和~/.profile两个配置文件对shell环境进行设置。如果sh不是登录sh那么这两个配置文件都不会被执行。在命令行执行sh -l那么启动的就是登录shell，如果只是执行sh，那么启动的就是非登录shell。

>在图形界面中使用ctrl + alt + t产生的一个bash的shell一般是非登录的交互式shell，使用su切换用户时，如果有-选项是启用登录shell，如果没有那就是非登录shell。如果使用ctrl + alt + f1，输入用户名字和密码后获得的是非登录shell。在bash中可以使用`shopt -q login_shell && echo 'Login shell' || echo 'Not login shell'`判断是否是login shell。


###shell的配置文件
Bourne shell的配置文件比较简单。一个是/etc/profile。另外一个是~/.profile。Bourne shell程序启动之后（正常的登录shell），先执行/etc/profile。这是一个系统级别的初始化文件，所有用户共享。之后再寻找用户主目录下的.profile文件，这是用户级别的配置，这个文件提供了用户定制和修改shell环境的功能。

Bourne Again shell就要相对复杂。如果是登录shell，首先执行的是/etc/profile（查看profile这个脚本，它还调用了/etc/profile.d/下的所有脚本）。然后会在用户的目录下按照顺序找.bash_profile，.bash_login，.profile文件。如果其中的一个被找到，那么就执行那个文件，剩下的不会被执行。（其实.profile脚本中还调用了.bashrc脚本）。而非登录的shell只会执行.bashrc文件。


