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

![Mosaic workflow](/assets/img/2024-02-28-“Mosaic:%20An%20Interoperable%20Compiler%20for%20Tensor%20Algebra”-1.png)

首先，Mosaic需要能够调用外部库函数，这要求不同的库函数需要向Mosaic提供统一的接口，因此作者实现了称为*External Function Interface*的DSL，用于描述不同的库函数。其次，TACO使用[tensor index notation](http://tensor-compiler.org/docs/pycomputations.html)来描述张量计算，而Mosaic使用tensor index notation和一种形式化语言\[[Chou et al. 2018](https://doi.org/10.1145/3276493)\]描述稀疏张量代数表达式输入。最后，作者在TACO的调度框架上集成了新的调度指令让生成结果更高效。

## Implementation

### External Function Interface

External Function Interface本质上是一个C++类，用于提供外部函数的信息让Mosaic能正确调用。每个External Function Interface包含以下7个部分：
* Calling description：提供函数名字、参数和返回类型(#Line 6)
* Setup code：一段代码，用于在调用函数之前初始化上下文（比如初始化特定内存、数据迁移等）(#Line 8)
* Teardown code：一段代码，函数调用后的postprocess（比如检查错误代码、释放内存、数据迁移等）(#Line 11)
* Function capabilities：用于描述函数执行的张量计算(#Line 12)
* Include paths(#Line 15)
* Library paths(#Line 17)
* Build flags(#Line 19)

样例如下图所示：
![Sample external function interface](/assets/img/2024-02-28-“Mosaic:%20An%20Interoperable%20Compiler%20for%20Tensor%20Algebra”-2.png)

#### Calling convention

需要注意的是Calling description、Setup code和Teardown code使用的都是新的语法，而非C++原本的语法，从返回类型为CallingDescription上能看出。

#### Compute Capabilities

函数语义描述的困难之处在于它的约束多种多样。有的函数只计算一个表达式，而有些能计算无数个表达式，还有些函数对数据的属性有要求，甚至还需要描述硬件约束。因此函数语义描述语言由三部分组成：
* Capability language：该语言在tensor index notation的基础上进行了扩展，支持了未指定秩的张量。下一张图片将贴出语法，具体内容不作详细介绍。
* Checker function：Capability面对某些约束仍无法表示，因此Mosaic提供了checker function。Checker function是由用户编写的C++函数，它接受一段tensor index notation代码并返回一个boolean值来表示该表达式当前是否能使用该函数进行计算——比如当硬件正在占用时，该函数返回false。相应地，该函数的开销远大于Capability language。
* Tensor Properties：针对输入的约束。许多函数仅接受特定形状的张量输入。Mosaic同样使用Capability language对张量属性进行定义。

![Capability language](/assets/img/2024-02-28-“Mosaic:%20An%20Interoperable%20Compiler%20for%20Tensor%20Algebra”-3.png)

除此之外，张量还可以注释属性(property)，比如该张量是Hermitian matrix，是否是正定的等等，由此可以使用更精细优化的函数。

### Expression binding

该步骤用于将张量子表达式绑定到特定的函数，可以手动绑定也可以自动绑定。

使用调度指令bind：$S'=S.bind(s,f)$，用于将表达式$S$中的pattern为$s$的子表达式绑定到函数$f$。bind指令返回绑定完成后的表达式$S'$，但出错时返回错误信息（比如$s$和$f$语义不匹配）。

随后，Mosaic会验证该绑定的正确性。Mosaic会使用Z3证明器检查输入和输出的数据类型是否匹配、运算是否匹配、索引变量匹配问题。如果不匹配，则绑定失败。

除了bind指令，Mosaic还提供一个更简洁的map指令：$S'=S.map(s,f)$。bind指令需要用户严格预处理数据使张量与函数接口吻合（如需要自己做tiling、reshape等操作），而map指令会调用自动绑定中的自动搜索算法，自动完成tiling和reshape操作来匹配函数接口以及匹配索引变量，省去了这一步。

### Automated search for bindings

除了手动绑定，Mosaic还支持输入表达式后自动绑定。

![Automated search for bindings](/assets/img/2024-02-28-“Mosaic:%20An%20Interoperable%20Compiler%20for%20Tensor%20Algebra”-4.png)

自动绑定分为五步完成：
1. 剔除输入输出不符合要求的外部函数
2. 使用pattern matching寻找匹配运算的函数
3. 匹配索引变量
4. tiling验证，确保索引变量变换正确，确保维度约束正确
5. 调用checker function检查生成的代码是否正确。如果报错，则尝试所有的排列（直到达到最大搜索深度）

以下将详细介绍步骤2、3、4。

#### Operator Pattern Matching

这一步主要匹配运算符，而忽略张量顺序和维度等问题。匹配算法会考虑到不同运算的分配性、关联性、交换性、拆分还原等性质进行匹配。函数能否匹配成功与函数的Capability description强相关，比如一个函数支持三个张量相加，而表达式中只有两个张量相加，则Mosaic会为其添加一个零张量相加来适配该函数。

#### Tensor Index-Variable Matching

这一步用于匹配不同张量之间的索引变量。比如$\sum_iA_{ij}+\sum_iB_{ik}$的第一维变量可以匹配，写成$\sum_i(A_{ij}+B_{ik})$。同时Mosaic还可以通过提升张量维度来让张量有更多的索引匹配空间（但不会提升到超过函数要求的最小维度）。匹配算法采用暴力枚举法，这是由于在大部分情况下张量的阶数都不大于4（根据TACO网站的统计仅有5\%），同时也几乎没有超过一个动态索引的表达式，因此需要搜索的空间很小，用暴力算法就足够了。搜索得到的合法映射列表被送往Tiling validation。

#### Tiling Validation

搜索中得到的映射需要满足约束条件。Mosaic将会生成一个Z3 solver的查询来验证映射是否正确，但是精确验证将会导致潜在的tiling机会被忽视。为了探寻更多优化空间，Mosaic放宽了**索引变量大小必须等于索引维度大小**的限制，改为**索引变量大小必须小于等于索引维度大小**。因此，如果验证结果是正确的，Mosaic会搜索tiling机会来进行优化。

### Code Generation

代码生成部分就是各种函数调用以及它们的预处理和后处理。而没有函数调用的运算则会调用TACO自带的代码生成工具进行代码生成。

### Evaluation

TODO

### Conclusion

TODO