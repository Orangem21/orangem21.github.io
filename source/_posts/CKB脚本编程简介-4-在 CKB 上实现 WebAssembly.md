---
title: 'CKB脚本编程简介[4]: 在 CKB 上实现 WebAssembly'
date: 2019-10-31 19:14:56
tags:
- CKB
- WebAssembly
category:
- Nervos
---

自从我们选择使用 RISC-V 构建 CKB VM（Virtual Machine 虚拟机）以来，我们几乎每一天都会被问及这样一个问题：为什么不像别人那样在 WebAssembly 上构建你的虚拟机呢？

在这个选择的背后其实有非常多的原因，可能需要另一篇文章或者一次演讲才能解释完其中的原因。从根本上来说，有一个相当重要的原因：构建软件最主要的就是找到正确的抽象概念，我们相信对于无许许可的区块链而言 RISC-V 比 WebAssembly 是一个更好抽象概念。

当然 WebAssembly 相比于其他更高级的编程语言和第一代区块链虚拟机而言已经是一个巨大的进步了，但 RISC-V 的运行级别还要比 WebAssembly 低得多，这使得它非常适合用于那些希望未来在运行几十年的公有链。

但还有一个问题没有得到回答：目前区块链行业很大一部分人都押注在 WebAssembly 上，（可以说） 基于 WebAssembly 的 dapps 构建了一个很好的生态系统。那么 CKB 如何与之竞争呢？正如上面所提到的，RISC-V 实际上位于一个比 WebAssembly 更低的抽象级别上，我们可以移植现有的 WebAssembly 的程序，并直接在 CKB VM 上运行他们。通过这种方式，我们既可以享受 RISC-V 提供的灵活性和稳定性，同时也可以拥抱 WebAssembly 的生态系统。

在本文中，我们将展示如何在 CKB VM 中运行 WebAssembly 程序，我们还会展示通过这种方式运行（程序），会比直接使用 WebAssembly VM 具有更多的优势。

就我个人而言，虽然我相信 WebAssembly 会有一些有趣的特性来支持不同的用例，但是我不相信 WebAssembly 会在区块链领域创建一个更好的生态。环顾四周，在基于 WebAssembly 的区块链中构建 Dapp 可能只有两种成熟的选择：Rust 和 AssemblyScript。人们一直在吹嘘 WebAssembly 在单个抽象的 VM 中支持任意语言的能力（我个人拒绝将 WebAssembly 称为 low-level 的虚拟机），但是在这里创建一个真正的 DApp，只能在这两种语言内选择其一。我认为，如果将仅支持 2 种的编程语言的虚拟机称为良好的 VM 生态系统，那么我们可能会有不同的定义。当然还有一些[其他语言](https://github.com/tweag/asterius)也在迎头赶上，但它们还没有稳定到可以被认为是一个繁荣的生态系统的阶段。虽然一些[有趣的语言](https://github.com/forest-lang/forest-compiler)在基于 WebAssembly 的环境中具有潜力，但是还没有人注意到，并去支持它们。如果你仔细观察，就会发现，使用 WebAssembly 的两个不同的区块链项目之间是否可以彼此共享合约，这目前仍然是个问题。当然，有人可能会说：“嗯，这只是一个时间问题，随着时间的推移，更有活力的 WebAssembly 生态系统将会发芽。"但同样的论点也适用于任何地方：为什么随着时间的推移，RISC-V 的生态系统不会变得更好？

咆哮到此为止，现在我们只是假设，WebAssembly 确实拥有一个区块链生态系统，我们可以证明，其中两个被广泛使用的语言：AssemblyScript 和 Rust，都可以在 CKB VM 环境中得到支持。


# AssemblyScript

我相信没有比演示更能说明问题的了。所以，让我们试试官方的 AssemblyScript，然后在 CKB 上运行编译好的程序。我们将只使用 AssemblyScript 简介页面中的[官方示例](https://docs.assemblyscript.org/)：

```
$ cat fib.ts
export function fib(n: i32): i32 {
  var a = 0, b = 1;
    for (let i = 0; i < n; i++) {
        let t = a + b; a = b; b = t;
  }
  return b;
}
```

关于如何安装，请参考 AssemblyScripts 的文档。为了方便起见，我提供了一些步骤，您可以在这里复制粘贴。

```
$ git clone https://github.com/AssemblyScript/assemblyscript.git
$ cd assemblyscript
$ npm install
$ bin/asc ../fib.ts -b ../fib.wasm -O3
$ cd ..
```

这样我们就有了一个编译好的 WebAssembly 程序，我们可以调用一个名为 [wasm2c](https://github.com/WebAssembly/wabt/tree/master/wasm2c) 的程序将它编译成 C 语言的源文件，然后通过 RISC-V 编译器将它编译成 RISC-V 程序，并在 CKB VM 上运行。

我敢肯定你会说：这是一个黑客行为！它这里对 WASM 程序进行了反编译，然后使它可以运行，你这是在作弊。这个问题的答案是，是但又不是：

* 一方面，我是在作弊，但我要提出的问题是：我们应该关心的是最终的结果，如果结果足够好，我们为什么要关心这是否是作弊呢？另外，现代编译器已经足够复杂了，就像一个完全的黑盒，我们怎么能确定这个反编译会得到更糟糕的结果呢？
* 另一方面，这只是将 WebAssembly 转换为 RISC-V 的一种方法。还有许多其他方法都可以实现相同的结果。我们将在后面的重述部分再次讨论这一点。

启动 `wasm2c` 然后转换 WebAssembly 程序：

```
$ git clone --recursive https://github.com/WebAssembly/wabt
$ cd wabt
$ mkdir build
$ cd build
$ cmake ..
$ cmake --build .
$ cd ../..
$ wabt/bin/wasm2c fib.wasm -o fib.c
```

您将在当前目录中看到一对 `fib.c` 和 `fib.h` 文件，它们包含 WebAssembly 程序转换的结果，当编译和调用正确时，它们将实现与 WebAssembly 程序相同的功能。

我们可以使用一个小的包装器 C 文件来调用 WebAssembly 程序：

```
$ cat main.c
#include <stdio.h>
#include <stdlib.h>

#include "fib.h"

int main(int argc, char** argv) {
  if (argc < 2) return 2;

  u8 x = atoi(argv[1]);

  init();

  u8 result = Z_fibZ_ii(x);

  return result;
}
```

这只是从 CLI 参数中读取一个整数，在 WebAssembly 程序中调用 Fibonacci 函数，然后返回结果。让我们先来编译它：

```
$ sudo docker run --rm -it -v `pwd`:/code nervos/ckb-riscv-gnu-toolchain:xenial bash
(docker) $ cd /code
(docker) $ riscv64-unknown-elf-gcc -o fib_riscv64 -O3 -g main.c fib.c /code/wabt/wasm2c/wasm-rt-impl.c -I /code/wabt/wasm2c
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccfUDYhE.o: in function `__retain':
/code/fib.c:1602: undefined reference to `Z_envZ_abortZ_viiii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccfUDYhE.o: in function `i32_load':
/code/fib.c:42: undefined reference to `Z_envZ_abortZ_viiii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccfUDYhE.o: in function `f17':
/code/fib.c:1564: undefined reference to `Z_envZ_abortZ_viiii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/fib.c:1564: undefined reference to `Z_envZ_abortZ_viiii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccfUDYhE.o: in function `f6':
/code/fib.c:1011: undefined reference to `Z_envZ_abortZ_viiii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccfUDYhE.o:/code/fib.c:1012: more undefined references to `Z_envZ_abortZ_viiii' follow
collect2: error: ld returned 1 exit status
(docker) $ exit
```

如上所示，这里有一个报错。它告诉我们有一个 `Z_ENVZ_ABORTZ_VIII` 函数没有定义。让我们深入了解为什么会发生这种情况。

首先，让我们将原始的 WebAssembly 文件转换为可读的形式：

```
$ wabt/bin/wasm2wat fib.wasm -o fib.wast
$ cat fib.wast | grep "(import"
(import "env" "abort" (func (;0;) (type 2)))
```

那么问题来了，WebAssembly 可以导入外部函数，在调用的时候，提供了额外的功能，事实上，著名的 [WASI](https://wasi.dev/) 就是基于 `import` 功能实现的，后面我们会看到 `import` 可以用来实现更多基于 WebAssembly 的区块链虚拟机不可能实现的有趣功能。

现在，让我们尝试一个 abort 执行，来修复报错:

```
$ cat main.c
#include <stdio.h>
#include <stdlib.h>

#include "fib.h"

void (*Z_envZ_abortZ_viiii)(u32, u32, u32, u32);

void env_abort(u32 a, u32 b, u32 c, u32 d) {
  abort();
}

int main(int argc, char** argv) {
  if (argc < 2) return 2;

  u8 x = atoi(argv[1]);

  Z_envZ_abortZ_viiii = &env_abort;

  init();

  u8 result = Z_fibZ_ii(x);

  return result;
}
$ sudo docker run --rm -it -v `pwd`:/code nervos/ckb-riscv-gnu-toolchain:xenial bash
(docker) $ cd /code
(docker) $ riscv64-unknown-elf-gcc -o fib_riscv64 -O3 -g main.c fib.c /code/wabt/wasm2c/wasm-rt-impl.c -I /code/wabt/wasm2c
(docker) $ exit
```

当然，您可以在 CKB 上测试已编译好的 `fib_riscv64` 程序。但是，有一个小技巧，在[测试套件](https://github.com/nervosnetwork/ckb-vm-test-suite)中有一个简单的 CKB VM [二进制文件](https://github.com/nervosnetwork/ckb-vm-test-suite/tree/master/binary/src)，我们可以使用它来运行这个特定的程序。值得一提的是，这个 CKB VM 二进制文件的工作方式与 CKB 中的 VM 略有不同。在当前示例中测试 WebAssembly 程序已经足够了。但是为了测试正确的 CKB 脚本，您可能希望使用新构建的[独立调试器](https://github.com/nervosnetwork/ckb-standalone-debugger)，它遵循所有 CKB 的语义。后面的文章将解释调试器是如何工作的。

让我们在测试套件中编译二进制文件并运行程序：

```
$ git clone --recursive https://github.com/nervosnetwork/ckb-vm-test-suite
$ cd ckb-vm-test-suite
$ git clone https://github.com/nervosnetwork/ckb-vm
$ cd binary
$ cargo build --release
$ cd ../..
$ ckb-vm-test-suite/binary/target/release/asm64 fib_riscv64 5
Error result: Ok(8)
$ ckb-vm-test-suite/binary/target/release/asm64 fib_riscv64 10
Error result: Ok(89)
```

这里的报错稍微有点误导，二进制将把程序中的任何非零结果都视为错误。由于测试的程序返回斐波那契计算结果作为返回值，二进制会把返回值（很可能不是零）视为错误，但是我们可以看到实际的错误值包含正确的斐波那契值。

现在我们证明 AssemblyScript 程序确实可以在 CKB VM 上工作！我确信更复杂的程序可能会遇到需要单独调整的错误，但是您已经了解了整个流程，并且知道在发生错误时应该去哪里查找 :)


# Rust

我们已经在 AssemblyScript 部分看到了一些简单的例子。让我们在 Rust 部分尝试一些更有趣的东西：我们可以在 Rust 代码中实现一个完整的签名验证吗？

事实证明我们可以！但这远远超出了我们在本博客文章中所能包含的内容。我已经准备好了一个[演示项目](https://github.com/nervosnetwork/wasm-secp256k1-test)来展示这一点。它用纯 Rust 语言实现了使用 [secp256k1 库](https://github.com/paritytech/libsecp256k1)进行签名验证的过程。如果您按照文件中的说明进行操作，您应该可以重现以下具体步骤：

* 将复杂的 Rust 程序编译到 WebAssembly 中
* 将 WebAssembly 程序转换为 RISC-V
* 在 CKB 虚拟机上运行生成的 RISC-V 程序


# 优于 WebAssembly 

还有一件事我们需要提到：如果您检查了 [Rust secp256k1 演示库](https://github.com/nervosnetwork/wasm-secp256k1-test/tree/bindgen)的 `Bindgen` 分支，并尝试相同的步骤，您将遇到以下错误：

```
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o: in function `core::result::unwrap_failed':
/code/secp.c:342: undefined reference to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:344: undefined reference to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:344: undefined reference to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:347: undefined reference to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:350: undefined reference to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o:/code/secp.c:353: more undefined references to `Z___wbindgen_placeholder__Z___wbindgen_describeZ_vi' follow
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o: in function `i32_store':
/code/secp.c:56: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_set_nullZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o: in function `i32_load':
/code/secp.c:42: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_set_nullZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o: in function `i32_store':
/code/secp.c:56: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_set_nullZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:56: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_set_nullZ_vi'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /tmp/ccYMiL3C.o: in function `i32_load':
/code/secp.c:42: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_growZ_ii'
/riscv/lib/gcc/riscv64-unknown-elf/8.3.0/../../../../riscv64-unknown-elf/bin/ld: /code/secp.c:42: undefined reference to `Z___wbindgen_anyref_xform__Z___wbindgen_anyref_table_growZ_ii'
collect2: error: ld returned 1 exit status
```

按照 AssemblyScript 示例中的相同步骤，我们可以在 WebAssembly 文件中进行某些 `imports`：

```
$ wabt/bin/wasm2wat wasm_secp256k1_test.wasm -o secp.wat
$ cat secp.wat | grep "(import"
(import "__wbindgen_placeholder__" "__wbindgen_describe" (func $__wbindgen_describe (type 3)))
(import "__wbindgen_anyref_xform__" "__wbindgen_anyref_table_grow" (func $__wbindgen_anyref_table_grow (type 4)))
(import "__wbindgen_anyref_xform__" "__wbindgen_anyref_table_set_null" (func $__wbindgen_anyref_table_set_null (type 3)))
```

这些实际上是在 Rust [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) 中所需的绑定环境的函数。我们将继续努力提供与 CKB 环境兼容的绑定。但是现在让我们退一步想想：这里所需要的环境功能并不是 WebAsembly 标准的一部分。标准中要求的是，当找不到导入条目时，WebAssembly VM 将停止执行，并出现错误。为了实现不同的特性，不同的基于 WebAssembly 的区块链可能会在这里注入不同的导入。使得编写一个兼容不同区块链的 WebAssembly 程序变得困难。


然而，在 CKB 环境中，我们可以根据需要加入任何环境函数。因此，支持所有针对不同区块链的 WebAssembly 程序。更重要的是，我们可以使用 imports 来为现有的 WebAssembly 程序引入新的特性。由于导入函数是与 WebAssembly 程序一起提供的，因此 CKB 本身不需要做任何事情来支持它。所有的奇迹都发生在一个 CKB 脚本中。对于支持 WebAssembly 的区块链，这些环境函数最有可能是固定的，并且是共识规则的一部分。你不能随意引进新的。类似地，这种在 CKB 上基于转换的流程，将使得支持新的 WebAssembly 功能（如垃圾收集或者线程处理）变得更加容易，这实际上只是将所需的支持功能写进 CKB 脚本的问题，这意味着当 WebAssembly 虚拟机更新时，你不需要再等待 6 个月，等待进行下一个硬分叉后（才能实现这些新的功能）。


# 这是易于实现的

您可能会有一个疑问：“我明白了，您在 RISC-V 创建了 WebASSembly，但我也可以在 WebAssembly 再现 RISC-V！WebAssembly 是灵活的！"从某种意义上说，这是可行的，一旦一种语言或虚拟机超过了一定的灵活性，它就可以被用来构建许多（甚至是疯狂的）东西。[jslinux](https://bellard.org/jslinux/tech.html) 的第一个版本，甚至可以用纯 JavaScript 模拟了完整的 x86。但这个问题的另一面是，易于实施的。在 RISC-V 上构建 WebAssembly 感觉更加自然，因为 WebAssembly 被抽象到了更高的级别，具有许多高级特性。例如更高级别的控制流、垃圾回收等。另一方面，RISC-V 模拟了真正的 CPU 所能做的事情，它是在计算机内部运行的实际 CPU 之上的一个非常薄的层。因此，虽然这两个方向确实都是可能的，但在 RISC-V 上搭建的 WebAssembly ，某些功能是更容易实现的。而在WebAssembly 上实现 RISC-V，可能会遇到很多问题。

一个可供选择的例子是 EVM，EVM 多年来一直在提倡图灵完备，但可悲的事实是，它几乎不可能在 EVM 上构建任意复杂的算法：要么是编码部分太难，要么是 gas 的消耗将是不合理的。人们不得不想出各种方法来介绍 EVM 上的最新算法，当伊斯坦布尔硬分叉完成时，我们只能在 EVM 中使用 Blake2b 算法。但是许多其他算法呢？

所有这些都是我们选择 RISC-V 的理由：我们希望在这一代的 CPU 架构上找到最小的一层，而 RISC-V 是我们在确保安全性和性能的同时，能够在区块链世界中找到的最显然的可选择模型。任何不同的模型，比如 WebAssembly、EVM 等，都应该是 RISC-V模型之上的一层，可以通过 RISC-V 模型自然地实现。然而，另一个方向可能根本不会让人感觉如此顺畅。


# 扼要重述

在这里，我们演示了您可以在 CKB VM 上运行 WebAssembly 程序，但我们想要指出的是，此流程并不是完全没有问题。其中一个问题是性能，我们的初步测试表明，基于 WebAssembly 的 secp256k1 演示运行速度比直接编译到 CKB VM 的类似的基于 C 语言的实现慢了 30 倍。经过一些调查，我们认为这是由于以下问题：

* 由于 WebAssembly 中内存的工作方式，wasm2c 必须首先将数据段放在纯 C 数组的代码中，然后在引导时，分配足够的内存，然后执行字符串拷贝将数据复制到分配的内存中。对于 secp256k1 示例，这意味着程序的每次引导都必须复制 1MB 的预计算乘法表。结合我们的 RISC-V 程序现在使用 newlib 的事实，newlib 包含一个简洁的 memcpy 实现，它针对代码大小和速度进行了优化。这会大大降低程序的运行速度。
* 虽然 wasm2c 可以为更简单的程序提供良好的性能，但对于 secp256k1 这样复杂且高度优化的算法，这意味着转换层可能失去许多优化的可能，因此会比直接编译到 RISC-V 的直接实现更慢。

幸运的是，这里的问题是完全可以解决的。上面提到的工作流是我们可以将 WebAssembly 程序转换为 RISC-V 程序的一种方式，但它绝对不是实现这一目标的唯一方式。就像我们上面提到的，转换层阻碍了优化可能，如果我们引入现代编译器来释放这里所有可能的优化呢？

已经有人在通过 LLVM 将 WebAssembly 程序转换为源代码方面取得了[进展](https://github.com/wasmerio/wasmer/tree/master/lib/llvm-backend)。这里实现的性能确实很好。由于 LLVM9 现在已经[正式支持](https://riscv.org/2019/09/llvm-9-releases-with-official-risc-v-target-support-asm-goto-clang-9-and-more-vincy-davis-packt-pub/) RISC-V，因此完全可以更改代码，通过 LLVM 直接生成 RISC-V 程序集，而不是 x86_64 程序集。这样，我们就可以通过 LLVM 直接将 WebAssembly 程序转换为 RISC-V 程序，享受 LLVM 在我们的代码上执行的所有高级优化。

因此，在这篇文章中我们提出的目前的解决方案，是完全可能的，同时为许多现有的案例实现足够好的性能（例如，许多类型的脚本为了安全可以用 Rust 编写，这样性能就不再是一个大问题），这种新的 LLVM 解决方案可以在未来为相同的流程提供更好的性能。而实现这些，对我们而言只是时间问题。
