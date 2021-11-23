---
layout: post
title: "__x86.get_pc_thunk.ax函数"
date: "2019-12-03"
categories: ["Computer"]
tags: ["__x86.get_pc_thunk", "x86", "asm"]
---

在Linux下反汇编32位的程序时，经常会遇到`__x86.get_pc_thunk.ax`函数。

例如在32位的Hello world程序中，`.text`段偏移`0x52c`处，调用了该函数：

```
0000051d <main>:
 51d:	8d 4c 24 04          	lea    0x4(%esp),%ecx
 521:	83 e4 f0             	and    $0xfffffff0,%esp
 524:	ff 71 fc             	pushl  -0x4(%ecx)
 527:	55                   	push   %ebp
 528:	89 e5                	mov    %esp,%ebp
 52a:	53                   	push   %ebx
 52b:	51                   	push   %ecx
 52c:	e8 28 00 00 00       	call   559 <__x86.get_pc_thunk.ax>
 531:	05 a7 1a 00 00       	add    $0x1aa7,%eax
 536:	83 ec 0c             	sub    $0xc,%esp
 539:	8d 90 08 e6 ff ff    	lea    -0x19f8(%eax),%edx
 53f:	52                   	push   %edx
 540:	89 c3                	mov    %eax,%ebx
 542:	e8 69 fe ff ff       	call   3b0 <puts@plt>
 547:	83 c4 10             	add    $0x10,%esp
 54a:	b8 00 00 00 00       	mov    $0x0,%eax
 54f:	8d 65 f8             	lea    -0x8(%ebp),%esp
 552:	59                   	pop    %ecx
 553:	5b                   	pop    %ebx
 554:	5d                   	pop    %ebp
 555:	8d 61 fc             	lea    -0x4(%ecx),%esp
 558:	c3                   	ret    
```

查看函数，内容如下（`xchg %ax,%ax`命令交换`%ax`与`%ax`，相当于`nop`）：

```
00000559 <__x86.get_pc_thunk.ax>:
 559:	8b 04 24             	mov    (%esp),%eax
 55c:	c3                   	ret    
 55d:	66 90                	xchg   %ax,%ax
 55f:	90                   	nop
```

简单来说，该函数的功能是将`%eip`寄存器内容传入`%eax`寄存器。相当于`mov %eip, %eax`。

这个函数在x86上的PIC(position independent code，即位置无关代码)中使用。它将`%eip`的位置加载到`%eax`寄存器中，从而实现对模块内部数据（例如全局变量）的访问。原因是x86的指令集中没有直接读取`%eip`的指令。

类似的函数还有：`__x86.get_pc_thunk.bx`，`__x86.get_pc_thunk.cx`， `__x86.get_pc_thunk.dx`。功能是类似的，只不过传入的寄存器分别为`%ebx`， `%ecx`， `%edx`。早期版本的编译器中，这个函数叫`__i686.get_pc_thunk.ax`。

在64位程序中不需要这个函数，因为`x86-64`指令集中可以通过`lea`指令直接获取`%rip`。
