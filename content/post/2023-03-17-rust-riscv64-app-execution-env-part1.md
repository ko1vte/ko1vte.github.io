---
title: "探究操作系统中应用程序的执行环境 （以Rust在riscv64gc平台为例子）（上篇）"
slug: "rust-riscv64-app-execution-env-part1"
date: 2023-03-17T16:58:14+08:00
tags: ["OS, Rust"]
aliases:
  - /SU15VTE/ca852c4f.html
  - /SU15VTE/ca852c4f
image: img/daliy.jpg
---

# 导言

*在读本文前，你需要知道的是，你不需要完全读懂本文的代码部分，重点在于理解操作系统中应用程序的执行环境结构。*\
在上个学期，我们已经学习了《计算机操作系统》这门课程，在这门课程里，我们主要学习的是操作系统的各种调度算法，包括实验也是编写某些调度算法的代码。而在计算机领域，操作系统扮演着一个十分重要的角色，它作为计算机硬件和用户应用程序之间的重要中介，操作系统为应用程序提供了执行环境。在我们的生活中，我们使用手机看视频，使用电脑查阅资料，用的是一个又一个的应用程序，而这些应用程序能在我们的硬件设备上运行，中间到底发生了些什么呢？本文将会围绕一个简单的 **“Hello World!”程序**，讲述计算机操作系统的原理和作用，为大家提供一个更为全面的视角。

# 应用程序运行发生了什么？

*以最简单的“hello world!”程序为例*\
“hello world!”这个程序相信是大家编写的第一个程序，而在我们第一次看到电脑中的terminal弹出“hello world!”时，是否有想过在运行时，计算机内发生了些什么呢？我们这里以rust为例。

``` bash
SU15VTE@SU15VTE:~/Desktop/code$ cargo new helloworld
     Created binary (application) `helloworld` package
SU15VTE@SU15VTE:~/Desktop/code$ cd helloworld/
SU15VTE@SU15VTE:~/Desktop/code/helloworld$ cargo run
   Compiling helloworld v0.1.0 (/home/SU15VTE/Desktop/code/helloworld)
    Finished dev [unoptimized + debuginfo] target(s) in 0.53s
     Running `target/debug/helloworld`
Hello, world!
```

这里我们使用cargo创建了一个新的项目，而项目中存放的crate根文件就是src/main.rs,里面默认存放了一段”hello world!”代码。

这里我不会多赘述有关rust的Package和Crate相关的知识，你只需要知道我们在用”Cargo new”命令时，创建了一个”hello world!”程序，然后我们使用cargo run来运行它。

在我们使用”cargo run”这个命令时，会进行以下的几个步骤：

程序会先检查Rust代码是否有错误，没有错误的话就会将其编译成可执行文件；在编译的过程中，编译器会将其转换为机器码，然后计算机运行该文件，输出了”hello world!”。

我上面说的这些东西相信大家都明白，然而在我们的在执行这个可执行文件的时候，系统内发生了几次调用，我们可以用命令“strace target/debug/helloworld”来跟踪Linux的系统调用和信号。

``` bash
//
execve("target/debug/helloworld", ["target/debug/helloworld"], 0x7ffd9b8b5c50 /* 62 vars */) = 0
//返回heap当前的位置
brk(NULL)                               = 0x55e0e9056000        
//将程序可执行文件映射到内存中
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fba85389000  
//判断文件权限 
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
//输出字符串
write(1, "Hello, world!\n", 14Hello, world!             
)         = 14
//退出程序
exit_group(0)                           = ?
```

## 现代操作系统

现代的大部分操作系统有两大特征：并发控制和资源共享。

在上文中，在我们输入完那条命令后会出现很长一串系统调用，我这里删掉了大部分的系统调用过程，但我保留了一部分。给大家看看系统调用的大概过程。实际上，我们不难看出，这里真正在执行“Hello World!”程序的系统调用就是后两行，但这个程序执行过程中出现了大量的系统调用信息，我们从上面这个例子可以看出，现代的大部分操作系统加入了大量的功能，当我们执行像“Hello world！”这种简单的程序时，它也会实现很多多余的功能。

## 多余的系统调用做了些什么

它们基本上用于对Standard libray和Kernel环境的初始化和对上层应用进行监控和控制，比如我们平时边听音乐边码文章，这一刻有多个程序在我们的操作系统上运行，我们的操作系统需要有强大的硬件资源管理能力，实现一些系统调用，才能让我们在复杂的应用环境下稳定。

# 应用程序执行环境的构造

![应用程序执行环境栈](/SU15VTE/ca852c4f/stack.PNG)\
上面这张应用程序执行环境的Stack图能让我们更直观地理解应用程序执行环境。其中的Instruction set和system call，function call为接口，拿“Hello World!”程序程序来讲，我们在rust上实现它需要函数调用”println！()”，而这个宏正是Rust的标准库std提供的，而标准库的各种功能总是需要我们操作系统内核提供系统调用来实现。而操作系统内核又需要带有相应指令集的硬件来支持，至此，现代操作系统的应用程序就组成了。

## 多层执行环境

现代的操作系统提供了多层执行环境，但其实除Application和HardWare，中间的所有组件都是可有可无的，他们不过是对下层进行抽象，上层提供环境。

## 需要多抽象？

### 编程语言发展史

我们最早期的计算机，是使用打孔的纸条来执行某个程序的，那时使用的还是机器语言，机器语言使用计算机硬件所支持的二进制指令组成，当时的编程极其繁琐；而后面又出现了汇编语言，汇编语言就是机器语言的一种抽象(abstraction)，它使用了助记符来代替了二进制指令，使得程序员跟容易编写和维护程序；后来又出现了高级语言，它具有更高的抽象，可以让我们更快地编写程序，并且拥有更好的维护性。编程语言的发展始终是以更高的抽象和更高效的方式来实现程序的组织和设计为目标。

## **应用程序的多层执行环境** 需要什么程度的抽象

抽象的优点在于它能让上层的以较小的代价来获得自己所需要的功能，并且还可以提供一定的保护，但并不是越抽象越好的，过度的抽象会丧失一定的灵活性，反而成为一种限制。因此，应用程序的执行环境需要抽象到什么程度，我们需要根据应用程序所需要的功能来考虑，这也是操作系统设计者需要考虑的一大问题。

### 没有函数库和操作系统内核的应用程序执行环境

当我们失去了函数库和操作系统内核时，我们就基本上只能用编写汇编代码这种方式来控制我们的硬件组件，抽象能力最低，但是灵活性极高，而在这种应用程序执行环境下，这种环境下我们一般会编写一些与架构有关的，而且用高级的编程语言是无法实现的模块或代码段。

### 有函数库没有操作系统内核的应用程序执行环境

如果我们只有函数库而没有操作系统内核，那么就意味着我们不用操作系统内核来提供过于通用的抽象，做过嵌入式开发(embedded development)的同学应该知道，一个嵌入式设备虽五脏俱全，有着CPU，存储器和各种I/O设备，但是我们通常只要他实现几种非常简单的功能。为实现某个功能，我们一般会使用函数库构建一个单独的应用程序，这就只需要用到只带函数库的执行环境就可以了，如果带上操作系统内核的话，就会像我们前文写”Hello World!”程序一样，多出很多无用的系统调用。

### 既有函数库又有操作系统内核的应用程序执行环境

带有这种应用程序执行环境的操作系统，相信大家都用过。没错，我们现在最常用的Linux和Windows系统，都是同时存在函数库和操作系统内核的，因为我们的应用场景通常会比较复杂，因此这类操作系统为了保证自己的通用性，需要更为强大的抽象和功能，需要多层执行环境。

# 目标平台

## Rust可支持的目标平台

### 上文”Hello World!”的目标平台

我们在之前的”Hello World!”程序中可以看到我们的目标平台是什么

``` bash
SU15VTE@SU15VTE:~/Desktop/code/helloworld$ rustc --version --verbose
rustc 1.65.0
binary: rustc
commit-hash: unknown
commit-date: unknown
host: x86_64-unknown-linux-gnu
release: 1.65.0
LLVM version: 14.0.0
```

从输出的信息我们可以看出我们的目标平台是**x86_64-unknown-linux-gnu**，我们可以看到我们的CPU架构是x86_64，CPU厂商未知，OS是Linux，运行的库是GNU lib。

### Rust支持的RISC-V的平台

我们文章开头已经说过了，我们会使用Rust和ricev64gc平台来作为例子。而Rust支持的ricev64gc平台有

``` bash
SU15VTE@SU15VTE:~/Desktop/code/helloworld$ rustc --print target-list | grep riscv64gc
riscv64gc-unknown-freebsd
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-none-elf
riscv64gc-unknown-openbsd
```

这里我们选择**riscv64gc-unknown-none-elf**作为我们的目标平台。接下来我们将会在这个**裸机平台 (bare-metal)**编写一个”Hello World!”环境。如上文所说的，当我们要实现一个简单的应用时，我们不需要用到**操作系统内核**提供过于通用的抽象。我们还可以看到上面的平台中没有带函数库而不带操作系统内核的平台，但是问题不大，我们要实现一个”Hello World!”程序，只需要我们实现一个\*\*println!\*\*的宏就可以了。

## Rust核心库

由于我们所在的平台是**裸机平台**，所以我们无法使用Rust完整的标准库std。但是Rust有一个被阉割过的[Rust核心库core](https://doc.rust-lang.org/core/)，我们可以使用Core库来编写我们的”Hello World!”程序，但是在完成这个程序前，我们还需要做一些事。那么在下一篇文章中，我们将正式开始编写这个”Hello World!”程序。

# 杂谈

本文已经为大家介绍了应用程序的执行环境，这是我第一次写这种文章，我尽可能以通俗易懂的方式向大家解释操作系统中应用程序执行环境，但表述方面可能还是有些问题，如果有错误的话请指正。
