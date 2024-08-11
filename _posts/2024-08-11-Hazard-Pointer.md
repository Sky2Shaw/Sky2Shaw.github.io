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
{: .mt-4 .mb-0 }

这周我为了实现一个基于流水线的并发、无锁的报文转发框架，研究了Hazard Pointer。这里简单记录下研究的结果。

## 什么是Hazard Pointer？
{: .mt-4 .mb-0 }

Hazard Pointer是一种用于并发系统中内存回收的技术。它最早是由Maged M. Michael为了解决免锁队列内存回收的问题而提出的。

## Hazard Pointer的原理
{: .mt-4 .mb-0 }

Hazard Pointer的原理其实并不复杂，简单来讲，它维护了一个所有线程都可以读的数组（当然也可以是链表），数组的元素是一个指针；对于特定的指针而言，只有一个线程可以修改它。当一个线程，不妨叫它线程A吧，需要访问某个指针时，它会将这个指针添加到数组中。线程B要释放这个指针时，会遍历这个全局数组，当发现这个指针在这个全局数组里面时，线程B就知道某个其他线程正在使用这个指针，于是它就会暂时不释放这个指针。线程A在访问完这个指针后，会从全局数组中删除这个指针，这个时候其他某个线程就可以安全地释放这个指针了。
{: .mt-4 .mb-0 }

这里有一点我其实一度是有点疑惑的，那就是，在线程B准备释放某个指针P，并且遍历全局数组发现指针P不在这个数组里面时；这个时候其他线程恰好抢占了CPU，将指针P添加到全局数组里面；接着当线程B继续执行时，岂不是会错误地释放指针P？
{: .mt-4 .mb-0 }

这毫无疑问是有可能的。但Hazard Pointer的使用方式规避了这一可能。下面我们看看Hazard Pointer的使用方式。

## Hazard Pointer的使用方式
{: .mt-4 .mb-0 }

首先，假设这么一个场景，在一个并发系统中，需要访问一个全局的数组，当数组的容量不够的时候，重新分配一个两倍大小的新数组，并将原来的数组释放掉。这个释放动作是非常危险的，因为可能有其他线程在访问旧数组，所以我们要通过Hazard Pointer来避免错误释放。释放的过程是这样的：
```c
    int *g_array;
    ...
    int *new_array = new int[2 * old_size]; // step 1
    memcpy(new_array, g_array, old_size * sizeof(int *)); // step 2
    int *old_array = g_array; // step 3
    g_array = new_array; // step 4
    bool found = scan_hzdptr(old_array); // step 5
    if (!found) delete[] old_array;
```
可以看出，这里面有个关键步骤，也就是step 4，它将指向全局数组的指针指向了新分配的数组，这就保证了其他线程如果没有将旧的指针“保护”起来，再也不能访问到旧的数组了。于是在step 5扫描完Hazard Pointer数组之后，若没有线程正在使用旧的数组，那么就可以安全地释放旧的数组了。