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

接下来我们看下trap handle的代码：
``` Assembly
.globl uservec
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #

        # save user a0 in sscratch so
        # a0 can be used to get at TRAPFRAME.
        csrw sscratch, a0

        # each process has a separate p->trapframe memory area,
        # but it's mapped to the same virtual address
        # (TRAPFRAME) in every process's user page table.
        li a0, TRAPFRAME
        
        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # initialize kernel stack pointer, from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), from p->trapframe->kernel_trap
        ld t0, 16(a0)

        # fetch the kernel page table address, from p->trapframe->kernel_satp.
        ld t1, 0(a0)

        # wait for any previous memory operations to complete, so that
        # they use the user page table.
        sfence.vma zero, zero

        # install the kernel page table.
        csrw satp, t1

        # flush now-stale user entries from the TLB.
        sfence.vma zero, zero

        # jump to usertrap(), which does not return
        jr t0
```

这段代码主要的逻辑是将当前的所有用户寄存器保存到TRAPFRAME中，TRAPFRAME是内存中的一块区域。然后将TRAPFRAME中保存的内核栈指针、硬件线程（hartid）、usertrap地址、内核页表分别加载到寄存器（分别为sp、tp、t0、t1）中，然后通过“sfence.vma zero, zero”强制完成所有虚拟内存地址转化，再通过“csrw satp, t1”切换页表，紧接着又执行了“sfence.vma zero, zero”，最后跳转到usertrap。概括下，就是保存用户寄存器到内存的某个区域，再切换内核页表，最终跳转到usertrap。

注意到这段代码有个切换页表的操作，按道理切换页表前后，虚拟内存会被映射到不同的物理内存，这也就意味着“csrw satp, t1”之后的几条指令都会被映射到其他的物理内存，这岂不是一个很严重的问题？取其他物理内存的指令会不会导致接下来运行的不是“sfence.vma zero, zero”和“jr t0”两条指令？答案是否定的，xv6采取了一个非常巧妙的设计，上述的这段代码在用户空间和内核空间被映射到了同一块物理内存，所以在页表切换前后并不会导致物理内存不一致。

接下来我们继续跟踪usertrap函数。幸运的是，这个函数是用C码写的：
``` c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```