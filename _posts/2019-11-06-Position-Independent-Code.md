---
layout: post
title: PLT和GOT--实现位置无关代码
author: Triangleowl
categories: linux
permalink: /:categories/:title
---
#### Table Of Contents
* #### [链接时重定位 ](#链接时重定位)
* #### [装载时重定位 ](#装载时重定位)
    * ##### [模块内部数据访问 ](#模块内部数据访问)
    * ##### [模块间数据访问 ](#模块间数据访问)
    * ##### [模块间函数调用 ](#模块间函数调用)
------------------------------------------------

### [链接时重定位](#链接时重定位)

程序是如何定位来自外部的符号（如引用外部的变量和函数）？ 一个简单的例子如下：

```
$ cat test.c
extern int foo;

int function(void)
{
	return foo;
}

#编译程序，生成目标文件test.o
$ gcc -c test.c

#将目标文件反汇编
$ objdump -d test.o
...
00000000 <function>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	a1 00 00 00 00       	mov    0x0,%eax
   8:	5d                   	pop    %ebp
   9:	c3                   	ret
...
```

可以看到本来应该将`foo`的值赋给`%eax`，但是汇编代码显示将0x0赋给了`%eax`。这是因为test.o无法定
位来自外部的`foo`，这等到链接过程由链接器来确定`foo`，如果使用链接器将`foo`与另一个目标文件的
`foo`绑定则在生成可执行文件时就可以确定`foo`的地址。也就是说上面的例子对`foo`的重定位是在链接时
完成的，叫做`链接时重定位(Link Time Relocation)`。

链接时重定位的代码需要指定加载地址，当加载器加载代码时只要按照指定的加载地址即可。这种代码并不是
位置无关代码(PIC)。如查看`/bin/bash`的program headers:

```
$ readelf --headers /bin/bash
...
Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  ...
  LOAD           0x000000 0x08048000 0x08048000 0x1098d8 0x1098d8 R E 0x1000
  LOAD           0x109ef8 0x08152ef8 0x08152ef8 0x049b0 0x09974 RW  0x1000
  ...
...

```
可以看到`/bin/bash`并不是位置无关代码，因为它的代码(`R``-``E`)将被加载到地址0x08048000处，数据(`R``W``-`)将被加载
到地址0x08152ef8处。

共享库就不能被指定加载地址，而是应该可以被加载到进程运行的任意地址，这就要采用位置无关代码技术。
如`ld-linux.so.2`共享库的program heasers中就没有指定加载地址。

```
$ readelf --headers /lib/ld-linux.so.2
...
Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x00000000 0x00000000 0x2220c 0x2220c R E 0x1000
  LOAD           0x022c80 0x00023c80 0x00023c80 0x00bd8 0x00c98 RW  0x1000
  ...
...
```

### [装载时重定位](#装载时重定位)

#### [模块内部数据访问](#模块内部数据访问)
接下来我们自己编译一个位置无关代码(使用gcc编译时添加 -fPIC 和 -shared选项)，然后将目标代码反
汇编，看看是怎么定位符号的。

```
$ cat test2.c
static int foo = 100;//静态变量

int function(void)
{
	return foo;
}

#加上-fPIC -shared 选项，生成位置无关代码
$ gcc -fPIC -shared -o libtest2.so test2.c
```
使用`objumpd`反汇编`libtest2.so`可以看到function()对应的汇编代码，调用了__x86.get_pc_thunk.ax,
__x86.get_pc_thunk.ax的代码很简单，只是将`(%esp)`的值传给`%eax`。有人可能会有疑问，这么简单
为什么还要封装成函数？因为处理器不支持直接获取当前程序的位置也就是`PC`，所以通过调用
__x86.get_pc_thunk.ax函数就是为了达到获取当前的`PC`。在调用__x86.get_pc_thunk.ax会先把`call`
的下一条指令的地址压如stack，当程序流程转向__x86.get_pc_thunk.ax时`%esp`正好指向`add    $0x1b28,%eax`
指令的地址。就可以将PC存入`%eax`了。


```
$ objdump -d libtest2.so
...
000004d0 <function>:
 4d0:	55                   	push   %ebp
 4d1:	89 e5                	mov    %esp,%ebp
 4d3:	e8 0d 00 00 00       	call   4e5 <__x86.get_pc_thunk.ax>
 4d8:	05 28 1b 00 00       	add    $0x1b28,%eax
 4dd:	8b 80 10 00 00 00    	mov    0x10(%eax),%eax
 4e3:	5d                   	pop    %ebp
 4e4:	c3                   	ret
...

000004e5 <__x86.get_pc_thunk.ax>:
 4e5:	8b 04 24             	mov    (%esp),%eax
 4e8:	c3                   	ret
...
```
执行完__x86.get_pc_thunk.ax返回function()后将`%eax`的值就被赋值为0x4d8, 然后将`%eax`加上0x1b28即：
0x4d8 + 0x1b28 = 0x2000。`%eax` + 0x10 = 0x2010。这里存放的应该就是`foo`的值了。反汇编一下看看是不是。

```
$ objdump --disassemble-all libtest2.so
...
00002010 <foo>:
    2010:	64 00 00             	add    %al,%fs:(%eax)
	...
...
```
果然在0x2010处存放的是十六进制的0x64,即十进制的100。

#### [模块间数据访问](#模块间数据访问)

回想一下上面的程序，虽然是位置无关代码，但是变量`foo`是一个`static`变量，所以要访问`foo`只要知道`foo`和当前
指令的偏移量就够了。但是如果引用了来自其他共享库中的变量会怎么样呢？因为共享库的加载地址是不确定的，那如何
做到引用外部变量而不出错呢？来看看下面的程序是如何实现重定位的。

稍微调整一下上面的代码，并添加 `-fPIC` `-shared`选项编译：

```
extern int foo;//外部变量

int function()
{
	return foo;
}

$ gcc -fPIC -shared -o libtest3.so test3.c
```
接着反汇编`libtest3.so`:

```
$ objdump -d libtest3.so
...
000004f0 <function>:
 4f0:	55                   	push   %ebp
 4f1:	89 e5                	mov    %esp,%ebp
 4f3:	e8 0f 00 00 00       	call   507 <__x86.get_pc_thunk.ax>
 4f8:	05 08 1b 00 00       	add    $0x1b08,%eax
 4fd:	8b 80 f4 ff ff ff    	mov    -0xc(%eax),%eax
 503:	8b 00                	mov    (%eax),%eax
 505:	5d                   	pop    %ebp
 506:	c3                   	ret
...
```
仔细观察可以发现`libtest3.so`和`libtest2.so`还是有点区别的。调用__x86.get_pc_thunk.ax之后
`%eax`值是0x4f8, `%eax` + 0x1b08 = 0x2000，0x2000 - 0x0c = 0x1ff4。也就是说0x1ff4处存放的应该是
foo的值。查看`libtest3.so`的重定位表可以发现0x1ff4处是一个类型为`R_386_GLOB_DAT`的值。当
`libtest3.so`被加载时，动态链接器会检查重定位表，并将`foo`的值放到`.got`条目中的对应位置(0x1ff4).

```
$ objdump -R libtest3.so

DYNAMIC RELOCATION RECORDS
...
OFFSET   TYPE              VALUE
00001ff4 R_386_GLOB_DAT    foo
...
```
从这个例子可以发现位置无关代码引用外部的变量与代码加载到内存的位置无关，因为代码指定从`.got`
中取数据，因为不能修改共享的代码，而`.got`属于数据，可以对其进行写操作，所以只要在加载时让动
态链接器找到对应的变量的值并存入到`.got`中，那么代码就可以引用了。这样就能保证在不修改代码的
情况下完成数据的访问。

#### [模块间函数调用](#模块间函数调用)
对于动态库函数的引用，理论上讲可以采用和[模块间数据访问](#模块间数据访问)一样的方法来实现函数地址
解析，但是ELF采用了另外一种更复杂而又精妙的方法。不仅要使用`GOT`表，还用到了`PLT`表。

再看一个例子：

```
$ cat test.c
extern int foo;

int function(void)
{
	return foo;
}

$ gcc -fPIC -shared -o libtest4.so test4.c
```
还是将`libtest4.so`反汇编进行分析。可以看到对`foo`函数的调用是通过`foo@plt`来定位`foo`的地址。

```
$ objdump -d libtest4.so
...
00000500 <function>:
 500:	55                   	push   %ebp
 501:	89 e5                	mov    %esp,%ebp
 503:	53                   	push   %ebx
 504:	83 ec 04             	sub    $0x4,%esp
 507:	e8 12 00 00 00       	call   51e <__x86.get_pc_thunk.ax>
 50c:	05 f4 1a 00 00       	add    $0x1af4,%eax
 511:	89 c3                	mov    %eax,%ebx
 513:	e8 98 fe ff ff       	call   3b0 <foo@plt>
 518:	83 c4 04             	add    $0x4,%esp
 51b:	5b                   	pop    %ebx
 51c:	5d                   	pop    %ebp
 51d:	c3                   	ret
...


```

`%eax`存放的是0x50c, `%ebx`存放的是`%eax`+0x1af4 = 0x2000。使用`readelf --sections`可以看到`.got.plt`的
地址就是0x2000,其实0x2000就是`过程链接表`(Procedure Linkage Table, PLT)。

```
$ readelf --sections libtest4.so
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  ...
  [20] .got              PROGBITS        00001fec 000fec 000014 04  WA  0   0  4
  [21] .got.plt          PROGBITS        00002000 001000 000010 04  WA  0   0  4
  ...
```

`PLT`和共享库的代码一样是多个进程共享的，也就是说不能改写`PLT`，但是可以通过`PLT`来改写数据部分来确定
引用外部符号的绝对地址。并且可执行文件的`PLT`表和共享目标文件的`PLT`是分开的。利用`PLT`表确定引用的。
`PLT`的结构如下:

```
Procedure Linkage Table

.PLT0:  pushl 4(%ebx)
        jmp *8(%ebx)
        nop; nop
        nop; nop
.PLT1:  jmp *name1@GOT(%ebx)
        pushl $offset
        jmp .PLT0@PC
.PLT2:  jmp *name2@GOT(%ebx)
        pushl $offset
        jmp .PLT0@PC
```
通过`PLT`来确定`foo`的地址可以分为一下几个步骤：
* 将共享库加载到内存，动态连接器将`GOT`表的第二项(`GOT[1]`)和第三项('GOT[2]')分别设置为特殊值。
`PLT[0]`也是一个特殊的条目。

* 调用者(caller)负责将`GOT`(注意是`.got.plt`, 不是`.got`)表的地址存放在`%ebx`中(注意，ELF规定必须
将`GOT`的地址放入到`%ebx`而不是其他寄存器)。  
```
 507:	e8 12 00 00 00       	call   51e <__x86.get_pc_thunk.ax>
 50c:	05 f4 1a 00 00       	add    $0x1af4,%eax
 511:	89 c3                	mov    %eax,%ebx
```

* 然后调到`foo`对应的`PLT`条目(`PLT[1]`)，即`foo@plt`。当首次调用`foo`前，`foo`对应的`GOT`表存放的是`foo`的
`PLT`条目的第二条指令`pushl 0x00`的地址，而不是`foo`的绝对地址。  
```
# foo 的 PLT
000003b0 <foo@plt>:
 3b0:	ff a3 0c 00 00 00    	jmp    *0xc(%ebx)
 3b6:	68 00 00 00 00       	push   $0x0
 3bb:	e9 e0 ff ff ff       	jmp    3a0 <_init+0x28>
```

* 然后跳转到`PLT`的`PLT[1]`的`pushl 0x0`指令执行。这里的`0x0`是重定位偏移，它是一个非负数。它是和`GOT`对应的。
在将重定位偏移量压如stack之后，程序就跳转到了`PLT[0]`。`PLT[0]`将`GOT`的第第二项`GOT[1]`(`%ebx`+0x4)压入
stack中。然后跳转到`GOT[2]`(`%ebx`+0x8)处,`GOT[2]`将控制权交给动态链接器。
```
# PLT[0]
000003a0 <foo@plt-0x10>:
 3a0:	ff b3 04 00 00 00    	pushl  0x4(%ebx)
 3a6:	ff a3 08 00 00 00    	jmp    *0x8(%ebx)
 3ac:	00 00                	add    %al,(%eax)
```
* 当动态链接器拿到控制权之后会根据压入stack的参数来确定引用的符号的值，然后将此值写入对应的`GOT`中，然后将控制
权还回给用调用程序。

![第一次调用foo](https://github.com/Triangleowl/picture/blob/master/GOT/GOT1.png?raw=true "第一次调用foo")


至此`foo`的`GOT`表中存放的已经是`foo`的绝对地址了，当再次代用`foo`时就不会像第一次调用这么麻烦，因为`GOT`表中已经
有`foo`的地址了。

![第二次调用foo](https://github.com/Triangleowl/picture/blob/master/GOT/GOT2.png?raw=true "第二次调用foo")




----
参考：  
《深入理解计算机系统》  
《程序员的自我修养》  
[PLT and GOT - the key to code sharing and dynamic libraries](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html)  
[ELF format](http://www.skyfree.org/linux/references/ELF_Format.pdf)  
