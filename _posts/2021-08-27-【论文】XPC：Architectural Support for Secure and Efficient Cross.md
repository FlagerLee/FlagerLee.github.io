---
title: 【论文】XPC：Architectural Support for Secure and Efficient Cross Process Call
date: 2021-08-27 17:36:00 +/-0800
author: flagerlee
excerpt: 论文阅读
categories: [计算机系统结构，微内核，论文阅读]
tags: [计算机系统结构]
---

## 背景

本篇论文通过硬件和软件层面的修改，引入了作者称为XPC的机制来解决微内核中IPC(Inter-Process Call)开销过大的问题。

关于为什么微内核中IPC的问题比单片内核中的问题更大，我更倾向于认为是因为微内核的IPC使用率更高导致的。

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

### $xcall-cap-reg$

指向$xcall-cap$的寄存器。

### $xcall$

$xcall$是论文提出的一个原语，用法是$xcall\ reg$，$reg$是一个标号，意思是进入第$reg$个$x-entry$继续执行，可以理解为是一个函数调用。

### $xret$

与$xcall$配套的原语，从$xcall$的执行中返回（即从调用的进程中返回原进程）。

### $relay-seg$

$relay\ memory\ segment$的缩写，直译为中继段。$relay-seg$是论文提出的用于进行快速消息传递的一种地址空间映射，可以做到零拷贝。对于$relay-seg$的虚拟地址，其地址空间连续，对应的物理地址空间也连续，且长度相同。$relay-seg$可以在进程之间传递，所以这一段物理地址空间也可以被不同的进程访问，从而实现零拷贝的信息传递。

当然，share memory会被TOCTTOU攻击的缺点同样存在于这种设想中，所以内核会保证一个$relay-seg$同时只能在一个cpu核心上活跃，从而避免其它进程监视甚至修改内存中的数据。

### $seg-reg\ \&\ seg-mask$

$seg-reg$和$seg-mask$是$relay-seg$的一部分。$seg-reg$是一个寄存器，用于翻译$relay-seg$。它由虚地址基址(VA Base)，物理地址基址(PA Base)，长度(Length)和访问权限(Permission)组成。

![seg-reg](/assets/img/posts/2021-08-27_XPC_Architectural_Support_for_Secure_and_Efficient_Cross/seg-reg.png)

$seg-mask$是配合$seg-reg$使用的。由于$seg-reg$的分配是由内核控制的，进程不能直接更改$relay-seg$的映射，所以通过使用$seg-mask$让callee收到的信息为$seg-mask\&seg-reg$，来让callee只收到$relay-seg$中的一部分信息。

### $seg-list$

一个process可能有多个$relay-seg$，这些$relay-seg$都保存在$seg-list$内。

### $seg-list-reg$

指向$seg-list$的寄存器。

### $swapreg$

$swapreg$是一个操作，用于将当前的$seg-reg$与$seg-list$中的一个$seg-reg$进行交换。使用方法是$swapreg\ reg$，$reg$是$seg-list$中的标号。

### $linkage\ record$

$linkage\ record$是在调用$xcall$之后记录的恢复状态必需的信息，相当于保存的context。$linkage\ record$里的信息包括页表指针(Page Table Pointer)，返回地址(Return Address，应该是$xcall$下一条指令的地址)，$xcall-cap-reg$，$seg-list-reg$和$relay-seg$。

### $link\ stack$

$link\ stack$是用于保存$linkage\ record$的栈，以线程为单位保存。从论文3.2节xcall的介绍中看，调用$xcall$后$linkage\ record$看起来是保存在caller的栈里的，但是从实际应用角度出发，$linkage\ record$内的内容是用于恢复调用前状态的，保存在caller栈内会让callee无法拿到对应的record，导致难以还原。所以我认为$linkage\ record$是保存在callee的$link\ stack$内的。

### $link-reg$

指向$link\ stack$的寄存器。

### $XPC-Engine-Cache$

由于IPC有高时间局部性和高预测性，所以可以使用一个cache对$x-entry$进行预取，用来加快$xcall$的速度。

## XPC执行流程

首先假设两个进程$A$和$B$进行通信，$A$发送消息给$B$，则称$B$为server，$A$为client。$B$首先注册一系列$x-entry$，内核会将这些$x-entry$存放到$x-entry-table$中；$A$向$B$发送消息的时候，先通过父进程或名称服务器获取到$B$注册的可用$x-entry$的编号$reg$；然后将需要发送的消息存放到$relay-seg$的那段内存中（或者原来就已经存在了）；再$xcall\ reg$调用$B$注册的$x-entry$：此时硬件会①检查$xcall-cap$，确认$A$是否有权限调用该$x-entry$②读取$x-entry$并检查valid bit（如果Enable $XPC-Engine-Cache$则会预取$x-entry$，不用进行这一部分）③记录$linkage\ record$并推入$linkage\ stack$中④跳转到$B$进程中，执行$B$中的代码；$B$进程会根据传递的$xcall-cap-reg$判断caller的身份（因为$xcall-cap-reg$和runtime state绑定，后面会说），并且可以使用$relay-seg$中的数据，也即完成了进程间通信。

## 实现

### risc-v

没有什么是扩充标准解决不了的（大雾）。risc-v的新增内容如下图所示：

![risc-v implementation](/assets/img/posts/2021-08-27_XPC_Architectural_Support_for_Secure_and_Efficient_Cross/riscv_implementation.png)

需要注意的是，如果实现了预取功能，则可以使用$xcall\ -reg$指令来预取$reg$号$x-entry$。当然这带来了一个问题，就是编号需要从1开始编，怎么想怎么奇怪（逃。

### MicroKernel

* 定义软件层面的Capability：$grant-cap$。$grant-cap$拥有$x-entry$的$grant$权限，并可以将$grant$和$xcall$的Capability给予其它线程。当线程创建$x-entry$的时候，其自带$grant$权限，并可以将$xcall-cap$授予给其它线程。
* 不经过内核的进程切换有一个隐患，就是内核不能得知当前用户进程的地址空间。比如进程$A$与进程$B$通信，内核认为当前地址空间是$A$的地址空间，但实际上此时正在执行$B$的代码，当前地址空间是$B$的地址空间。如果此时发生了异常或中断而陷入内核，则内核会用$A$的地址空间保存上下文或执行其他操作，从而导致错误。解决方法是将线程状态分为scheduling state和runtime state。scheduling state中包含了调度相关的信息，比如内核栈、优先级、时间片等；而runtime state包含的是地址空间和Capability。每个线程都绑定了一个scheduling state，不会更改，保证线程的正常运行；而runtime state可能会有很多。因此内核可以通过线程的runtime state来正确为线程提供服务。另外，$xcall-cap-reg$是与runtime state绑定的，所以内核可以通过$xcall-cap-reg$来寻找对应的runtime state。
* 一个$x-entry$可以被多个进程同时调用，因此需要在创建$x-entry$时指明该$x-entry$最多可以被多少个进程调用。为了实现这个特性，XPC库为每个调用提供了XPC的上下文，上下文包括一个执行栈（称为C-Stack）和本地数据。在创建$x-entry$时，会同时创建指定数量个XPC上下文和一个trampoline。当调用$x-entry$时，trampoline会自动找到一个空闲的XPC上下文，切换到对应的C-Stack并恢复调用前的本地数据；如果不存在空闲的上下文，则抛出异常或等待资源释放。这样做的缺点是容易遭到DOS攻击，攻击者可以通过大量调用$x-entry$来进行攻击：这要求内核或进程采取措施来限制这种攻击（作者举的例子是M3和Intel QP的信用系统，不太了解）。
* 需要考虑的事情是如何优雅地处理调用链中的异常。比如有一条调用链$A\rightarrow B\rightarrow C$，在$C$进程中执行的时候$B$进程崩溃，则$C$在$x-ret$的时候会跳到错误的地址，导致崩溃。作者给出的实现是：在进程结束的时候，内核会扫描所有的$link\ stack$并找出与该进程有关的链接记录（通过页表判断）并将valid置为0，这样在$xret$的时候就会触发异常并交给内核进行处理。（但是感觉这样的处理在$link\ stack$比较多的时候会很耗时间，尤其是微内核的IPC操作及其频繁，$link\ stack$不会少；不知道实际测试时表现如何。）

### Android Binder

我对Android中的Binder机制很不了解，所以这部分看得也似懂非懂的，之后再填。



## IPC调用链

为了实现零拷贝，还有一些问题需要解决。

* 可能有调用链中的进程增大消息长度（类似的情况有request header）。假定有一个调用链$A\rightarrow B\rightarrow[C|D]$，$A$调用$B$，$B$可能调用$C$或$D$。定义$S_{self}(B)$是进程$B$自己可能增加的空间大小，$S_{all}(B)$是进程$B$及其之后调用链可能增加的总空间大小，则$S_{all}(B)=S_{self}(B)+Max(S_{all}(C),S_{all}(D))$。在进程$A$第一次调用进程$B$的时候，$A$会请求$S_{all}(B)$，从而预留足够的空间。我个人对此有些疑问，$S_{all}$是一个递归计算得到的值，$A$怎么在调用$B$之前正确且快速地从$B$获取$S_{all}(B)$？这很容易构造出一个足够复杂的调用链让这个计算过程变得很缓慢。我没看过这部分的代码，也对这一部分的实现有很多疑惑。
* 可能有调用链中的进程缩小消息长度（类似的情况有OS将数据分block写入磁盘），这种情况比较简单，使用$seg-mask$寄存器即可。
* 在进程销毁的时候需要释放$relay-seg$，这部分操作由内核完成。内核会扫描该进程的$seg-list$，将caller的$relay-seg$返回给caller，自己的$relay-seg$回收。
