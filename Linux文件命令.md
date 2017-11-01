##Linux文件命令
### which
which的作用就是在path指定的路径中搜索某个系统命令的位置并且返回地一个结果
eg：

```
tan@ttt:~/Documents/git_dir/others$ which passwd
/usr/bin/passwd
```

### man
关于man这个命令大家肯定不陌生，经常看man文档的人就知道，为什么man文档是xxx(1)的形式呢？这个数字是什么意思，反正我是用到现在才知道，真是惭愧

原来man文档将文档分成8种大类：

1. Executable programs or shell commands
2. System calls (functions provided by the kernel)
3. Library calls (functions within program libraries)
4. Special files (usually found in /dev)
5. File formats and conventions eg /etc/passwd
6. Games
7. Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7)
8. System administration commands (usually only for root)
9. Kernel routines [Non standard]


如果要准确的访问某个内容可以`man [number] xxx`



