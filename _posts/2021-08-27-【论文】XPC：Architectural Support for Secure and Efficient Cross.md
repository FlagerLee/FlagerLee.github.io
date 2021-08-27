---
title: 【论文】XPC：Architectural Support for Secure and Efficient Cross Process Call
date: 2021-08-27 17:36:00 +/-0800
author: flagerlee
excerpt: 论文阅读
categories: [计算机系统结构，微内核，论文阅读]
tags: [计算机额系统结构]
---

## 原语/概念/名词

### Domain

Domain是指某个对象及其操作的集合$\\{Object,\ Operations\\}$。Domain switch则是字面意思，切换当前可以操作的对象，比如切换地址空间则是进行一次Domain switch。

### Endpoint

Endpoint是微内核中的一种用于实现IPC通信机制的小内核对象，代表为seL4。Endpoint由一个等待发送队列或一个等待接收队列组成（猜测二者是同一队列，发送多于接收则queue发送，接收多于发送则queue接收）。参考资料[seL4 Docs](https://docs.sel4.systems/Tutorials/ipc.html)。

### Capability

Capability是L4微内核中提出的一种权限管理机制。个人理解，之前的特权级机制针对的是进程，但这种粗粒度的权限划分容易让进程获得过高的权限（比如进程需要执行某个高权限操作A，但是系统又不希望其执行其它同样权限的操作，这是特权级机制的弊端）；而Capability机制则是针对操作进行权限划分，细粒度地操作权限。[参考资料](https://pdos.csail.mit.edu/6.828/2008/lec/l-microkernel.html)。

### $x-entry$

$x-entry$是不经过内核的进程切换的入口，进程通过命令指定一个$x-entry$作为入口，进入注册该$x-entry$的进程继续执行。

$x-entry$由页表指针(Page Table Pointer)，Capability指针(Capability Pointer)，入口地址(Entry Address)和段列表(Segmentation List)（valid位可以认为是$x-entry-table$的内容）。

### $xcall-cap$

$xcall\ capability$的缩写，是一个bitmap，其中第i个bit标志着当前进程是否可以调用第i号$x-entry$。

### $xcall$

$xcall$是论文提出的一个原语，用法是$xcall\quad\#reg$，$\#reg$是一个标号，意思是进入第$\#reg$个$x-entry$继续执行，可以理解为是一个函数调用。

### $xret$

与$xcall$配套的原语，从$xcall$的执行中返回（即从调用的进程中返回原进程）。

### $relay-seg$

$relay\ memory\ segment$的缩写，直译为中继段。$relay-seg$是论文提出的用于进行快速消息传递的一种地址空间映射，可以做到零拷贝。对于$relay-seg$的虚拟地址，其地址空间连续，对应的物理地址空间也连续，且长度相同。$relay-seg$可以在进程之间传递，所以这一段物理地址空间也可以被不同的进程访问，从而实现零拷贝的信息传递。

### $seg-reg\ \&\ seg-mask$

$seg-reg$和$seg-mask$是$relay-seg$的一部分。$seg-reg$是一个寄存器，用于翻译$relay-seg$。它由虚地址基址(VA Base)，物理地址基址(PA Base)，长度(Length)和访问权限(Permission)组成。

![seg-reg](/assets/img/posts/2021-08-27_XPC_Architectural_Support_for_Secure_and_Efficient_Cross/seg-reg.png)

*_seg-reg_*

$seg-mask$

