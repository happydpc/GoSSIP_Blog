---
layout: post
title: "Gigahorse: Thorough, Declarative Decompilation of Smart Contracts"
date: 2019-04-18 19:32:51 +0800
comments: true
categories: 
---

作者：Neville Grech, Lexi Brent, Bernhard Scholz, Yannis Smaragdakis

单位：University of Athens and University of Malta, The University of Sydney, University of Athens

出处：ICSE'19

原文：http://yanniss.github.io/gigahorse-icse19.pdf

<hr/>

## Abstract

智能合约是运行在区块链上的应用程序，其具有自主交易加密货币和代币的能力。随着智能合约的飞速发展，各种各样的安全威胁也接踵而至。然而，由于智能合约是以非常底层的字节码的形式存储并运行在区块链上的，如何有效的针对智能合约进行复杂的程序分析已经成为了一个亟待解决的难题。

文章提出了Gigahorse工具链，其核心是一个能够将智能合约由EVM字节码反编译为三地址码的反编译器。反编译得到的中间代码能够显式的体现出原先EVM字节码中隐式的数据流和控制流依赖。Gigahorse通过使用一种基于逻辑的声明式规范，利用高层的语义理解指导底层的反编译过程，成功反编译了超过99.98%的已部署合约。此外，Gigahorse还提供了一套功能齐全的工具链，以用于进一步的程序分析。

<!--more-->

## 1 Introduction

EVM字节码是一种底层的基于栈的设计，其几乎没有任何其它语言中存在的那些抽象，例如函数和函数调用。这种特性极大的阻碍了人们对于智能合约的分析与验证。由于字节码是存储并运行在区块链上的，其设计必须能够保证合约的高效执行和紧凑存储。因此，智能合约的字节码永远都不会为了可读性和反编译的需求进行任何的优化。

只有在理解了EVM字节码的高层语义的前提下，反编译器才能把底层的字节码重建为高层的中间代码。

Gigahorse解决了上述挑战，并做出了如下贡献：

- Gigahorse提出了一个高效的反编译器，相比于目前已有的其它反编译器，其具有更高的精度和完备性；
- Gigahorse提供了一套功能齐全的工具集，可用于进一步的程序分析；
- Gigahorse提供了一种新的反编译思路，即高层的语义理解能够反过来指导底层的反编译过程；
- Gigahorse展示了一种非常规的反编译方式，即使用一种基于逻辑的声明式规范来定义反编译器；
- Gigahorse在以太坊智能合约的全集上进行了评估，并正在为Contract Library网站（https://contract-library.com) 提供支持，该网站免费提供以太坊区块链上所有合约的反编译结果。

## 2 Background

### A. 以太坊区块链中的底层字节码

EVM字节码是一种底层的基于栈的中间代码，而JVM字节码则是一种更高层次的中间代码，两者差异如下：

- 与JVM字节码不同，EVM字节码没有结构体和对象的概念，也没有函数的概念；
- JVM字节码具有丰富的类型系统，而EVM字节码则只有一种类型，即256bit的字；
- JVM字节码在不同的控制流路径下操作数栈的深度是固定的，而EVM字节码则没有这样的执行约束，因此在EVM字节码中识别标准的控制流结构是一件十分困难的事情；
- 在EVM字节码中，所有的控制流跳转都是面向变量的，而不是面向常量的，跳转的目标地址都是从栈中读取的，而在JVM字节码中，所有的控制流跳转都明确定义了目标地址，与数据流无关；
- JVM字节码定义了函数调用和返回的指令，而在EVM字节码中，所有的合约内函数调用都被转化为控制流跳转，即合约内的所有函数被融合为一条指令流，通过控制流跳转传递控制权。

在调用合约内函数时，EVM字节码会将返回地址、参数以及目标地址压入栈中，并执行控制流跳转。在从被调用函数返回时，EVM字节码会从栈中弹出返回地址，并跳转返回。

### B. 声明式程序分析

声明式程序分析是指以一组逻辑蕴涵规则为基础，通过不断递归的规则推理，直至最小不动点，以实现程序分析。Datalog语言是声明式程序分析中最常用到的一种规范语言，其计算是基于逻辑蕴涵规则与递归的。

## 3 Gigahorse Decompilation Specification

### A. 反编译步骤概述

由原始的EVM字节码开始，Gigahorse的反编译步骤如下：

1. 查找基本块边界，输出被分割为基本块的原始字节码；
2. 对基本块进行本地栈分析，输出每个基本块对栈执行的操作；
3. 对合约进行全局栈分析并动态构造CFG，生成全局三地址码；
4. 启发式推断公开函数和私有函数的边界，将全局CFG转化为本地CFG与调用图；
5. 推断所有函数的参数和返回值，生成函数三地址码。

### B. 输入语言

![20190413164935](/images/2019-04-18/20190413164935.png)

Next关系返回一条语句的下一条语句。

Block关系将一条语句映射到一个基本块。

BlockHead与BlockTail关系分别表示一个基本块的首尾语句。

PushesAndPops关系返回一条语句或者整个基本块压入和弹出的栈元素的最大个数。

### C. 本地栈分析

这一步总结了每个基本块对栈执行的操作，引入了变量的概念，并分析了每条语句执行时的栈内容。

![20190413165349](/images/2019-04-18/20190413165349.png)

LocalDefines关系将一条语句与一个新引入的变量相关联。

LocalStackIn与LocalStackOut关系分别表示在一条语句的执行前后各个栈位置上存储的变量。

VariableValue关系将一个变量映射到常量。

NewFreshVar关系是引入一个新变量的构造器。

### D. 全局栈分析与CFG构造

这一步计算了所有控制流跳转的目标地址，动态构造了CFG并对合约进行了全局栈分析。

![20190413171848](/images/2019-04-18/20190413171848.png)

BlockIn和BlockOut关系是LocalStackIn与LocalStackOut关系由本地到全局的推广。

由于控制流跳转的目标地址都是从栈上存储的变量中读取的，因此CFG构造与全局数据流分析是互相递归的。

在完成这一系列的分析之后，Gigahorse基于如下的模式和规则生成了全局三地址码：

![20190413222844](/images/2019-04-18/20190413222844.png)

## 4 Reconstructing Source Level Functions

函数的语义信息在编译的过程中消解了，Gigahorse反编译的一个关键步骤就是函数的推断。函数的推断过程与上一节中CFG的构造过程形成了一个良性循环，即CFG是函数推断的基础，而函数的语义信息又反过来提升了CFG构造的精度。

### A. 公开函数

通过从EVM字节码中识别由Solidity编译器引入的函数调度模式，Gigahorse反编译器能够找到合约中所有的公开函数。

![20190413231121](/images/2019-04-18/20190413231121.png)

通过查询已有的公开函数签名数据库，Gigahorse尝试将函数描述符与源代码级别的函数签名相匹配。

### B. 寻找返回的函数

为了推断私有函数的边界，Gigahorse采用了复杂的启发式规则，这些规则需要完整的全局数据流信息以及CFG。第一个启发式规则是寻找调用函数通过栈向被调用函数传递地址，被调用函数在执行结束之后使用该地址跳转返回的行为。

![20190413233001](/images/2019-04-18/20190413233001.png)

这一启发式规则在经过进一步的完善之后被定义为FnCallRet关系。

### C. 进一步分解函数

Gigahorse采用额外的启发式规则来检测那些不返回的函数，即如果一个基本块能够从之前识别的两个不同的函数中到达，则该基本块一定位于一个独立的函数中。该启发式规则只能推断出被多次调用的私有函数的边界，而那些只被调用了一次的私有函数便会被内联进调用函数，因此该启发式规则并不会影响Gigahorse反编译的精度。

实际上，这一启发式规则是递归的，即随着更多的函数被识别，更多的基本块也能够从之前识别的两个不同的函数中到达。

![20190414000938](/images/2019-04-18/20190414000938.png)

在FunctionEntry关系到达最小不动点，即不再识别新的函数之后，Gigahorse基于如下的算法生成了本地CFG与调用图。也就是说，函数推断能够帮助过滤掉之前构造的CFG中不满足函数抽象的错误边，提升CFG构造的精度。

![20190414001232](/images/2019-04-18/20190414001232.png)

### D. 推断函数参数与本地变量

Gigahorse通过计算被调用函数在执行过程中从栈中弹出的由调用函数压入的变量的个数来推断函数参数的个数，并通过类似的方式来推断函数的返回值的个数。

Gigahorse反编译的最后一个步骤就是推断本地变量以及与这些变量相关的数据流信息，并生成最终的函数三地址码，其方法与前述步骤类似，故在此不再赘述。

## 5 Implementation and Discussion

Gigahorse反编译器的实际Datalog规范具有更多的技术细节，其包含数百条逻辑蕴涵规则，并使用Soufflé作为其执行引擎。

Gigahorse能够以结构化的形式以及文本的形式产生输出。

![20190414005320](/images/2019-04-18/20190414005320.png)

Gigahorse被设计为一个智能合约分析框架，分析人员能够在其输出的函数三地址码的基础上进行进一步的程序分析。此外，Gigahorse还提供了用于分析反编译结果的API以及分析函数库。

## 6 Evaluation

文章的实验尝试去回答以下几个研究问题：

+ **RQ1：**可扩展性—Gigahorse是否是一个高效并且适用于全部合约的反编译器？
+ **RQ2：**完备性—反编译生成的中间代码对原始EVM字节码代码的覆盖程度如何？
+ **RQ3：**精确性—反编译生成的中间代码是否与原始EVM字节码的高层语义精确匹配？

### A. RQ1：可扩展性

![20190414011537](/images/2019-04-18/20190414011537.png)

Porosity与Vandal无法成功反编译的合约的体积都较大。

![20190414011627](/images/2019-04-18/20190414011627.png)

### B. RQ2：完备性

![20190414011703](/images/2019-04-18/20190414011703.png)

### C. RQ3：精确性

![20190414011735](/images/2019-04-18/20190414011735.png)
