---
title: "Mosaic: An Interoperable Compiler for Tensor Algebra"
date: 2024-02-28 11:20:00 +0800
categories: [论文, PLDI, '2023']
tags: [tensor, compilation, dsl]     # TAG names should always be lowercase
math: true
---

## Background

Sparse Tensor Algebra(TODO)

稀疏张量代数(Sparse Tensor Algebra)被用于描述张量计算。目前许多计算平台(如CPU、GPU、向量运算器、领域特定硬件等)都实现了在各自平台上高度优化过的稀疏张量代数库，因此张量计算通常被实现为一系列对张量代数库函数的调用序列。按照这个逻辑，好像张量计算这个问题也不难...吗？

张量计算还有一个特点是多个运算可以融合(Fusing)成一个运算，比如SDDMM运算(sparse sampled dense-dense matrix multiplication)。简而言之，有一个稀疏矩阵$B_{m,n}$，两个稠密矩阵$C_{m,j}$和$D_{j,n}$，先计算矩阵$C_{m,j}$和$D_{j,n}$的点积$E_{m,n}=C_{m,j}\cdot D_{j,n}$，再计算矩阵$B_{m,n}$和$E_{m,n}$的每个对应元素乘积，即$A_{ij}=B_{ij}\cdot E_{ij}$，最终得到矩阵$A_{m,n}$。用代数表达即如下公式：

$$A_{ij}=\sum_kB_{ij}\cdot C_{ik}\cdot D_{kj}=B_{ij}\sum_kC_{ik}\cdot D_{kj}$$

SDDMM运算需要一次稠密矩阵计算，一次稀疏矩阵计算，但通过融合算子之后可以只进行一次稀疏矩阵运算，并且花费总时间更少。许多融合算子能在渐进时间复杂度上优于原先的运算，甚至可以通过诸如调整循环顺序等方法来避免Transpose、Reshape等操作。但库函数一般只提供通用的计算，融合算子需要手写代码或通过编译器生成代码。但是编译器生成代码的运算速度与库函数的运算速度往往相差很大，因此对于一个张量运算表达式，最好是综合考虑融合算子和库函数调用，采用二者之一或者融合使用。

好像这样就行了...吧？但问题就是，现在的编译器**不支持**同时融合生成代码和库函数调用两种方法，需要手写，而手写的麻烦事想想就很多(作者给出了一堆理由，懒得抄了)。所以看到这里都知道作者要实现一个支持自动生成混合代码的编译器啦！但问题还不止于此，由于可选用的库太多了再加上考虑到人都是懒惰的，因此作者最终实现的是一个支持自动生成混合代码且拥有自动调度(auto-scheduling)功能的稀疏张量代数编译器。

## Mosaic

Mosaic是一个TACO的扩展，它能将稀疏张量代数表达式编译成“编译器自动生成代码”和“外部函数调用”的混合代码，并基于TACO的调度框架新增了一系列调度指令。其基本流程如下图所示：

![Mosaic workflow](assets/img/2024-02-28-“Mosaic: An Interoperable Compiler for Tensor Algebra”-1.png)