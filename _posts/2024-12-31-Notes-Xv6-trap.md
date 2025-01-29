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
接近年末，手头的项目都已收尾，闲来无事决定研究下操作系统。xv6是个很好的研究对象。于是一边结合文档看源码一边写lab。回来有空坐在飘窗上写写笔记，老婆带着熊娃在床上睡觉。于是有了以下内容。

## 系统调用
在学习xv6之前，系统调用在我的印象中就是操作系统开放给用户的一些列诸如open、fork之类的函数，并没有很深入地去了解其原理。下面我们借助xv6来深入研究一下系统调用。
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
首先la a0, init将可执行文件的名称加载到a0寄存器，la a1, argv将参数加载到a1寄存器，li a7, SYS_exec将系统调用号加载到a7寄存器。所有相关寄存器都设置完毕后，执行了ecall指令，于是就发生了所谓的系统调用，也就是说：在risc-v架构中，系统调用是通过ecall来实现的。

ecall指令会切换cpu模式并修改一些寄存器，然后设置pc（program counter，程序计数器，cpu从程序计数器指向的内存取指令并执行），让cpu跳转到指定的pc处继续执行。下面我们来深入研究下这个ecall指令：

在学习ecall指令之前，我们需要先了解risc-v CPU的三种模式以及相关的控制状态寄存器（Control and status register， CSR）。
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

这段代码首先检查了sstatus寄存器的SPP位（该位标志了发生cpu模式切换前cpu所处的模式），如果是内核模式，会panic。回顾调用ecall指令时已经将SPP位置为0，所以这边会接着往下执行。接着将stvec寄存器保存的trap handle切换为kernelvec，这样发生中断时会跳转到kernelvec。接着读取sepc寄存器，并保存到trapframe的epc字段（sepc记录的是发生系统调用或trap时的pc）。接下来根据scause寄存器，即发生trap的原因，回顾调用ecall时已经将scause设置为0x8即系统调用，所以这里会走系统调用分支，其他分支我暂时也没有梳理清楚，所以这里只关注处理系统调用的分支：

首先将trapframe中的epc字段加上4，这是为了当完成系统调用回到用户态时，在pc + 4的位置继续执行。接着开启中断，调用syscall函数。现在才开中断的原因，是为了保证sepc、scause、stvec、sstatus等寄存器的值都已经被正常获取或设置。一旦中断打开，发生trap时，这些寄存器都有可能被修改。

我们看一下关键函数syscall到底做了什么：
``` c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
好家伙，竟然如此简单：取trapframe中的a7字段，即系统调用号，然后执行syscalls数组中的对应syscall。回想下，我们在ecall时已经将系统调用号设置到a7寄存器，又在trap handle（uservec）中将a7保存到trapframe中。系统调用号就是这样从用户空间传递到了内核空间。关于什么是trapframe，后面会追加一段内容详细分析。                   

接下来我们跟着函数流程，继续研究usertrapret函数：
``` c
//
// return to user space
//
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to uservec in trampoline.S
  uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
  w_stvec(trampoline_uservec);

  // set up trapframe values that uservec will need when
  // the process next traps into the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to userret in trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64))trampoline_userret)(satp);
}
```

这个函数的功能如其名称所示，是用于返回到用户空间。因为在前面的进入内核空间时(usertrap)已经将trap的handle(保存于stvec寄存器中)设置为kernelvec，所以在返回用户空前前需要将trap的handle切回usertvec。接着将内核页表、内核栈、内核trap函数、硬件线程保存到trapframe中。然后设置sstatus寄存器，让cpu知道是由监督模式切换到用户模式的。接着开启中断，设置返回用户空间后的指令寄存器，计算用户空间页表，最后执行trampoline.S中的userret函数，并将用户空间页表作为参数传入。

最后我们看一下trampoline.S中的userret函数：
```
userret:
        # userret(pagetable)
        # called by usertrapret() in trap.c to
        # switch from kernel to user.
        # a0: user page table, for satp.

        # switch to the user page table.
        sfence.vma zero, zero
        csrw satp, a0
        sfence.vma zero, zero

        li a0, TRAPFRAME

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0
        ld a0, 112(a0)
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```
这个函数首先切换用户态页表，接着将之前uservec保存在trapframe中的寄存器数据恢复回去，最后执行sret指令，sret进行了以下动作：
1. 更新 PC：PC <- sepc
2. 恢复特权级别：如果 sstatus.SPP 为 1，则进入监督模式；否则进入用户模式。
3. 恢复中断使能状态：将 sstatus.SPIE 的值赋给 sstatus.SIE，然后将 sstatus.SPP 清零。

至此，我们将xv6中trap的完整流程梳理了一遍。