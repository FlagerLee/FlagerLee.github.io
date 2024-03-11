---
title: "Synthesizing MILP Constraints for Efficient and Robust Optimization"
date: 2024-03-07 12:00:00 +0800
categories: [论文, PLDI, '2023']
tags: [synthesize, programming language]
math: true
---

## Background

### MILP

MILP全称Mixed Integer Linear Programming，混合整数线性规划，即变量约束既有整数约束又有条件约束的线性规划。

目前MILP的的约束不好写，有两方面的原因：第一是许多约束很难直接由线性函数表示出来，因为有很多约束是以布尔变量表示的，而且会有很多诸如$AND\ \land$，$OR\ \lor$，$Implication\ \longrightarrow$等逻辑符号，而这些布尔操作是非线性操作，需要将这些约束转化成整数操作（0和1表示）。第二是很多专家在写MILP约束的时候总是将原约束写成进行手动优化后的表达式，而这对用户会产生很大的困扰。因此，作者提出了一种将布尔逻辑运算自动转换为正确高效的MILP约束的方法。

![Alt text](/assets/img/2024-03-07-0-1.png)

### SyGuS

SyGuS全程Syntax Guided Synthesis，是基于语法的自动综合问题（并不是某种语言或工具，是问题的缩写）。在本文中，作者提出了一种DSL来解决SyGuS问题。

## Method

顶层程序SynMio输入SMT规范程序，并输出等效的MILP约束。SynMio算法如下图所示：

![Alt text](/assets/img/2024-03-07-0-2.png)

SynMio首先从输入$\phi$中提取常数、关系、变量和数组并用其初始化DSL。DSL的语法如下图所示：

![Alt text](/assets/img/2024-03-07-0-3.png)

在循环内，SynMio首先检查MILP约束$\psi$是否可实现；如果$S_E$中有证据表明$\psi$不可实现，则强化DSL以排除这种情况（即EnrichDSL子程序的功能，它返回强化后的DSL和更新过的输入$\phi$），这一部分算法后续将详细讲解。随后调用函数GenCandi，使用DSL、$S_E$和$\phi$生成MILP约束$\psi$。如果$\phi$和$\psi$被VerifyEQ子程序检验为相等的，则返回空集$E$，否则返回错误集合$E$，随后更新错误集$S_E$。如果返回了空集$E$则说明生成完成，程序退出。

### EnrichDSL

### GenCandi

### VerifyEQ

这些部分不是那么感兴趣且涉及到很多线性规划和凸优化等内容，就没看了