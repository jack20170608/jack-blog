---
title: "GCC入门"
date: 2020-11-13T12:07:43+08:00
categories: ["读书笔记-CSAPP"]
tags: ["CSAPP","深入理解计算机系统第三版","gcc"]
draft: false
---

## 1. GCC 是什么
&emsp;&emsp;GCC的全称是GNU编译器套装(英语: GNU Compiler Collection,简称GCC)，指一套编程语言的编译器，以GPL及LGPL许可证所发行的自由软件，是GNU工具链的组成部分之一。GCC目前可以处理C、C++、Fortran、Pascal、Objective-C、Java、Ada、Go等其他语言。许多操作系统包括多类Unix系统，如Linux及BSD家族都采用GCC作为标准C语言编译器。很多开源项目，包括Linux kernel 和 GNU 工具，都是使用 GCC 进行编译的。

## 2. CentOS8安装GCC
&emsp;&emsp;默认的 CentOS软件源包含了一个，名称为 “Development Tools”软件包组,它包含了GNU编辑器集合，GNU调试器(GDB)，和其他编译软件所必需的开发库和工具例如Git等等。想要安装开发工具软件包，可以以拥有**sudo**权限用户身份或者**root**身份运行下面的命令：

```bash 
sudo dnf group install "Development Tools"
```
这个命令将会安装一系列软件包，包括gcc、g++、make、git等等。你可能还想安装关于如何使用 GNU/Linux开发的手册。
```bash
sudo dnf install man-pages
```
&emsp;&emsp;通过使用```gcc --version```命令打印 GCC 版本，来验证 GCC 编译器是否被成功安装:
```bash 
[jack@localhost ~]$ gcc --version
gcc (GCC) 8.3.1 20191121 (Red Hat 8.3.1-5)
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
&emsp;&emsp;在 CentOS 8 软件源中 GCC 的默认可用版本号为8.3.1。看到这些表面GCC已经在你的CentOS系统上安装好了，你可以开始使用它了。

## 3. 编译HelloWorld
&emsp;&emsp;用VIM创建HelloWorld.c，然后输入最简单的Hello World程序。输入完成后用gcc编译成可执行文件hello。
```cpp
#include <stdio.h>
int main(){
    printf("Hello, World \n");
}
```

```bash
[jack@localhost chapter01]$ vim HelloWorld.c 
[jack@localhost chapter01]$ gcc HelloWorld.c -o hello
[jack@localhost chapter01]$ ./hello 
Hello, World 
[jack@localhost chapter01]$
```
## 4. GCC编译过程
&emsp;&emsp;GCC把源文件**HelloWorld.c**翻译成一个可执行文件hello分为4个阶段完成。如下图所示，执行这4个阶段的程序(预处理、编译器、汇编器和链接器)一起构成了编译系统(compilation system)。
![gcc-steps](gcc-steps.svg)

+ 预处理阶段。预处理器(cpp)根据以字符#开头的命令，修改原始的C程序。比如HelloWorld.c中的第一行```#include<stdio.h>```命令告诉预处理器读取系统头文件stdio.h的内容，并把它直接插入程序的文本中。执行结果得到另外一个C程序，通常是以.i作为文件扩展名。

&emsp;&emsp;在这里我们使用gcc的-E选项，就能实现对源文件的预处理。值得注意的是，默认情况下 gcc -E 指令只会将预处理操作的结果输出到屏幕上，并不会自动保存到某个文件。因此该指令往往会和 -o 选项连用，将结果导入到指令的文件中。比如：
```bash
[jack@localhost chapter01]$ gcc -E HelloWorld.c -o HelloWorld.i
[jack@localhost chapter01]$ ls
hello  HelloWorld.c  HelloWorld.i
[jack@localhost chapter01]$ tail  HelloWorld.i
extern int __overflow (FILE *, int);
# 879 "/usr/include/stdio.h" 3 4
# 2 "HelloWorld.c" 2
# 3 "HelloWorld.c"
int main(){
    printf("Hello, World \n");
}
```
默认生成的文件会去掉头文件中的注释，如果想生成注释，可以使用-C选项，可以看到使用-C后文件的行数增加了很多。
```bash
[jack@localhost chapter01]$ gcc -E -C HelloWorld.c -o HelloWorld.i
[jack@localhost chapter01]$ wc -l HelloWorld.i
1965 HelloWorld.i
[jack@localhost chapter01]$ gcc -E HelloWorld.c -o HelloWorld.i
[jack@localhost chapter01]$ wc -l HelloWorld.i
719 HelloWorld.i
```

+ 编译阶段。编译器(ccl)将预处理文件(HelloWorld.i)翻译成文本文件(HelloWorld.s),他包含一个汇编语言程序。该程序包含了函数main的定义，如下所示：
```as
        .file   "HelloWorld.c"
        .text
        .section        .rodata
.LC0:
        .string "Hello, World "
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    $.LC0, %edi
        call    puts
        movl    $0, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (GNU) 8.3.1 20191121 (Red Hat 8.3.1-5)"
        .section        .note.GNU-stack,"",@progbits

```
&emsp;&emsp;汇编语言程序提供了一个低级的机器语言指令，他为不同的高级语言的不同编译器提供了通用的输出语法。在这里可以使用GCC的-S指令把预编译代码经过一系列的词法分析、语法分析、语义分析以及优化加工等操作转换成当前机器支持的汇编程序。
```bash
[jack@localhost chapter01]$ gcc -S HelloWorld.i
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ ls
hello  HelloWorld.c  HelloWorld.i  HelloWorld.s
```

+ 汇编阶段。汇编器(as)将HelloWorld.s翻译成为机器语言指令(二进制)，把这些指令打包成一种叫可重定位目标程序(relocatable object program)的格式，并将结果保存到HelloWorld.o中。简单地理解，汇编其实就是将汇编代码转换成可以执行的机器指令。大部分汇编语句对应一条机器指令，有的汇编语句对应多条机器指令。相对于编译操作，汇编过程会简单很多，它并没有复杂的语法，也没有语义，也不需要做指令优化，只需要根据汇编语句和机器指令的对照表一一翻译即可。
&emsp;&emsp;通过为 gcc 指令添加 -c 选项（注意是小写字母 c），即可让 GCC 编译器将指定文件加工至汇编阶段，并生成相应的目标文件。例如：
```bash
[jack@localhost chapter01]$ ls
hello  HelloWorld.c  HelloWorld.i  HelloWorld.s
[jack@localhost chapter01]$ gcc -c HelloWorld.s
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ ls
hello  HelloWorld.c  HelloWorld.i  HelloWorld.o  HelloWorld.s
```

+ 链接阶段。我们注意到，HelloWorld程序中调用了printf函数，它是每个C编译器都会提供的标准C库中的一个函数。而printf函数实际存在于一个名为printf.o的单独预编译好了的目标文件中，而这个文件必须以某种方式合并到我们的HelloWorld.o的程序中。链接器(id)就负责处理这种合并，结果就得到hello这个可执行目标文件，它可以被加载到内存，由系统执行。

&emsp;&emsp;GCC如果不指定任何选项，会检测输入文件的类型，自行判断是源文件(.c)、预处理文件(.i)、汇编文件(.s)、目标文件(.o)，自动执行需要的操作以生成可执行文件。如下面的示例，GCC会对HelloWorld.c、HelloWorld.i、HelloWorld.s以及HelloWorld.o分别生成这4个可执行文件。
```bash
[jack@localhost chapter01]$ ls
HelloWorld.c  HelloWorld.i  HelloWorld.o  HelloWorld.s
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ 
[jack@localhost chapter01]$ gcc HelloWorld.c -o h1
[jack@localhost chapter01]$ gcc HelloWorld.i -o h2
[jack@localhost chapter01]$ gcc HelloWorld.s -o h3
[jack@localhost chapter01]$ gcc HelloWorld.o -o h4
[jack@localhost chapter01]$ ./h1
Hello, World 
[jack@localhost chapter01]$ ./h2
Hello, World 
[jack@localhost chapter01]$ ./h3
Hello, World 
[jack@localhost chapter01]$ ./h4
Hello, World 
[jack@localhost chapter01]$ ls
h1  h2  h3  h4  HelloWorld.c  HelloWorld.i  HelloWorld.o  HelloWorld.s
```
&emsp;&emsp;常用入门命令简单归纳：

| 指令标签 | 示例                                | 描述                                                                                 |
| -------- | ----------------------------------- | ------------------------------------------------------------------------------------ |
| -E       | gcc -E HelloWorld.c -o HelloWorld.i | 源文件预编译                                                                         |
| -S       | gcc -S HelloWorld.c -o HelloWorld.s | 源文件编译生成汇编文件                                                               |
| -c       | gcc -o HelloWorld.c -o HelloWorld.o | 源文件生成机器指令文件                                                               |
| -o       |                                     | 指定输出文件名，可以配合以上三种标签使用, 参数经常可以省略，省略后会以默认名称输出。 |
| 无标签   | gcc HelloWorld.c -o hello           | 生成名为a.out的可执行文件                                                            |
| -g       | gcc -g HelloWorld.c -o hello        | 生成可执行文件并添加调试信息                                                         |



## 5. GCC处理多文件项目

&emsp;&emsp;在一个C项目中，往往在存储多个源文件，如果仍按照之前“先单独编译各个源文件，再将它们链接起来”的方法编译该项目，需要编写大量的编译指令，事倍功半。事实上，利用 gcc 指令可以同时处理多个文件的特性，可以大大提高我们的工作效率。例如下面是一个有2个源文件和一个头文件的C语言项目：
```text
[jack@localhost demo2]$ ls
add.c  add.h  main.c
[jack@localhost demo2]$ cat add.h
#include <stdio.h>

int add(int a, int b);
[jack@localhost demo2]$ cat add.c
#include "add.h"

int add(int a, int b){
	return a + b;
}
[jack@localhost demo2]$ cat main.c
#include <stdio.h>
#include "add.h"

int main(){
	int a = 1, b = 2;
	int c = add(1 , 2);
	printf("c is %d\n",c);
}
```
如果采用分别编译，试试如下命令：
```bash
[jack@localhost demo2]$ gcc -c add.c -include add.h
[jack@localhost demo2]$ 
[jack@localhost demo2]$ ls
add.c  add.h  add.o  main.c
[jack@localhost demo2]$ gcc -c main.c -include add.h
[jack@localhost demo2]$ 
[jack@localhost demo2]$ ls
add.c  add.h  add.o  main.c  main.o
[jack@localhost demo2]$ gcc add.o main.o -o main
[jack@localhost demo2]$ 
[jack@localhost demo2]$ ls
add.c  add.h  add.o  main  main.c  main.o
[jack@localhost demo2]$ ./main
c is 3
```
但是如果文件数量太大，写起来就太麻烦，也可以共用一条gcc命令来实现。
```bash
[jack@localhost demo2]$ ls
add.c  add.h  main.c
[jack@localhost demo2]$ gcc main.c -o main -include add.h add.c
[jack@localhost demo2]$ ls
add.c  add.h  main  main.c
[jack@localhost demo2]$ ./main
c is 3

```

## 其他 
&emsp; &emsp; gcc其实包含了很多内容，例如链接库的设置、参数优化、，仅仅作为入门文章就不再赘续了，有兴趣的读者可以继续在互联网上找相关的资料。

## 参考资料

1. [GCC Home](https://gcc.gnu.org)
2. [维基百科GCC](https://zh.wikipedia.org/wiki/GCC)
3. [如何在 CentOS 8 上安装 GCC](https://cloud.tencent.com/developer/article/1626637)
4. [GCC是什么？](http://c.biancheng.net/view/7930.html)