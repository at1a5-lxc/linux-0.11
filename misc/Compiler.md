Linux 0.11内核使用的编译工具
================================================================================

语言编译过程就是把人类能理解的高级语言转换成计算机硬件能理解和执行的二进制机器指令的过程。
这种转换过程通常会产生一些效率不是很高的代码，所以对一些运行效率要求高或性能影响较大的部分代码
通常就会直接使用汇编语言来编写，或者对高级语言编译产生的汇编程序再进行人工修改优化处理。

Linux 0.11主要使用的编译器如下所示:

* 一种是能产生16位代码的as86汇编器，使用配套的ld86链接器；
* 另一种是GNU的汇编器gas（as），使用GNU ld链接器来链接产生的目标文件。
* 编译c语言的编译工具gcc

as86和ld86
--------------------------------------------------------------------------------

as86和ld86是由MINIX-386的主要开发者之一Bruce Evans编写的Intel 8086、80386汇编编译程序和链接程序。
在刚开始开发Linux内核时Linus就已经把它移植到了Linux系统上。它虽然可以为80386处理器编制32位代码，
但是Linux系统仅用它来创建:
* 16位的启动引导扇区程序boot/bootsect.s
* 实模式下初始设置程序boot/setup.s的二进制执行代码。

该编译器快速小巧，并具有一些GNU gas没有的特性，例如宏以及更多的错误检测手段。
不过该编译器的语法与GNU as汇编编译器的语法不兼容，而更近似于微软的MASM、Borland公司的Turbo ASM和NASM等汇编器的语法。
这些汇编器都使用了Intel的汇编语言语法（如操作数的次序与GNU as的相反等）。

as86的语法是基于MINIX系统的汇编语言语法，而MINIX系统的汇编语法则是基于PC/IX系统的汇编器语法。
PC/IX是很早以前在Intel 8086 CPU上运行的一个类UNIX操作系统，Tanenbaum就是在PC/IX系统上进行MINIX系统开发工作的。

Bruce Evans是MINIX操作系统32位版本的主要修改编制者之一，他与Linux的创始人Linus Torvalds是好友。
在Linux内核开发初期，Linus从Bruce Evans那里学到了不少有关类UNIX操作系统的知识。
MINIX操作系统的不足之处也是两个好朋友探讨得出的结果。MINIX的这些缺点正是激发Linus
在Intel 80386体系结构上开发一个全新概念操作系统的主要动力之一。
Linus曾经说过："Bruce是我的英雄"，因此我们可以说Linux操作系统的诞生与Bruce Evans也有着密切的关系。

### as86和ld86的使用方法和选项如下

##### as86的使用方法和选项：

```
as [-03agjuw] [-b [bin]] [-lm [list]]
[-n name] [-o objfile] [-s sym] srcfile
默认设置 (除了以下默认值以外，其他选项默认为关闭或无；
若没有明确说明a标志，则不会有输出):

-3   使用80386的32位输出；
list 在标准输出上显示；
name  源文件的基本名称（即不包括"."后的扩展名）；
各选项含义：
-0  使用16bit代码段；
-3  使用32bit代码段；
-a  开启与GNU as、ld的部分兼容性选项；
-b  产生二进制文件，后面可以跟文件名；
-g  在目标文件中仅存入全局符号；
-j  使所有跳转语句均为长跳转；
-l  产生列表文件，后面可以跟随列表文件名；
-m  在列表中扩展宏定义；
-n  后面跟随模块名称（取代源文件名称放入目标文件中）；
-o  产生目标文件，后跟目标文件名（objfile）；
-s  产生符号文件，后跟符号文件名；
-u  将未定义符号作为输入的未指定段的符号；
-w  不显示警告信息；
```

##### ld86的使用方法语法和选项如下:

```
对于生成Minix a.out格式的版本：
ld [-03Mims[-]]  [-T textaddr] [-llib_extension] [-o outfile] infile...

对于生成GNU-Minix的a.out格式的版本：
ld [-03Mimrs[-]] [-T textaddr] [-llib_extension] [-o outfile] infile...

默认设置(除了以下默认值以外，其他选项默认为关闭或无):
-03  32位输出；
outfile  a.out格式输出；
-0  产生具有16bit魔数的头结构，并且对-lx选项使用i86子目录；
-3  产生具有32bit魔数的头结构，并且对-lx选项使用i386子目录；
-M  在标准输出设备上显示已链接的符号；
-T  后面跟随正文基地址 (使用适合于strtoul的格式)；
-i  分离的指令与数据段（I&D）输出；
-lx 将库/local/lib/subdir/libx.a加入链接的文件列表中；
-m  在标准输出设备上显示已链接的模块；
-o  指定输出文件名，后跟输出文件名；
-r  产生适合于进一步重定位的输出；
-s  在目标文件中删除所有符号。
```

as与ld
--------------------------------------------------------------------------------

### 简介

上面介绍的as86汇编器仅用于编译内核中的boot/bootsect.S引导扇区程序和实模式下的设置程序boot/setup.s。

内核中其余所有汇编语言程序（包括C语言产生的汇编程序）均使用as来编译，并与C语言程序编译产生的模块链接。
本节以80x86 CPU硬件平台为基础介绍Linux内核中使用汇编程序语法和GNU as汇编器（简称as汇编器）的使用方法。
我们首先介绍as汇编语言程序的语法，然后给出常用汇编伪指令（指示符）的含义和使用方法。

由于操作系统许多关键代码要求有很高的执行速度和效率，因此在一个操作系统源代码中通常就会包含大约10%的
起关键作用的汇编语言代码。Linux操作系统也不例外，它的32位初始化代码、所有中断和异常处理过程接口程序
以及很多宏定义都使用了as汇编语言程序或扩展的嵌入汇编语句。是否能够理解这些汇编语言程序的功能也就无疑
成为理解一个操作系统具体实现的关键之一。

在编译C语言程序时，GNU gcc编译器会首先输出一个作为中间结果的as汇编语言文件，然后gcc会调用as汇编器把这个
临时汇编语言程序编译成目标文件。即实际上as汇编器最初是专门用于汇编gcc产生的中间汇编语言程序的，
并非作为一个独立的汇编器使用。因此，as汇编器也支持很多C语言特性，这包括字符、数字和常数表示方法以及表达式形式等方面。

GNU as汇编器最初是仿照BSD 4.2的汇编器进行开发的。现在的as汇编器能够配置成产生很多不同格式的目标文件。
虽然编制的as汇编语言程序与具体采用或生成什么格式的目标文件关系不大，但是在下面介绍中若涉及目标文件格式时，
我们将围绕Linux 0.11系统采用的a.out目标文件格式进行说明。

### 编译as汇编语言程序

使用as汇编器编译一个as汇编语言程序的基本命令行格式如下：

```
as [ 选项 ] [ -o objfile ] [ srcfile.s ...]
其中，objfile是as编译输出的目标文件名；srcfile.s是as的输入汇编语言程序名。
```

如果没有使用输出文件名，那么as会编译输出名称为a.out的默认目标文件。在as程序名之后，命令行上可包含编译选项和文件名。
所有选项可随意放置，但是文件名的放置次序同编译结果密切相关。

一个程序的源程序可以放置在一个或多个文件中，程序的源代码无论怎样分割或放置在几个文件中
都不会改变程序的语义。程序的源代码是所有这些文件按次序的组合结果。每次运行as编译器，
它只编译一个源程序。但一个源程序可由多个文本文件组成（终端的标准输入也是一个文件）。

我们可以在as命令行上给出零个或多个输入文件名。as将会按从左到右的顺序读取这些输入文件的内容。
在命令行上任何位置处的参数若没有特定含义的话，将会被作为一个输入文件名看待。如果在命令行上
没有给出任何文件名，那么as将试图从终端或控制台标准输入中读取输入文件内容。在这种情况下，
若已没有内容要输入时就需要手工键入Ctrl-D组合键来告知as汇编器。
若想在命令行上明确指出把标准输入作为输入文件，那么就需要使用参数" "。

as的输出文件是输入的汇编语言程序编译生成的二进制数据文件，即目标文件。
除非我们使用选项"-o"指定输出文件的名称，否则as将产生名为a.out的输出文件。
目标文件主要用于作为链接器ld的输入文件。
目标文件中包含已汇编过的程序代码、协助ld产生可执行程序的信息，可能还包含调试符号信息。

gcc
--------------------------------------------------------------------------------

GNU gcc对ISO标准C89描述的C语言进行了一些扩展，其中一些扩展部分已经包括进ISO C99标准中。

使用gcc汇编器编译C语言程序时通常会经过4个处理阶段，即:

预处理阶段 --> 编译阶段 --> 汇编阶段 --> 链接阶段, 如下图所示：

https://github.com/leeminghao/doc-linux/blob/master/0.11/misc/gcc.jpg

* 在预处理阶段中: gcc会把C程序传递给C前处理器cpp，对C语言程序中指示符和宏进行替换处理，输出纯C语言代码；
* 在编译阶段: gcc把C语言程序编译生成对应的与机器相关的as汇编语言代码；
* 在汇编阶段: as汇编器会把汇编代码转换成机器指令，并以特定二进制格式输出保存在目标文件中;

最后, GNU ld链接器把程序的相关目标文件组合链接在一起，生成程序的可执行映像文件

深度解析
--------------------------------------------------------------------------------

有关更多链接加载的过程如下:

https://github.com/leeminghao/doc-linux/blob/master/linker/LinkerAndLoader.md