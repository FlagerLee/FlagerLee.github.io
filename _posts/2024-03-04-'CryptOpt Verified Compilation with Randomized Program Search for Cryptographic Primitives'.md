---
title: "CryptOpt: Verified Compilation with Randomized Program Search for Cryptographic Primitives"
date: 2024-03-04 12:00:00 +0800
categories: [论文, PLDI, '2023']
tags: [crypto, programming language]
math: true
---

## Background

### Crypto program

密码学核心程序通常采用手写汇编等较为底层的方式编写而非采用高级语言。其主要原因为
* 密码学代码对速度要求高，手写优化更好。编译器的优化通常更针对于控制流图的优化，而在基本块内的优化策略很少。然而密码学程序会尽可能避免使用依赖于数据的控制流和内存访问（为了防范侧信道攻击），这是的编译器对代码的优化有限，在基本块内手写汇编的优化会更多。
* 手写可以更好地防范各种攻击
* 手写可以保证代码行为一致（即不会受到优化的影响）

### Random Local Search

RLS是一种简单的最优化问题的策略。RLS首先随机生成候选解，随后在候选解上进行随机“突变”，生成新的候选集。如果候选集中的某突变效果不比候选解差，则该突变保留，用以替代原候选解，否则保留原候选解。该过程一直持续到结束条件被触发。

RLS的问题是其效果对初始的候选解选择高度敏感，而Bet-and-Run方法解决了这个问题。Bet-and-Run会同时独立运行RLS，从中选取最优解，随后再以该最优解作为初始候选解开始继续优化。

### Finite-Field Arithmetic for Cryptography

有限域算术（Finite-Field Arithmetic, FFA）是在有限域内进行的算术，简单举例就是XOR和AND运算是GF(2)中的加法和乘法（GF(2)是伽罗瓦域，在GF(2)中的任意运算$\otimes$等价于$A\otimes B = C (mod\ 2)$）。许多常见的密码学算法都是有限域算术，比如椭圆曲线加密（ECC）。

有限域算术的运算量较大，对运行效率的要求也很高，因此其中许多函数的实现都进行了手动优化，比如OpenSSL中的Curve25519，NIST P-256等函数。但是由于这一部分的代码基本上没有复杂的控制流，是“直线型”代码（straight-line code），因此编译优化效果有限，而手动优化通过挑选指令、指令调度、寄存器分配等优化，甚至于cache预测、cache替换策略等微架构优化策略，效果比编译器生成代码好很多。

### Fiat Cryptography

Fiat Cryptography是由[Erbsen et al. [2019]](https://dl.acm.org/doi/abs/10.1145/3421473.3421477)提出的，能将域算术（field arithmetic）转换为代码并提供Coq证明功能正确的框架。Fiat Cryptography会根据域的范围生成被证明正确的Fiat IR，然后通过支持的语言（C，Java，Zig）生成一个实现。

### Equivalence Checking

Equivalence Checking是为了保证生成的程序行为和代码严格一致，比如CompCert使用Coq证明自己是一个正确的C编译器。但是证明一个编译器是正确的是工作量巨大的工程，因此更多的解决方法是提供一个checker function来验证编译器输出是正确的。

Equivalence Checking还可以使用SMT solver来推理证明high-level input和low-level output的等价性。

### E-graph

E-graph是一种用于储存等价关系的数据结构，它能通过一系列的变换和搜索来找到表达式的最简表达。

## CrpytOpt

作者的目标是加强Fiat Cryptography的功能，既要提高生成代码的性能，又要减小生成的可信库的大小。为此作者实现并为Fiat Cryptography整合了两个新组件：CryptOpt optimizer和program-equivalence checker，如下图所示：

![](/assets/img/2024-03-04-'CryptOpt%20Verified%20Compilation%20with%20Randomized%20Program%20Search%20for%20Cryptographic%20Primitives'/1.png)

CryptOpt optimizer使用之前提到的RLS算法对程序进行优化。program-equivalence checker用于在给定原始Fiat IR和优化过的汇编的情况下验证二者的行为是否等价。checker本身是在Coq中实现并进行验证的，因此checker本身是可信的，只有optimizer是不可信的。但是optimizer在设计上是进行保留语义的转换，因此optimizer是被期望正确的。

### Randomized Search For Assembly Programs

![](/assets/img/2024-03-04-'CryptOpt%20Verified%20Compilation%20with%20Randomized%20Program%20Search%20for%20Cryptographic%20Primitives'/2.png)

Fiat IR语法如上所示，需要注意的是整数以二进制形式表示以清晰表明位宽。部分表达式有多返回值，比如带进位的加法或产生双宽结果的乘法。可以看出来，Fiat IR没有控制流语句，程序都是线性的，因此对Fiat IR生成代码的优化可以集中在指令层面。CryptOpt的优化分为三个方面：Instruction Scheduling，Instruction Selection和Register Allocation。

#### Instruction Scheduling

虽然RLS对初始解的要求较高，但是作者提供的变异策略保证通过一系列变异可以从任何随机初始解获取任何可能的排序。变异规则为随机挑选一个指令并随机移动到一个合法的位置。合法的位置即为满足数据依赖的位置。移动的距离将尽可能地远，使得得到的结果能尽快使用。

#### Instruction Selection

x86等复杂指令集的一个重要特性是每种高级操作都可能有多种实现方式，而每种实现方式的语义和对流水线的影响都有细微差别。比如加法操作可以使用add、adcx、adc等指令实现，但是这些指令有着很细微的差别，比如在无输入进位的情况下，使用adcx指令需要清除进位标志。CryptOpt使用模板来描述每种操作的可能实现，比如adcx指令的模板会考虑到这一点，在执行adcx指令之前会发出一条清除进位的指令clc，除非CryptOpt确定进位已经被清除。

生成初始解的时候，CryptOpt会为每个操作随机选择一个模板作为初始映射。突变则会随机选取一个操作并使用该操作的另一种指令模板代替原先的模板。

#### Register Allocation

CryptOpt中的寄存器分配有两个目的：决定将哪些值分配给寄存器，以及在缺乏寄存器的时候释放哪些寄存器。

当计算得到新结果或者从内存中读取值（从输入获取或寄存器溢出）时需要分配寄存器。当需要分配寄存器的时候，CryptOpt会选择空闲的寄存器，否则会将某些寄存器中的内容保存到内存并使用这些寄存器。选择的逻辑是根据寄存器将来的使用情况，选择距离下一次使用最远的寄存器。

由于寄存器分配的算法是固定且确保最佳的，因此这一步不需要进行突变。

### Checking Program Equivalence

作者在Coq中实现了一个程序等价性验证程序来验证生成的汇编行为是否与Fiat IR一致。作者为Fiat IR和x86-64两种语言的相关子集（即被使用到的部分）开发了简单的符号执行引擎（symbolic-execution engine）并以通用逻辑格式生成程序状态描述。之后检查两个程序的函数输出状态（即输出寄存器和指定的内存位置）是否储存了可被证明为相等的值。作者为此开发了一个等价定理证明器，它使用了E-graph来储存等价性图并借鉴了SMT求解器。

Fiat IR的执行引擎比较简单，因为Fiat IR本质上就是一堆代数表达式的组合，最终结果能很轻易地用表达式表达出来。而x86-64地执行引擎更复杂，因为逻辑符号不再是程序变量，而是寄存器和内存地址。作者创建了一个字典，key是逻辑变量和整数的pair，value是内存地址（存疑，原文为： Thus, it is appropriate to make the symbolic memory a dictionary keyed off of pairs
of logical variables and integers. The logical variable is the base address of an array in memory,
while the integer gives a fixed offset into one of its words），这样可以将每条x86指令分解为多个较为简单的操作。在执行结束后，会从调用结束后的寄存器和内存中读取结果，并与Fiat IR得到的结果进行比较。匹配算法如下图所示：
![Alt text](/assets/img/2024-03-04-'CryptOpt%20Verified%20Compilation%20with%20Randomized%20Program%20Search%20for%20Cryptographic%20Primitives'/3.png)

## performance

TODO