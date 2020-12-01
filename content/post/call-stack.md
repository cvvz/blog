---
title: "用树莓派分析函数调用栈"
date: 2019-09-03T18:44:12+08:00
draft: false
comments: true
keywords: ["call stack", "调用栈", "树莓派"]
tags: ["Linux"]
---
> 理解本篇文章需要具备一些GDB、汇编、寄存器的基础知识。可以在阅读的过程中碰到不理解的地方再针对性的学习。

## 寄存器

分析函数调用栈涉及到的几个特殊用途的寄存器如下：

| ARM     | X86     | 用途    |
| :--:    | :--:    | :--:   |
| r11（fp） | rbp（ebp） | 栈帧指针 |
| r13（sp） | rsp（esp） | 栈顶指针 |
| r14（lr） | N/A     | 返回地址 |
| r15（pc） | rip     | 指令指针（程序计数器） |

## 函数调用栈

如下图（《程序员的自我修养》图10-4）所示：

{{< figure src="/栈.jpg" width="500px" >}}

图中，栈帧指针（ebp）指向的内存空间中保存的是上一个栈的栈帧指针（old ebp）。这是X86的情形，在树莓派中分析函数调用栈时发现，ARM的栈帧指针（fp）指向的是函数返回地址。

这只是不同架构CPU的底层实现的不同，并没有优劣之分。

### 入栈过程

一个函数的调用过程可以分为如下几步：

* 首先压栈的是参数，且**从右向左**依次压栈；
* 接着压入返回地址；
* 接着被调函数执行“标准开头”（x86）：

```x86asm
push rbp
mov rbp rsp
```

“标准开头”执行过程如下：

* 首先rbp入栈；
* rbp入栈后，rsp自动加8（64位），rsp此时指向存放rbp的栈帧地址；
* 接着令`%rbp=%rsp`，这就使得rbp指向存放着上一个栈的rbp的内存地址。

而ARM（32位）的“标准开头”长这样：

```armasm
push {fp, lr}
add fp, sp, #4
```

* 返回地址(lr)入栈
* 栈帧指针(fp)入栈
* 接着令`%fp=%sp+4`，也就是**使fp（栈帧指针）指向存放返回地址的内存**。

不论栈帧指针指向的是上一个栈帧指针，还是返回地址，都能**通过函数的栈帧指针偏移找到调用函数的地址，因此根据栈帧指针的链式关系，可以回溯出整个函数的调用关系链**。这对于一些复杂问题的定位是非常有帮助的。

> GCC的编译选项`--fomit-frame-pointer`可以使程序不使用栈帧指针，而使用栈指针顶定位函数的局部变量、参数、返回地址等。这么做的好处是可以多出一个寄存器（栈帧指针）供使用，程序运行速度更快，但是就没发很方便的使用GDB进行调试了。

### 出栈过程

出栈与入栈动作刚好相反。

x86的“标准结尾”如下：

```x86asm
leaveq
retq
```

实际上`leaveq`内部分为两条指令：

```x86asm
movq %rbp, %rsp
popq %rbp
```

所以，出栈过程可以分解为如下三步：

* 第一步是通过将rbp地址赋给rsp，即此时rsp指向的内存存放的是上一个栈的rbp。
* 第二步弹出栈顶的数据到rbp中，即rbp指向上一个栈的栈底，出栈动作导致rsp自增，于是rsp此时指向的内存中存放函数返回地址；
* 第三步通过`retq`指令将栈顶地址pop到rip，即rip此时指向函数退出后的下一条指令，rsp则指向上一个栈的栈顶。

这三步做完后，rsp、rbp、rip就恢复到调用函数以前的现场。

ARM的行为和x86一致，它的“标准结尾”长这样：

```armasm
sub sp, fp, #4
pop {fp, pc}
```

## 基于树莓派3分析函数调用栈

我在树莓派3中运行了如下所示的C语言代码，并用GDB进行了调试：

> 树莓派3使用的是**32位、arm架构CPU**，因此下面的调试过程涉及到的寄存器以及地址信息和64位x86 CPU不同

```C
#include <stdio.h>

void test2(int i)
{
    int ii;
    ii = i;
}

char test(char c)
{
    int i;
    printf("%c",c);
    test2(i);
    return c;
}

int main()
{
    char c = 'a';
    char ret;
    ret = test(c);
    return 0;
}
```

### 分析函数调用（入栈）过程

使用GDB进行调试，将断点打在main函数调用test之前，并使用`disassemble`查看反汇编结果：

```armasm
(gdb) b *0x000104bc
Breakpoint 2 at 0x104bc: file main.c, line 21.
(gdb) disassemble /m main
Dump of assembler code for function main:
18 {
   0x000104a0 <+0>: push {r11, lr}
   0x000104a4 <+4>: add r11, sp, #4
   0x000104a8 <+8>: sub sp, sp, #8

19 char c = 'a';
   0x000104ac <+12>: mov r3, #97 ; 0x61
   0x000104b0 <+16>: strb r3, [r11, #-5]

20 char ret;
21 ret = test(c);
   0x000104b4 <+20>: ldrb r3, [r11, #-5]
   0x000104b8 <+24>: mov r0, r3
=> 0x000104bc <+28>: bl 0x10468 <test>
   0x000104c0 <+32>: mov r3, r0
   0x000104c4 <+36>: strb r3, [r11, #-6]

22 return 0;
   0x000104c8 <+40>: mov r3, #0

23 }
   0x000104cc <+44>: mov r0, r3
   0x000104d0 <+48>: sub sp, r11, #4
   0x000104d4 <+52>: pop {r11, pc}

End of assembler dump.
```

查看此时栈帧指针和栈顶指针的值：

```armasm
(gdb) i r r11 sp
r11            0x7efffaec 2130705132
sp             0x7efffae0 0x7efffae0
(gdb) x /xw 0x7efffaec
0x7efffaec: 0x76e8f678
(gdb) info symbol 0x76e8f678
__libc_start_main + 276 in section .text of /lib/arm-linux-gnueabihf/libc.so.6
```

可以看到，栈帧指针指向的返回地址是`__libc_start_main + 276`，即**main函数是由__libc_start_main调用的**。

由前面分析得知，栈帧指针-4地址处存放的是上一个函数的栈帧指针，于是我们继续向上追溯`__libc_start_main`的调用者地址，可以发现其值为0：

```armasm
(gdb) x /xw 0x7efffaec-4
0x7efffae8: 0x00000000
```

**因此可以认为`__libc_start_main`是所有进程真正的起点。**

接着执行调用test函数的命令，使用`si`单步运行，并查看汇编指令：

```armasm
(gdb) si
test (c=0 '\000') at main.c:10
10 {
(gdb) disassemble
Dump of assembler code for function test:
=> 0x00010468 <+0>: push {r11, lr}
   0x0001046c <+4>: add r11, sp, #4
   0x00010470 <+8>: sub sp, sp, #16
   0x00010474 <+12>: mov r3, r0
   0x00010478 <+16>: strb r3, [r11, #-13]
   0x0001047c <+20>: ldrb r3, [r11, #-13]
   0x00010480 <+24>: mov r0, r3
   0x00010484 <+28>: bl 0x10300 <putchar@plt>
   0x00010488 <+32>: ldr r0, [r11, #-8]
   0x0001048c <+36>: bl 0x10440 <test2>
   0x00010490 <+40>: ldrb r3, [r11, #-13]
   0x00010494 <+44>: mov r0, r3
   0x00010498 <+48>: sub sp, r11, #4
   0x0001049c <+52>: pop {r11, pc}
End of assembler dump.
(gdb) i r $lr
lr             0x104c0 66752
(gdb) info symbol $lr
main + 32 in section .text of /root/main
```

可以看到此时lr寄存器中保存的指令即调用test后的下一条指令。继续向下执行：

```armasm
(gdb) ni
0x0001046c 10 {
(gdb) i r r11 sp
r11            0x7efffaec 2130705132
sp             0x7efffad8 0x7efffad8
```

观察到将r11和lr入栈后，sp减少了8字节，不难猜测，高4字节存放了lr的值（返回地址），低4字节存放了sp的值（上一个栈的栈帧指针）：

```armasm
(gdb) x /xw 0x7efffad8
0x7efffad8: 0x7efffaec
(gdb) x /xw 0x7efffadc
0x7efffadc: 0x000104c0
(gdb) i r $lr $r11
lr             0x104c0 66752
r11            0x7efffaec 2130705132
```

继续执行：

```armasm
(gdb) ni
0x00010470 10 {
(gdb) i r $r11
r11            0x7efffadc 2130705116
```

此时r11指向的是函数返回地址，而不是像x86一样指向上一个栈帧指针，和前面所说的一致。

## 分析函数返回（出栈）过程

test函数的汇编指令如下所示：

```armasm
(gdb) disassemble /m test
Dump of assembler code for function test:
10 {
   0x00010468 <+0>:	push	{r11, lr}
   0x0001046c <+4>:	add	r11, sp, #4
   0x00010470 <+8>:	sub	sp, sp, #16
   0x00010474 <+12>:	mov	r3, r0
   0x00010478 <+16>:	strb	r3, [r11, #-13]

11		int i;
12		printf("%c",c);
   0x0001047c <+20>:	ldrb	r3, [r11, #-13]
   0x00010480 <+24>:	mov	r0, r3
   0x00010484 <+28>:	bl	0x10300 <putchar@plt>

13		test2(i);
   0x00010488 <+32>:	ldr	r0, [r11, #-8]
   0x0001048c <+36>:	bl	0x10440 <test2>

14		return c;
   0x00010490 <+40>:	ldrb	r3, [r11, #-13]

15	}
   0x00010494 <+44>:	mov	r0, r3
=> 0x00010498 <+48>:	sub	sp, r11, #4
   0x0001049c <+52>:	pop	{r11, pc}

End of assembler dump.
```

函数运行完毕进入出栈流程的执行过程分为如下几步：

* 首先通过 `sub sp, r11, #4` 将栈顶指针指向上一个栈帧指针
* 接着通过 `pop {r11, pc}` 将上一个栈帧指针赋值给r11，并将返回地址赋值给pc
* 两次pop后，栈顶指针自动往栈底方向退两次

最终，栈顶指针（sp）、栈帧指针（r11）和指令指针（pc）都还原成了main函数调用test前的样子，用GDB查看寄存器内容证实了这一点：

```armasm
(gdb) disassemble 
Dump of assembler code for function main:
   0x000104a0 <+0>:	push	{r11, lr}
   0x000104a4 <+4>:	add	r11, sp, #4
   0x000104a8 <+8>:	sub	sp, sp, #8
   0x000104ac <+12>:	mov	r3, #97	; 0x61
   0x000104b0 <+16>:	strb	r3, [r11, #-5]
   0x000104b4 <+20>:	ldrb	r3, [r11, #-5]
   0x000104b8 <+24>:	mov	r0, r3
   0x000104bc <+28>:	bl	0x10468 <test>
=> 0x000104c0 <+32>:	mov	r3, r0
   0x000104c4 <+36>:	strb	r3, [r11, #-6]
   0x000104c8 <+40>:	mov	r3, #0
   0x000104cc <+44>:	mov	r0, r3
   0x000104d0 <+48>:	sub	sp, r11, #4
   0x000104d4 <+52>:	pop	{r11, pc}
End of assembler dump.
(gdb) i r r11 sp pc
r11            0x7efffaec	2130705132
sp             0x7efffae0	0x7efffae0
pc             0x104c0	0x104c0 <main+32>
```
