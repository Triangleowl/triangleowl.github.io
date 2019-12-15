---
layout: post
title: AFL 源码码插桩分析
author: Triangleowl
categories: Fuzzing
permalink: /:categories/:title
---

## Table of Contents
#### [afl-gcc分析 ](#afl-gcc分析)
#### [afl-as分析 ](#afl-as分析)
* #### [查找代码部分 ](#查找代码部分)
* #### [查找 basic block ](#查找-basic-block)
* #### [忽略插桩部分 ](#忽略插桩部分)  
------------------------

### [afl-gcc分析](#afl-gcc分析)
`AFL` 既可以对源代码进行插桩也可以结合 `QEMU` 对二进制进行插桩，本问主要针对 `AFL` 的源码插桩进行分析。

我们都知道`C`程序从源代码到可执行文件要经过四个步骤：`预处理`、`编译`、`汇编`、`链接`。每个步骤的工作可以概括如下：  

* __预处理阶段__. 预处理器(`cpp`)根据以 _**#**_ 开头的代码来修改原始的C程序。比如 _hello world_ 程序将 _#include_ 命令告诉预处理器读取 _stdio.h_ 的内容，并插入到 _hello world_ 程序中生成 _hello.i_ 文件。  

* __编译阶段__. 编译器(`cc1`)将 _hello.i_ 文件翻译成汇编文件 _hello.s_。

* __汇编阶段__. 汇编器(`as`)将 _hello.s_ 文件中的汇编指令翻译成机器语言指令，并把这些指令打包成_可重定位目标文件_，并按照`ELF`文件格式生成 _hello.o_ 文件。

* __链接阶段__. _hello world_ 程序调用了 _printf_ 函数，但是我们的源文件里并没有这个函数的代码，它是一个C语言标准库里的代码。链接阶段就由连接器(`ld`)来确定 _printf_ 函数的地址，从而确保程序在执行时能正确调用 _printf_ 函数。

`gcc`其实也是上面几个工具的一个wrapper，首先`gcc`会调用`cpp`对代码进行预处理，然后将预处理后的文件交给`cc1`执行编译生成汇编文件，汇编器`as`将汇编文件作为输入生成机器码--目标文件，最后由连接器`ld`将目标文件链接生成可执行文件。

AFL 实际上是在在编译过程完成后汇编过程之前对汇编文件进行插桩的，AFL 使用`afl-gcc`对C/C++程序源码进行插桩(afl-g++是afl-gcc的一个链接)。而`afl-gcc`又是`gcc/g++`的wrapper,`afl-gcc`设置`-B direcotry`后再调用`gcc/g++`。`afl-gcc`的`main`函数如下：

```
/*
afl-gcc.c:main()
*/

find_as(argv[0]); //查找argv[0]的目录(即afl-gcc的目录)供 edit_params()函数使用

edit_params(argc, argv); //设置 -B 选项和参数

execvp(cc_params[0], (char**)cc_params); //调用 gcc
```

而`gcc/g++`又是`cpp`、`cc1/cc1plus`、`as`、`ld`的wrapper,`-B directory`的作用是让`gcc/g++`首先在 *directory* 查找`cpp`、`cc1/cc1plus`、`as`、`ld`。`afl-gcc`将 *direcotry* 设置为afl-gcc所在的目录，当进行预处理和编译时到 *direcotry* 找不到`cpp`和`cc1/cc1plus`就会使用默认的`cpp`和`cc1/cc1plus`。而当`cc1/cc1plus`编译完成后在 *direcotry* 查找汇编器 `as`，刚好 *dircotry* 有`as`(这个`as`其实是`afl-as`的链接)。接下来就是使用`afl-as`对汇编代码插桩。

### [afl-as分析](#afl-as分析)
`afl-as`是`as`的wrapper，`afl-as`会在汇编代码的代码相应位置插入统计代码，然后调用真正的`as`进行汇编。统计代码是在`afl-as.h`文件中，`afl-as`负责找到每个 *basic block* 插入 `afl-as.h`中的统计代码。`afl-as.c:main()`主要调用了两个函数：
```
/*
afl-as.c:main()
*/

edit_params(argc, argv); //调整传递给真正的汇编器`as`的参数。

add_instrumentation(); //判断分支，插入统计代码
```

#### [查找代码部分](#查找代码部分)
`afl-as.c`中最重要的就是 *add_instrumentation()* 函数，它只对代码部分进行插桩。具体的代码如下,如果是代码部分则会将 `instr_ok` 置1。

```
if (line[0] == '\t' && line[1] == '.') {

  /* OpenBSD puts jump tables directly inline with the code, which is
     a bit annoying. They use a specific format of p2align directives
     around them, so we use that as a signal. */

  if (!clang_mode && instr_ok && !strncmp(line + 2, "p2align ", 8) &&
      isdigit(line[10]) && line[11] == '\n') skip_next_label = 1;

  if (!strncmp(line + 2, "text\n", 5) ||
      !strncmp(line + 2, "section\t.text", 13) ||
      !strncmp(line + 2, "section\t__TEXT,__text", 21) ||
      !strncmp(line + 2, "section __TEXT,__text", 21)) {
    instr_ok = 1;
    continue;
  }

  if (!strncmp(line + 2, "section\t", 8) ||
      !strncmp(line + 2, "section ", 8) ||
      !strncmp(line + 2, "bss\n", 4) ||
      !strncmp(line + 2, "data\n", 5)) {
    instr_ok = 0;
    continue;
  }

}
```

#### [查找 basic block](#查找-basic-block)
`afl-as`通过两种方法来查找 *basic block*, 一种是通过 `标识符`， 另一种是通过条件跳转指令来识别 *basic block*。

`afl-as` 通过 `标识符` 查找 *basic block* 的方法很简单，具体是以 “点号”(`.`)开始,以“冒号”(`:`)结束，中间是字母数字组合。如果找到这种 `标识符` 就将 `instrument_next` 置1。

```
if (strstr(line, ":")) {
      if (line[0] == '.') {
          ...
          instrument_next = 1;
      }
    }
```
另外还有一种情况就是产生条件分支的地方，一般是进行比较根据比较结果来决定是否跳转(如 jnz xxx)，但是条件跳转指令的下一条也是一个 *basic block* 的开始处，但是没有 `标识符`, 所以要通过判断指令本身来决定是否是条件跳转指令，如果是则在在条件跳转指令后插入统计代码。

```
/* Conditional branch instruction (jnz, etc). We append the instrumentation
   right after the branch (to instrument the not-taken path) and at the
   branch destination label (handled later on). */

if (line[0] == '\t') {

  if (line[1] == 'j' && line[2] != 'm' && R(100) < inst_ratio) {

    fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32,
            R(MAP_SIZE));

    ins_lines++;

  }

  continue;

}
```

#### [忽略插桩部分](#忽略插桩部分)
`afl-as` 并不是对所有的 *basic block* 都进行插桩，它会忽略以下集中情况: Intel汇编、源代码内嵌汇编代码。

`cc1/cc1plus`生成的是 `AT&T` 汇编，如果发现 *.intel_syntax* 则跳过，不对Intel汇编插桩。使用源代码内嵌汇编编程时，生成的汇编代码会在内嵌汇编处用 *#APP* 和 *#NO_APP* 来标记。

```
/* Detect syntax changes, as could happen with hand-written assembly.
   Skip Intel blocks, resume instrumentation when back to AT&T. */

if (strstr(line, ".intel_syntax")) skip_intel = 1;
if (strstr(line, ".att_syntax")) skip_intel = 0;

/* Detect and skip ad-hoc __asm__ blocks, likewise skipping them. */

if (line[0] == '#' || line[1] == '#') {

  if (strstr(line, "#APP")) skip_app = 1;
  if (strstr(line, "#NO_APP")) skip_app = 0;

}
```

这就是 `AFL` 的插桩过程，具体的插桩指令 [AFL内部实现细节小记] 写的非常好。


[AFL内部实现细节小记]:https://rk700.github.io/2017/12/28/afl-internals/
