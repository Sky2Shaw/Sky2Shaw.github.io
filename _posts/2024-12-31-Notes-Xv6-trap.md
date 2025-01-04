---
title: xv6笔记：Trap
description: notes on trap in xv6
author: sky2shaw
date: 2024-12-31 21:22:00 +0800
categories: [Blogging]
tags: [os]
pin: true
math: trues
mermaid: true
---

## 前言
接近年末，手头的项目都已收尾，闲来无事决定研究下操作系统。xv6是个很好的研究对象。于是一边结合文档看源码一边写lab。回来有空坐在飘窗上写写笔记ß，老婆带着熊娃在床上睡觉。于是有了以下内容。

## 系统调用
在学习xv6之前，系统调用在我的印象中就是操作系统开放给用户的一些列诸如open、fork之类的函数，并没有很深入地去了解其原理。
在进一步讨论之前，我们先回顾下什么是系统调用。AI告诉我们：系统调用是用户程序与操作系统内核之间的接口，用于请求操作系统执行某些特权操作或访问受保护的资源。基础的图像是这样的：
![system call base picture](/assets/img/xv6.001.png)

这个定义些微有点抽象，我们不妨直接到xv6的源码里面看下，系统调用到底是怎么实现的吧。

首先我们来看下xv6是如何通过exec系统调用拉起第一个用户态进程的：

``` Assembly
# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall
```
首先la a0, init将可执行文件的名称加载到a0寄存器，la a1, argv将参数加载到a1寄存器，li a7, SYS_exec将系统调用号加载到a7寄存器。所有相关寄存器都设置完毕后，执行了ecall指令。

在学习ecall指令时，我们需要先了解risc-v CPU的三种模式以及相关的控制状态寄存器（Control and status register， CSR）。
三种模式：

1. 用户模式
2. 监督模式
3. 机器模式

这三种模式的主要区别是能够执行的指令集不同。
用户模式的特权级别最低，只能执行非特权指令，只能访问自己的虚拟内存（关于虚拟内存，需要单独开一篇博客来讲），主要用于运行用户进程；
监督模式的特权级别相比用户模式要高，能够执行一些特权指令，管理虚拟内存等，但仍然存在一定的限制。监督模式用于执行操作系统内核。
机器模式拥有最高的特权级别，能够执行所有指令，一般用于引导加载程序。这里我们主要关注用户模式和监督模式。

了解了risc-v cpu的三种模式后，我们再回到ecall指令。当cpu处于用户模式时，执行ecall指令，cpu会切换到监督模式，同时会更新几个相关的CSR寄存器。CSR寄存器即控制状态寄存器，是一组特殊的寄存器，用于配置和监控系统硬件资源。和普通寄存器不同，CSR寄存器一般只能在监督模式和机器模式下访问。ecall指令会修改sstatus、sepc、scause寄存器，这些寄存器开头的s代表监督模式（supervisor）。sepc存储当前的pc，用于cpu模式切回用户模式时继续从当前pc的下一条指令执行；scause用于存储cpu模式切换的原因，例如系统调用需要将scause设置为0x8,这样cpu模式切换为监督模式后，cpu才能知道发生切换的原因，并执行相应的操作。sstatus寄存器稍微复杂一点，它的SIE（Supervisor Interrupt Enable）位会被清0，用于关闭中断；它的SPP（Supervisor Previous mode）位会被置为0（当前的用户模式），用于表示cpu模式在切换前处于用户模式。最后，ecall指令会将sstvec中存储的trap handle的地址赋给pc，cpu开始执行trap handle的代码。




