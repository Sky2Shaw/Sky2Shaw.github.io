---
title: Hazard Pointer
description: notes on Hazard Pointer
author: sky2shaw
date: 2024-08-11 21:41:00 +0800
categories: [Blogging, Demo]
tags: [cocurrent]
pin: true
math: true
mermaid: true
---

## 前言
这周为了实现一个基于流水线的并发、无锁的报文转发框架，研究了Hazard Pointer。这里简单记录下研究的结果

## 什么是Hazard Pointer？

Hazard Pointer是一种用于并发系统中内存回收的技术。它最早是由Maged M. Michael为了解决免锁队列内存回收的问题而提出的。Hazard Pointer的原理其实并不复杂，简单来讲，它维护了一个所有线程都可以读的数组，数组的元素是一个指针；对于特定的指针而言，只有一个线程可以修改它。当一个线程，不妨叫它线程A吧，需要访问某个指针时，它会将这个指针添加到数组中。线程B要释放这个指针时，会遍历这个全局数组，当发现这个指针在这个全局数组里面时，线程B就知道某个其他线程正在使用这个指针，于是它就会暂时不释放这个指针。线程A在访问完这个指针后，会从全局数组中删除这个指针，这个时候其他某个线程就可以安全地释放这个指针了。

