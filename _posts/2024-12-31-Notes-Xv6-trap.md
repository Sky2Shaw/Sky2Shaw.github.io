---
title: xv6笔记: Trap
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
在学习xv6之前，系统调用在我的印象中就是操作系统开放给用户的一些列诸如open、fork之类的函数，并没有很深入地去了解其原理。
在进一步讨论之前，我们先回顾下什么是系统调用。AI告诉我们：系统调用是用户程序与操作系统内核之间的接口，用于请求操作系统执行某些特权操作或访问受保护的资源。基础的图像是这样的：
![system call base picture](//assets/img/xv6.001.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

这个定义些微有点抽象，我们不妨直接到xv6的源码里面看下，系统调用到底是怎么实现的吧。



