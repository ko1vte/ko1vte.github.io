---
title: ichigo ichie-第二期-rev-thanks4allthefish
date: 2023-06-06 01:23:33
tags: ["CTF", "reverse", "SpaceHeroesCTF 2023"]
cover: img/daliy.jpg
permalink: p/ichigo-ichie-ep2-rev-thanks4allthefish/
alias:
  - SU15VTE/f27987f9.html
---
# 题目介绍

来自5月份的一场CTF比赛，这次报了两场，~~第一晚另外一场在坐牢就跑来打SpaceHeroesCTF了~~先上本次的战绩吧，逆向题其他的感觉没什么好讲的，来说说这道比较有意思的题。\
![png](/SU15VTE/f27987f9/%E6%8E%92%E5%90%8D.PNG)\
![png](/SU15VTE/f27987f9/%E5%88%86%E6%95%B0.PNG)

# 反转啦！—逆向分析

拿到一个文件，第一时间要做的就是对这个文件进行分析，看看它是32位的程序还是64位的程序吧。

``` bash
$ file thanks4allthefish 
thanks4allthefish: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2e14332fc6eef51cfee511f46848f5ec23b02efc, for GNU/Linux 4.4.0, not stripped
```

好吧，这是一个x86-64的程序，我们直接上IDA反汇编出它。来到main函数

``` c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int result; // eax
  int i; // [rsp+2Ch] [rbp-84h]
  unsigned int v6; // [rsp+38h] [rbp-78h]
  FILE *stream; // [rsp+58h] [rbp-58h]
  char s[32]; // [rsp+60h] [rbp-50h] BYREF
  char s2[40]; // [rsp+80h] [rbp-30h] BYREF
  unsigned __int64 v10; // [rsp+A8h] [rbp-8h]

  v10 = __readfsqword(0x28u);
  megaInit();
  global_argc = argc;
  global_argv = (__int64)argv;
  global_envp = (__int64)envp;
  print_art();
  puts(
    "Welcome, human, to the 42nd centennial dolphin acrobatics show!  Better get to it.\n"
    "These dolphins aren't going to train themselves...");
  v6 = getppid();
  snprintf(s, 0x14uLL, "/proc/%d/comm", v6);
  stream = fopen(s, "r");
  fgets(s2, 32, stream);
  for ( i = 0; i <= 31; ++i )
  {
    if ( s2[i] == 10 )
    {
      s2[i] = 0;
      break;
    }
  }
  if ( !strncmp("fish", s2, 4uLL) )
  {
    puts("\nThe dolphins aren't in the mood for fish right now.");
    result = 0;
  }
  else if ( !strncmp("bash", s2, 4uLL) )
  {
    puts("\nThe dolphins don't appreciate your threats of violence.");
    result = 0;
  }
  else if ( !strncmp("tidbits", s2, 7uLL) )
  {
    puts("\nUpon seeing the tidbits, the dolphins begin their performance.");
    puts(
      "\n"
      "As you give them the signal, you are amazed by the dolphins' uncanny ability to\n"
      "perform a double-backwards-somersault through a hoop whilst whistling \"The Star\n"
      "Spangled Banner.\"  You can't help but wonder if there's some hidden meaning behind\n"
      "their actions.");
    if ( !(unsigned int)tricks() )
      sub_21960();
    result = 0;
  }
  else
  {
    puts("\nThe dolphins are hungry...");
    result = 0;
  }
  return result;
}
```

看起来代码很长，让我们来分段看看它做了些什么吧。

``` c
v6 = getppid();
  snprintf(s, 0x14uLL, "/proc/%d/comm", v6);
  stream = fopen(s, "r");
  fgets(s2, 32, stream);
  for ( i = 0; i <= 31; ++i )
  {
    if ( s2[i] == 10 )
    {
      s2[i] = 0;
      break;
    }
  }
```

首先，程序获取了自己运行的pid，然后把pid拼接到 **/proc/%d/comm** 中，将其赋值给s，此时的s是 **/proc/\$\[pid\]/comm**，然后我们再用fopen的方法将s打开，读取s中的值，也就是 **/proc/\$\[pid\]/comm** 的值，这里来解释下这个 **/proc** 文件夹是个什么东西。

> **Linux文件系统层次结构—proc文件夹**\
> /proc 是一个Linux的虚拟文件系统，它有时被称为进程信息伪文件系统。它不包含“真实”文件，而是包含运行时系统信息（例如系统内存、安装的设备、硬件配置等）。因此，它可以被视为内核的控制和信息中心。事实上，相当多的系统实用程序只是调用这个目录中的文件。例如，“lsmod”与“cat /proc/modules”相同，而“lspci”是“cat /proc/pci”的同义词。通过更改位于此目录中的文件，您甚至可以在系统运行时读取/更改内核参数 (sysctl)。

而这里的 **/proc/\$\[pid\]/comm** 里面的内容是进程的命令名，我们在Linux中使用ps命令就可以查看到当前进程的进程的命令名：

``` bash
$ ps                       
    PID TTY          TIME CMD
  76302 pts/1    00:00:00 zsh
  76326 pts/1    00:00:00 python
  86306 pts/1    00:00:00 gdb
 102759 pts/1    00:00:00 ps
```

此时的 **/proc/\$\[pid\]/comm** 的内容被读取到了s2中。

``` bash
if ( !strncmp("fish", s2, 4uLL) )
 {
   puts("\nThe dolphins aren't in the mood for fish right now.");
   result = 0;
 }
 else if ( !strncmp("bash", s2, 4uLL) )
 {
   puts("\nThe dolphins don't appreciate your threats of violence.");
   result = 0;
 }
 else if ( !strncmp("tidbits", s2, 7uLL) )
 {
   puts("\nUpon seeing the tidbits, the dolphins begin their performance.");
   puts(
     "\n"
     "As you give them the signal, you are amazed by the dolphins' uncanny ability to\n"
     "perform a double-backwards-somersault through a hoop whilst whistling \"The Star\n"
     "Spangled Banner.\"  You can't help but wonder if there's some hidden meaning behind\n"
     "their actions.");
```

之后我们会将s2和fish，bash，tidbits进行对比，如果你的comm不是tidbits，程序就会终止，这一步其实我们可以利用gdb绕过它，\
![png](/SU15VTE/f27987f9/comm.PNG)\
![png](/SU15VTE/f27987f9/RealPass.PNG)\
绕过后，我们就可以进入到下一个阶段了。

``` bash
if ( !(unsigned int)tricks() )
      sub_21960();
    result = 0;
  }
  else
  {
    puts("\nThe dolphins are hungry...");
    result = 0;
  }
  return result;
```

这里的sub_21960()就是输出加密过的flag，让我们看看tricks()

``` bash
__int64 tricks()
{
  int i; // [rsp+4h] [rbp-42Ch]
  FILE *stream; // [rsp+10h] [rbp-420h]
  char s[1032]; // [rsp+20h] [rbp-410h] BYREF
  unsigned __int64 v4; // [rsp+428h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  stream = popen("/bin/grep tidbits /proc/*/comm", "r");
  if ( !stream )
  {
    puts("ERROR: This challenge depends on grep.");
    exit(1);
  }
  putchar(10);
  for ( i = 0; fgets(s, 1024, stream) && i <= 5; ++i )
    printf("Performed trick %c...\n", (unsigned int)(i + 65));
  if ( i <= 5 )
  {
    if ( i > 4 )
      return 0LL;
    puts("\nYou ran out of treats.  The dolphins are no longer following your lead.");
  }
  else
  {
    puts("\nYou overfed the dolphins and they decided to take a nap.");
  }
  return 1LL;
}
```

先用grep获取所有进程的命令名为 **tidbits** 的进程。\
`stream = popen("/bin/grep tidbits /proc/*/comm", "r");`\
如果tidbits大于4的话，就会返回0，也就是 `if ( !(unsigned int)tricks() )`成立，这里我们如果继续用gdb继续调试下去的话，程序也会终止。\
![png](/SU15VTE/f27987f9/error.PNG)

# 解题过程

现在我们知道我们需要用tidbits这个进程命令名运行程序，并且系统名为tidbits的进程命令名的进程要有5个，才能得到flag，那么我们有没有办法让进程使用tidbits这个进程命令名呢？实际上我们只需要将/usr/bin/sh这个可执行文件给cp下来，然后改成你想要的进程命令名就可以了。\
![png](/SU15VTE/f27987f9/sh.PNG)\
然后多开几个shell，打开改名为tidbits的sh。\
![png](/SU15VTE/f27987f9/finish.PNG)\
开启5个tidbits后用其中一个打开我们的程序。获取到flag。\
![png](/SU15VTE/f27987f9/flag.PNG)

**flag:shctf{0k_but_h4v3_y0u_s33n_th3_m1c3}**
