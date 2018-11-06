# Vim操作

经常用vim打开文件进行编辑之后发现没有写权限，这时已经修改了很多文件内容，不得不退出vim，然后使用sudo vi重新编辑。面对这种情况，可以使用这条ex命令`w !sudo tee %`保存所以写的内容。命令解释：w !cmd作用是将当前缓冲区的内容作为外部cmd的标准输入，然后tee是从标准输入读入，写入标准输出和文件的工具，%是vim的一个只读寄存器，里面保存的是vim被编辑文件的名字。

## 配色
ubuntu首先的配色好像能看清，然后不记得装了什么之后配色注释成蓝色了完全看不清楚，然胡其实在/usr/share/vim/vim74/colors/目录下面有多种配色方案，首先是default，在.vimrc里面加入你的选择`colorscheme desert`就行
## 对齐文本
+ 全部对齐 `gg=G`
+ visual模式下选中文本加上`=`
+ 对当行格式化`==`
+ 对以下多行格式化`[count]==`
+ 选择多行后，执行`=`

使用命令`:%!python -m json.tool`可以格式化json串，这条命令的含义是%表示整个文档范围，然后作为外部命令的管道输入，感叹号开始的是执行外部的shell命令，调用python的json.tool模块进行格式化。最后执行的结果覆盖原来的缓冲区

## 命令格式

vim的命令采用下面的格式。

```
[OPERATOR][NUMBER][MOTION]
```

Operator是动词。

- d – Delete (等同于cut命令)
- c – Change
- y – Yank
- p – Insert last deleted text after cursor (put command)
- r – Replace
- v - 可视化选择

Motion表示操作的上下文。

- w – 直到下一个单词的起始位置前面。
- s - sentence
- p - paragraph
- t - tag
- b - block
- e – 直到当前单词的最后一个位置。
- $ – 直到当前行的最后一个位置。
- ) – 下一个句子的开始。
- ( – 当前句子的开始。
- } – 下一段的开始。
- { – 当前段的开始。
- ] – 下一段部分（section）的开始
- `[` – 当前部分（section）的开始
- `H` – 当前屏幕的顶部行
- `L` – 当前屏幕的最后一行

Count是可选的，表示command和motion的重复次数

- i - inside
- a - around
- NUM: number (e.g.: 1, 2, 10)

实例

- dw 删除一个词
- d4w 删除四个词
- d$ 删除当前行
- dd 删除当前行（d$的快捷方式）
- d2$ 删除两行
- cis - Change inside sentence，删除当前句子，并进入insert模式
- yip - yank inside paragrah 复制当前段落

## 撤销命令

- u 撤销上个命令
- g+ 回到上一个新的版本(是加号)
- g- 回到上一个老的版本
## 翻页

+ 整页翻ctrl + f,ctrl + b  f就是forward，b就是backword
+ 半页翻就是ctrl + d，ctrl + u d就是down，u就是up 

## 移动光标
- h – Left
- k – Up
- l – Right
- j – Down
- G 移动到文件最后一行
- 123 + G 跳到指定行
- gg 移动到文件第一行
- 在正常模式输入zz能够使得光标在正中间，方便正常书写代码，在插入模式下ctrl+o然后再输入zz
- ctrl + g 查看当前文件总行数
- % 移动到当前代码区块的开始/结尾（匹配`()`，`[]`，`{}`）

## 插入文字

- i 当前位置前面
- a 当前位置后面
- o 当前行下方新增一行
- O 当前行上方新增一行

## 删除

- x 删除当前字符

## 搜索，替换

语法为 :  
    `[addr]s/源字符串/目的字符串/[option] ` 


[addr] 表示检索范围，省略时表示当前行  

-  如:1,20 表示从第1行到20行  
-  % 表示整个文件，同“1,$”；  
- .,$  从当前行到文件尾；  
-  s    表示替换操作

[option] : 表示操作类型  

- g 表示全局替换，没加这个标志表示替换行的第一次出现    
- c 表示进行确认  
- p 表示替代结果逐行显示（trl + L恢复屏幕)  
- 省略option时仅对每行第一个匹配串进行替换  
- 如果在源字符串和目的字符串中出现特殊字符，需要用”\”转义









## 执行shell命令

- :!ls -al

## 复制，粘贴，剪切

### 选择文本

- v+光标移动 （按字符选择）高亮选中所要的文本，然后进行各种操作（比如，d表示删除）。
- V （按行选择）
- v+选中的内容+c 更改选中的文字

### 复制：y(ank)

- y 用v命令选中文本后，用y进行复制
- yy 复制当前行，然后用p进行复制
- 5yy 复制从当前行开始的5行
- y_ 等同于yy
- Y 等同于yy
- yw 复制当前单词
- y$ 从当前位置复制到行尾
- y0 从当前位置复制到行首
- y^ 从当前位置复制到第一个非空白字符
- yG 从当前行复制到文件结束
- y20G 从当前行复制到第20行
- y?bar 复制至上一个出现bar的位置

### 粘贴

- p 在光标位置之后粘贴
- P 在光标位置之前粘贴

### 剪切

- v + 选中的内容 + d 剪切

### 剪贴板

（1） 简单复制和粘贴

vim提供12个剪贴板，它们的名字分别为vim有11个粘贴板，分别是`0`、`1`、`2`、`...`、`9`、`a`、`“`。如果开启了系统剪贴板，则会另外多出两个：`+`和`*`。使用`:reg`命令，可以查看各个粘贴板里的内容。

```bash
:reg
```

在vim中简单用y只是复制到`“`（双引号)粘贴板里，同样用p粘贴的也是这个粘贴板里的内容。

（2）复制和粘贴到指定剪贴板

要将vim的内容复制到某个粘贴板，需要退出编辑模式，进入正常模式后，选择要复制的内容，然后按"Ny完成复制，其中N为粘贴板号（注意是按一下双引号然后按粘贴板号最后按y），例如要把内容复制到粘贴板a，选中内容后按"ay就可以了。

要将vim某个粘贴板里的内容粘贴进来，需要退出编辑模式，在正常模式按"Np，其中N为粘贴板号。比如，可以按"5p将5号粘贴板里的内容粘贴进来，也可以按"+p将系统全局粘贴板里的内容粘贴进来。

（3）系统剪贴板

Vim支持系统剪贴板，需要打开clipboard功能。使用下面的命令，检查当前版本的Vim，是否支持clipboard。

```bash
$ vim --version
```

如果支持，会出现clipboard字样，并且前面有一个+号。


如果不支持的话，需要安装图形化界面的vim（即gvim），或者重新编译vim。

```bash
$ sudo apt-get install gvim
正在读取软件包列表... 完成
正在分析软件包的依赖关系树
正在读取状态信息... 完成
Package gvim is a virtual package provided by:
  vim-gtk 2:7.4.488-7
  vim-gnome 2:7.4.488-7
  vim-athena 2:7.4.488-7
You should explicitly select one to install.

E: Package 'gvim' has no installation candidate

$ sudo apt-get install vim-gnome
```

另一种方法，是安装vim-gui-common。

```bash
$ sudo apt-get install vim-gui-common
```

安装以后，可以用命令行界面，启动gvim，这时可用系统剪贴板.

```bash
$ gvim -v
```

星号（`*`）和加号（`+`）粘贴板是系统粘贴板。在windows系统下， * 和 + 剪贴板是相同的。对于 X11 系统， * 剪贴板存放选中或者高亮的内容， + 剪贴板存放复制或剪贴的内容。打开clipboard选项，可以访问 + 剪贴板；打开xterm_clipboard，可以访问 * 剪贴板。 * 剪贴板的一个作用是，在vim的一个窗口选中的内容，可以在vim的另一个窗口取出。

复制到系统剪贴板
- `"*y`
- `"+y`
- `"+2yy` – 复制两行
- `{Visual}"+y` - copy the selected text into the system clipboard
- `"+y{motion}` - copy the text specified by {motion} into the system clipboard
- `:[range] y +` - copy the text specified by `[range]` into the system clipboard

剪切到系统剪贴板
- `"+dd` – 剪切一行

从系统剪贴板粘贴到vim
- `"*p`
- `"+p`
- `Shift+Insert`
- `:put +` - Ex command puts contents of system clipboard on a new line
- `<C-r>`+ - From insert mode (or commandline mode)

`"+p`比 Ctrl-v 命令更好，它可以更快更可靠地处理大块文本的粘贴，也能够避免粘贴大量文本时，发生每行行首的自动缩进累积，因为`Ctrl-v`是通过系统缓存的stream处理，一行一行地处理粘贴的文本。

## 多窗口

垂直切分窗口，Ctrl-w + s 或者使用下面的命令。

```bash
:split <文件名>
```

水平切分窗口，Ctrl-w + v 或者使用下面的命令。

```bash
:vsplit <文件名>
```

如果省略文件名，则打开的是当前文件。

切换窗口的命令。

- Ctrl-w +  Ctrl-w
- Ctrl-w + direction key

## vimrc文件配置

打开语法高亮

```bash
:syntax on
```

禁止使用箭头键。

```bash
nnoremap <Left> :echoe "Use h"<CR>
nnoremap <Right> :echoe "Use l"<CR>
nnoremap <Up> :echoe "Use k"<CR>
nnoremap <Down> :echoe "Use j"<CR>
```

在窗口间移动。

```bash
nnoremap <c-j> <c-w>j
nnoremap <c-k> <c-w>k
nnoremap <c-h> <c-w>h
nnoremap <c-l> <c-w>l
```

## 命令行模式

```bash
# 列出所有buffer
:ls

# 列出所有buffer（包括不可见buffer）
:ls!

# 在当前窗口打开一个新的文件，
# 新建一个buffer，原有文件成为不可见buffer
:e file1

# 新建一个未命名的buffer，然后将其存为 /tmp/foo
:enew
:w /tmp/foo
```


### 多窗口管理
#### 分屏启动vim
大写的On 垂直分屏  小写的水平 

    vim -on file1 file2 file3  
    vim -On file1 file2 file3
#### 移动光标
 先按ctrl + w 然后就是利用h,j,k,l移动光标了
 ctrl + w 按两次也可以
#### 移动分屏
 先按ctrl + w 然后是H，J，K，L移动屏幕(H向左，L向右，J向下，K向上)
#### 分割
: + split  filename 水平分割
: + vsplit filename 垂直分割
#### 打开多个文件
`vi file1 file2 file3`  
用: n file2 跳转到指定的第二个文件













