---
layout: post
title: main 函数背后的故事
author: Triangleowl
categories: linux
permalink: /:categories/:title
---

#### **Table Of Contents**
#### [hello-world的编译过程 ](#hello-world的编译过程)  
#### [hello-world的执行过程 ](#hello-world的执行过程)
* ##### [函数的入口地址 ](#函数的入口地址)
* ##### [__libc_start_main_函数 ](#__libc_start_main_函数)
* ##### [__libc_csu_init函数 ](#__libc_csu_init函数)
* ##### [_init函数 ](#_init函数)

### [hello-world的编译过程](#hello-world的编译过程)  

我们都知道一个简单的 _hello world_ C程序是经过`预处理`、`编译`、`汇编`、`链接`几个阶段最终生成可执行文件的。每个过程的工作可以概括如下：  

* __预处理阶段__. 预处理器(`cpp`)根据以 _**#**_ 开头的代码来修改原始的C程序。比如 _hello world_ 程序将 _#include_ 命令告诉预处理器读取 _stdio.h_ 的内容，并插入到 _hello world_ 程序中生成 _hello.i_ 文件。  

* __编译阶段__. 编译器(`cc1`)将 _hello.i_ 文件翻译成汇编文件 _hello.s_。

* __汇编阶段__. 汇编器(`as`)将 _hello.s_ 文件中的汇编指令翻译成机器语言指令，并把这些指令打包成_可重定位目标文件_，并按照`ELF`文件格式生成 _hello.o_ 文件。

* __链接阶段__. _hello world_ 程序调用了 _printf_ 函数，但是我们的源文件里并没有这个函数的代码，它是一个C语言标准库里的代码。链接阶段就由连接器(`ld`)来确定 _printf_ 函数的地址，从而确保程序在执行时能正确调用 _printf_ 函数。

简单的用命令来概括就很简单了，准备好了 _hello world_ 程序之后，只需下面一个命令就能立马从 _hello world_ 源程序得到可执行程序。  

```C
$ cat hello.c
#include <stdio.h>

int main(int argc, char **argv)
{
	printf("hello world\n");

	return 0;
}

$ gcc hello.c -o hello
```
那么问题来了， _hello world_ 的执行过程是怎样的呢？你可能会说很简单啊，程序成 _main_ 函数开始执行，调用 _printf_ 函数，所以我们能看到一句 _hello world_ ，然后程序退出。没错，程序就是这样运行的，从 _main_ 函数开始的。如果再深究一下：哪个函数调用了 _main_ 函数呢？这篇blog来介绍一下 _main_ 函数是怎么被调用的。



### [hello-world的执行过程](#hello-world的执行过程)

#### [函数的入口地址](#函数的入口地址)

首先我们用`readelf -h`查看一下 _hello_ 的基本信息：

```
$ readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x8048310  //入口地址
  Start of program headers:          52 (bytes into file)
  Start of section headers:          6108 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         31
  Section header string table index: 28
```

_hello_ 程序的入口地址是 _0x8048310_，打开`GDB`在 _0x8048310_ 设置断点，然后运行。程序停在了 *__start_* 函数。

```
$ gdb ./hello -q

(gdb) b *0x8048310
Breakpoint 1 at 0x8048310
(gdb) r
Starting program: /home/me/tmp/hello/hello

Breakpoint 1, 0x08048310 in _start ()

```
我们反汇编一下 *__start_* 函数：

```
(gdb) disas _start
Dump of assembler code for function _start:
   0x08048310 <+0>:	xor    %ebp,%ebp          //将 %ebx 清零
   0x08048312 <+2>:	pop    %esi               //将 argc 弹如 %esi
   0x08048313 <+3>:	mov    %esp,%ecx          //将 argv 存入 %ecx
   0x08048315 <+5>:	and    $0xfffffff0,%esp   
   0x08048318 <+8>:	push   %eax              
   0x08048319 <+9>:	push   %esp             //stck end
   0x0804831a <+10>:	push   %edx             // _rtld_fini 函数地址
   0x0804831b <+11>:	push   $0x80484a0       // _fini 函数地址
   0x08048320 <+16>:	push   $0x8048440       // _init 函数地址
   0x08048325 <+21>:	push   %ecx             // argv
   0x08048326 <+22>:	push   %esi             // argc
   0x08048327 <+23>:	push   $0x804840b       // main 函数地址
   0x0804832c <+28>:	call   0x80482f0 <__libc_start_main@plt>
   0x08048331 <+33>:	hlt
```

首先说明一下，在进入 __start_ 函数数时 stack 里面存放的是 _argc_ 、 _argv_ 、 _env_ 。进入 __start_ 函数后先将`%ebx`清0，然后将 _argc_ 弹到 `%esi`中。此时`%esp`已经指向 _argv_ ，再将 _argv_ 存入`%ecx`中。再将`%esp`设置为16字节对齐。接着连续执行8次压栈操作，这是将 ___libc_start_main_ 函数的的参数压入stack中，但是`%eax`的值对 ___libc_start_main_ 没有影响，这么做是为了对齐。将 ___libc_start_main_ 函数需要的参数都压入stack之后就是调用 ___libc_start_main_ 函数了。可以说 __start_ 函数就是为 ___libc_start_main_ 函数做准备的。  

如果在 ___libc_start_main_ 函数处设一个断点，然后使用`bt`就可以看到 ____libc_start_main_ 函数的参数情况。

```
$ gdb ./hello -q
Reading symbols from ./hello...(no debugging symbols found)...done.
(gdb) b __libc_start_main
Breakpoint 1 at 0x80482f0
(gdb) r
Starting program: /home/me/tmp/hello/hello

Breakpoint 1, __libc_start_main (main=0x804840b <main>, argc=1, argv=0xbffff034,
    init=0x8048440 <__libc_csu_init>, fini=0x80484a0 <__libc_csu_fini>,
    rtld_fini=0xb7fea880 <_dl_fini>, stack_end=0xbffff02c) at ../csu/libc-start.c:134
```



#### [__libc_start_main_函数](#__libc_start_main_函数)

从名字就可以看出来 ___libc_start_main 是一个库函数，所以不能直接调用 ___libc_start_main 函数，而是要通过`PLT`表来确定 ___libc_start_main 函数的地址。 ___libc_start_main 函数所做的工作主要有：

* 处理 `setuid` 、`setgid`安全问题。

* 启动线程。

* 将 `fini`(hello-world程序本身的)和`rtld_fini`(动态连接器的)作为参数传给`at_exit`，使他们被`at_exit`函数调用，完成清理工作。

* 调用init函数

* 调用`main`函数。

* 调用`exit`函数，经将`main`函数的返回值返回给系统。



#### [__libc_csu_init函数](#__libc_csu_init函数)    
___libc_start_main_ 函数调用的 __init_ 函数实际上是 ___libc_csu_init_ 函数，而 ___libc_csu_init_ 函数的也被添加到了 _hello_ 中，使用 `objdump -d`可以看到 ___libc_csu_init_ 函数汇编如下：

```
$ objdump -d hello
...
08048440 <__libc_csu_init>:
 8048440:	55                   	push   %ebp
 8048441:	57                   	push   %edi
 8048442:	56                   	push   %esi
 8048443:	53                   	push   %ebx
 8048444:	e8 f7 fe ff ff       	call   8048340 <__x86.get_pc_thunk.bx>
 8048449:	81 c3 b7 1b 00 00    	add    $0x1bb7,%ebx
 804844f:	83 ec 0c             	sub    $0xc,%esp
 8048452:	8b 6c 24 20          	mov    0x20(%esp),%ebp
 8048456:	8d b3 0c ff ff ff    	lea    -0xf4(%ebx),%esi
 804845c:	e8 47 fe ff ff       	call   80482a8 <_init>
 8048461:	8d 83 08 ff ff ff    	lea    -0xf8(%ebx),%eax
 8048467:	29 c6                	sub    %eax,%esi
 8048469:	c1 fe 02             	sar    $0x2,%esi
 804846c:	85 f6                	test   %esi,%esi
 804846e:	74 25                	je     8048495 <__libc_csu_init+0x55>
 8048470:	31 ff                	xor    %edi,%edi
 8048472:	8d b6 00 00 00 00    	lea    0x0(%esi),%esi
 8048478:	83 ec 04             	sub    $0x4,%esp                        //循环开始
 804847b:	ff 74 24 2c          	pushl  0x2c(%esp)
 804847f:	ff 74 24 2c          	pushl  0x2c(%esp)
 8048483:	55                   	push   %ebp
 8048484:	ff 94 bb 08 ff ff ff 	call   *-0xf8(%ebx,%edi,4)
 804848b:	83 c7 01             	add    $0x1,%edi
 804848e:	83 c4 10             	add    $0x10,%esp
 8048491:	39 f7                	cmp    %esi,%edi
 8048493:	75 e3                	jne    8048478 <__libc_csu_init+0x38>   //循环结束
 8048495:	83 c4 0c             	add    $0xc,%esp
 8048498:	5b                   	pop    %ebx
 8048499:	5e                   	pop    %esi
 804849a:	5f                   	pop    %edi
 804849b:	5d                   	pop    %ebp
 804849c:	c3                   	ret    
 804849d:	8d 76 00             	lea    0x0(%esi),%esi

...

 08048340 <__x86.get_pc_thunk.bx>:
 8048340:	8b 1c 24             	mov    (%esp),%ebx
 8048343:	c3                   	ret    
```

进入 ___libc_csu_init_ 函数后先将要使用到的几个寄存器压入stack中，然后紧接着就调用了 ___x86.get_pc_thunk.bx_ 函数。 ___x86.get_pc_thunk.bx_ 要做的工作就是将 ___libc_csu_init_ 函数中 `call __x86.get_pc_thunk.bx` 指令的下一条指令的地址保存到`%ebx`中。接着 ___libc_csu_init_ 又会调用 __init_ 函数， __init_ 函数的也被添加到了 _hello_ 中。调用完 __init_ 函数之后又会执行一个循环操作，主要是将 _argc_ 、 _argv_ 、 _envp_ 作为参数调用函数。对应的C代码如下：

```
void
__libc_csu_init (int argc, char **argv, char **envp)
{

  _init ();

  const size_t size = __init_array_end - __init_array_start;
  for (size_t i = 0; i < size; i++)
      (*__init_array_start [i]) (argc, argv, envp);
}
```


#### [_init函数](#_init函数)
和 __libc_csu_fini_ 函数一样， __init_ 函数的也被添加到了 _hello_中， __init_函数的汇编代码如下：

```
080482a8 <_init>:
 80482a8:	53                   	push   %ebx
 80482a9:	83 ec 08             	sub    $0x8,%esp
 80482ac:	e8 8f 00 00 00       	call   8048340 <__x86.get_pc_thunk.bx>
 80482b1:	81 c3 4f 1d 00 00    	add    $0x1d4f,%ebx
 80482b7:	8b 83 fc ff ff ff    	mov    -0x4(%ebx),%eax
 80482bd:	85 c0                	test   %eax,%eax
 80482bf:	74 05                	je     80482c6 <_init+0x1e>
 80482c1:	e8 3a 00 00 00       	call   8048300 <__libc_start_main@plt+0x10> (.got.plt)
 80482c6:	83 c4 08             	add    $0x8,%esp
 80482c9:	5b                   	pop    %ebx
 80482ca:	c3                   	ret    

 08048300 <.plt.got>:
 8048300:	ff 25 fc 9f 04 08    	jmp    *0x8049ffc
 8048306:	66 90                	xchg   %ax,%ax
```

__init_ 函数执行完之后返回到 ___libc_csu_init_ 函数。 ___libc_csu_init_ 函数执行完成之后又返回到 ___libc_start_main_ 函数，然后我们的 _main_ 函数就开始被调用了。 _main_ 就不用说了， _main_函数执行完成之后会调用 _exit_ 做好收尾工作。这就是`hello-world`程序执行的大致过程了。

------------------
可能有人会有这样的疑问： _main_ 函数的参数不是有三个( _argc_、 _argv_、 _envp_ )吗？按理说 ___libc_start_main_ 调用 _main_ 函数的时候应该将这三个参数都传递给 _main_ 函数的，但是为什么 ___libc_start_main_ 函数的参数只有 _argc_ 和 _argv_ 而没有 _envp_ 呢？这是因为 ___libc_start_main_ 函数通过调用 ___libc_init_first_ (源代码如下)就可以得到 _envp_ , 并将全局变量 __environ_ 设置为 _envp_ ，所以在 ___libc_start_main_ 函数调用 _main_ 函数时就不需要再显示的传递 _envp_ 参数。

```
// 通过调用 __libc_init_first 函数将 envp 存到全局变量 _environ 中

void __libc_init_first(int argc, char *arg0, ...)
{
    char **argv = &arg0, **envp = &argv[argc + 1];
    __environ = envp;
    __libc_init (argc, argv, envp);
}
```

参考：  
[Linux x86 Program Start Up or - How the heck do we get to main()?](http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html)  
[Linux X86 程序启动 – main函数是如何被执行的？](https://luomuxiaoxiao.com/?p=516)
