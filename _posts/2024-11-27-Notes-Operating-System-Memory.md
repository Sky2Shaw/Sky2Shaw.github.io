---
title: 操作系统导论读书笔记：内存
description: notes on operating system
author: sky2shaw
date: 2024-11-27 22:27:00 +0800
categories: [Blogging, Demo]
tags: [os]
pin: true
math: true
mermaid: true
---

## 前言
我最近在学习操作系统，准备完成xv6的Lab，顺便读一读操作系统导论（Operating System: Three Easy Pieces）。这本书确实是本好书，值得一读。为了避免读了就忘，写篇博客记录自己的理解。

## 什么是内存的虚拟化？
首先一个问题，什么是内存的虚拟化？内存的虚拟化，是指操作系统给进程提供了一个“假象”，让进程看起来拥有一块很大的独占的连续物理内存。以64位系统为例，进程可以拥有2^64个字节大小的虚拟内存，也就有2^64个虚拟地址。为什么说说这些地址是虚拟的呢？因为进程看到的这些地址并不是真实物理内存的地址，需要操作系统进行转换才能得到真实的物理内存地址。

## 为什么要虚拟化内存？
第二个问题，为什么要虚拟化内存？或者说，为什么操作系统要为每个进程制造一个它们都拥有巨大独占内存的假象呢？主要是为了让生活变得更加轻松。有了这样的一个假象之后，程序员在编写程序的时候就不用管数据或代码到底存放在物理内存中的哪个地方了，也不用管内存是否足够大。想象一下，必须将代码和数据放到一个拥挤的小而拥挤的内存，这将是多么痛苦的事情啊。
还有一点是为了不同进程之间内存的隔离。不同进程的虚拟内存被操作系统映射到了不同的物理内存，这样就从根本上避免了不同进程之间错误修改对方内存数据的风险。

## 操作系统是如何实现内存的虚拟化的？
我们来想一下，如果让我们来实现内存的虚拟化，我们该如何实现。如何为进程制造一个它有巨大连续内存的假象？最好这个内存还是从零开始的，毕竟这样是最简单的。
我们知道，执行程序时，程序的代码和数据会被加载到实际的物理内存中。假设我们的这个程序占用了16kB的内存，内存的起始位置是实际物理内存的第32kB。这个时候要给进程制造一个它占用的内存是从0到16kB连续空间的假象，操作系统可以做这样一件事，当程序要访问虚拟内存的第0个地址时，也就是物理内存的第32kB时，我们的操作系统将这个0转换成物理内存的第32kB。进程继续访问虚拟内存的第1kB，操作系统将这个1kB转换成物理内存的第33kB，依此类推。这其中其实有一个分层的思想，进程和物理内存硬件之间，多出了一个中间层，中间层的工作由操作系统来完成，它接收进程要访问的虚拟地址，将这个虚拟地址转化为硬件能够定位的真实的物理地址，并向硬件发起请求，要求返回该地址的数据，并将数据返回给进程。分层的思想在计算机领域是无处不在的。高层通过低层提供的服务，实现功能，但又无需感知低层是如何实现的。操作系统完成了虚拟内存地址到物理内存地址的转换，向进程屏蔽了转换的细节。

上面我们提到了，操作系统替进程完成了虚拟内存到物理内存的转换，那操作系统是怎么做到的呢？这是我们接下来要重点讲的内容。我们思考一下，首先是最简单的场景，进程的虚拟内存连续地存放在物理内存中（线性映射），这个场景非常的简单，只需要将进程所占用的物理内存的起始地址存放在某个地方即可，这样操作系统在进行虚拟地址到物理地址的转换时，只需要使用这个公式即可：物理地址 = 虚拟地址 + 基址。是不是非常简单？这个基址呢，当然应该放在寄存器里面，因为这是需要频繁使用的关键数据。


