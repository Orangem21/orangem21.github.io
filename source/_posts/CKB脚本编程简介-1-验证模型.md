---
title: 'CKB脚本编程简介[1]: 验证模型'
date: 2019-07-07 20:16:20
tags:
- Bitcoin
- Nervos

category:
- Nervos
---

截至目前，CKB中的Cell验证模型或多或少已经趋于稳定，因此我将在这里开始写一系列文章来介绍CKB脚本编程。我的目标是补充在阅读白皮书后编写CKB脚本所需的所有缺失的细节实现，这样您就可以开始探索CKB呈现的这个美丽的仙境。

您可能会注意到我将在CKB上运行的代码称为`脚本`，而不是`智能合约`。这是因为智能合约对我来说是一个令人困惑的术语，我在这里想用另一个词来表示CKB独特的可编程性。CKB中的脚本不一定只是我们在脚本语言中看到的脚本，例如Ruby，JS，它实际上是指在CKB VM上运行的RISC-V格式二进制文件。

这第一篇文章将专门介绍CKB v0.14.0中引入的[全新验证模型](https://github.com/nervosnetwork/ckb/pull/913)。这可能听起来很无聊，但我保证这是最后一篇没有实际例子的帖子 :P

请注意，尽管我认为CKB的编程模型现在非常稳定，但目前仍然在进行开发，因此可能会有变化。我会尽力确保这篇文章更新，但如果有什么让你感到困惑的话，这篇文章现在正在描述CKB的这次[提交](https://github.com/nervosnetwork/ckb/commit/a02c675c50c5969a588fa7f6356f08861d8f5f92)。

## 概述

下面一张图片说明了CKB的真实交易过程：

![Transaction Example](/images/tx.svg)

这张图中有很多内容，我们将在稍后的文章中再次回到此图。今天，我们将只关注单元数据结构中的2个实体：`lock`和`type`。

```rust
pub struct CellOutput {
    pub capacity: Capacity,
    pub data: Bytes,
    pub lock: Script,
    #[serde(rename = "type")]
    pub type_: Option<Script>,
}
```

从数据结构中我们可以看到`lock`和`type`共享相同的结构，稍后我们可以证明它们也是在同一个环境中执行的，它们之间的差异只是在几个小细节中：

* `lock` 是必须的, while `type` 是可选项
* 通常, 他们用于不同的实例

我们首先从`type`脚本开始。

## type 脚本

请注意，注意这里的名字只是一个幸运的意外，它与心爱的[编程语言](https://www.typescriptlang.org/)无关.

如果你考虑一下，CKB（或大多数基于UTXO的区块链）上的交易只会将一组Cell（或UTXO）转换为另一组Cell。有趣的是，这里的实际转换过程。这就是我们开始设计CKB验证模型的地方：我们如何构建模型以更好地验证Cell 转换？

这就是`type`脚本的用武之地：`type`脚本用于验证 Cell 转换阶段的某些规则。这里的一些例子包括：

* 验证UDT（用户定义的Token）余额以确保不会无效地发出新Token。


## lock 脚本


## 运行模型

现在让我们看看是什么时候执行 lock 和 type 脚本的。

## 回到例子

这是我们之前看到的交易：

![Transaction Example](/images/tx.svg)

在图中，执行流程如下:

 1. `Lock Script 1` 执行一次。 
 2. `Lock Script 2` 执行一次。 
 3. `Type Script 1` 执行一次。 
 4. `Type Script 2` 执行一次。

在后面的文章中，我们可以看到 lock 和 type 脚本都在相同的环境中执行，并且都可以访问整个交易。如果任何一个脚本失败，整个交易就会失败。只有当所有脚本都成功时，交易才被认为是有效的。

有几点值得一提:

> 尽管有 2 个带有`Lock Script 1`的 input cell，但它只执行一次，由实际的 lock 脚本来定位具有相同 lock 脚本的所有 input cell，并验证两个签名。

> 在这个交易中只执行 input cell 中的 lock 脚本，例如：这里不执行`Lock Script 3`。

> 即使 input cell 和 output cell 都包含`Type Script 1`，也只执行一次。

> 在 input 和 output cell 中都会执行 type 脚本，其中包括`Type Script 1`和`Type Script 2`。

> 有些 cell 没有 type 脚本，在本例中我们只是省略了执行。

## 规则

现在我们总结一下规则:

> 在 input cell 中的 lock 脚本会被收集和解压，每个单独的 lock 脚本会被执行一次，并且只执行一次。

> input 和 output cell 中的 type 脚本(如果存在的话)会被收集在一起并解码，每个单独的 type 脚本都会被执行一次，并且只执行一次。

> 任何脚本失败，则整个交易验证失败。

## 下期预告

现在已经介绍了 cell 模型，我们将在下一篇文章中研究如何实际编写 CKB VM 脚本。将验证默认的 secp256k1 lock 脚本，来演示 CKB VM 脚本的使用周期。

