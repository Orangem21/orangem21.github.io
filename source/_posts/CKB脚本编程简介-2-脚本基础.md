---
title: 'CKB脚本编程简介[2]: 脚本基础'
date: 2019-07-24 20:16:20
tags:
- Bitcoin
- Nervos

category:
- Nervos
---

上一篇我们介绍了当前 CKB 的验证模型。这一篇会更加有趣一点，我们要向大家展示如何将脚本代码真正部署到 CKB 网络上去。我希望在你看完本文后，你可以有能力自行去探索 CKB 的世界并按照你自己的意愿去编写新的脚本代码。

需要注意的是，尽管我相信目前的 CKB 的编程模型已经相对稳定了，但是开发仍在进行中，因此未来还可能会有一些变化。我将尽力确保本文始终处于最新的状态，但是如果在过程到任何疑惑，本文以[此版本下的 CKB](https://github.com/nervosnetwork/ckb/commit/80b51a9851b5d535625c5d144e1accd38c32876b) 作为依据。

警告：这是一篇很长的文章，因为我想为下周更有趣的话题提供充足的内容。所以如果你没有充足的时间，你不必马上完成它。我在试着把它分成几个独立的不凡，这样你就可以一次尝试一个。

## 语法

在继续之前，我们先来区分两个术语：脚本（script）和脚本代码（script code）

在本文以及整个系列文章内，我们将区分脚本和脚本代码。脚本代码实际上是指你编写和编译并在 CKB 上运行的程序。而脚本，实际上是指 CKB 中使用的脚本数据结构，它会比脚本代码稍微多一点点：

```
pub struct Script {
    pub args: Vec<Bytes>,
    pub code_hash: H256,
    pub hash_type: ScriptHashType,
    }
```

我们目前可以先忽略`hash_type`，之后的文章再来解释什么是`hash_type`以及它有什么有趣的用法。在这篇文章的后面，我们会说明`code_hash`实际上是用来标识脚本代码的，所以目前我们可以只把它当成脚本代码。那脚本还包括什么呢?脚本还包括`args`这个部分，它是用来区分脚本和脚本代码的。`args`在这里可以用来给一个 CKB 脚本提供额外的参数，比如：虽然大家可能都会使用相同的默认的 lock script code，但是每个人可能都有自己的 pubkey hash，`args` 就是用来保存 pubkey hash 的位置。这样，每一个CKB 的用户都可以拥有不同的 lock script ，但是却可以共用同样的 lock script code。

请注意，在大多数情况下，脚本和脚本代码是可以互换使用的，但是如果你在某些地方感到了困惑，那么你可能有必要考虑一下两者间的区别。

## 一个最小的 CKB 脚本代码

你可能之前就已经听所过了，CKB （编者注：此处指的应该是 CKB VM）是基于开源的 RISC-V 指令集编写的。但这到底意味着什么呢？用我自己的话来说，这意味着我们（在某种程度上）在 CKB 中嵌入了一台真正的微型计算机，而不是一台虚拟机。一台真正的计算机的好处是，你可以用任何语言编写任何你想写的逻辑。在这里，我们展示的前面几个例子将会用 C语言编写，以保持简单性（我是说工具链中的简单性，而不是[语言](http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html)），之后我们还会切换到基于 JavaScript 的脚本代码，并希望在本系列中展示更多的语言。记住，在 CKB 上有无限的可能！

正如我们提到的，CKB VM 更像是一台真正的微型计算机。CKB 的代码脚本看起来也更像是我们在电脑上跑的一个常见的 Unix 风格的可执行程序。

```
int main(int argc, char* argv[])
{
  return 0;
}
```

当你的代码通过 C 编译器编译时，它将成为可以在 CKB 上运行的脚本代码。换句话说，CKB 只是采用了普通的旧式 Unix 风格的可执行程序(但使用的是 RISC-V 体系结构，而不是流行的 x86 体系结构)，并在虚拟机环境中运行它。如果程序的返回代码是 0 ，我们认为脚本成功了，所有非零的返回代码都将被视为失败脚本。

在上面的例子中，我们展示了一个总是成功的脚本代码。因为返回代码总是 0。但是请不要使用这个作为您的 lock script code ，否则您的 token 可能会被任何人拿走。

但是显然上面的例子并不有趣，这里我们从一个有趣的想法开始:我个人不是很喜欢胡萝卜。我知道胡萝卜从营养的角度来看是很好的，但我还是想要避免它的味道。如果现在我想设定一个规则，比如我想让我在 CKB 上的 Cell 里面都没有以`carrot`开头的数据?让我们编写一个脚本代码来实现这一点。

为了确保没有一个 cell 在 cell data
中包含`carrot`，我们首先需要一种方法来读取脚本中的 cell data。CKB 提供了`syscalls`来帮助解决这个问题。

为了确保 CKB 脚本的安全性，每个脚本都必须在与运行 CKB 的主计算机完全分离的隔离环境中运行。这样它就不能访问它不需要的数据，比如你的私钥或密码。然而，要使得脚本有用，必须有特定的数据要访问，比如脚本保护的 cell 或脚本验证的事务。CKB 提供了`syscalls`来确保这一点，`syscalls`是在 RISC-V 的标准中定义的，它们提供了访问环境中某些资源的方法。在正常情况下，这里的环境指的是操作系统，但是在 CKB VM 中，环境指的是实际的 CKB 进程。使用`syscalls`， CKB脚本可以访问包含自身的整个事务，包括输入（inputs）、输出（outpus）、见证（witnesses）和 deps。


好消息是，我们已经将`syscalls`封装在了一个易于使用的头文件中，非常欢迎您在这里[查看这个文件](https://github.com/nervosnetwork/ckb-system-scripts/blob/66d7da8ec72dffaa7e9c55904833951eca2422a9/c/ckb_syscalls.h)，了解如何实现`syscalls`。最重要的是，您可以只获取这个头文件并使用包装函数来创建您想要的系统调用。

现在有了`syscalls`，我们可以从禁止使用`carrot`的脚本开始:

```
#include <memory.h>
#include "ckb_syscalls.h"

int main(int argc, char* argv[]) {
  int ret;
  size_t index = 0;
  volatile uint64_t len = 0; /* (1) */
  unsigned char buffer[6];

  while (1) {
    len = 6;
    memset(buffer, 0, 6);
    ret = ckb_load_cell_by_field(buffer, &len, 0, index, CKB_SOURCE_OUTPUT,
                                 CKB_CELL_FIELD_DATA); /* (2) */
    if (ret == CKB_INDEX_OUT_OF_BOUND) {               /* (3) */
      break;
    }

    if (memcmp(buffer, "carrot", 6) == 0) {
      return -1;
    }

    index++;
  }

  return 0;
}
```

以下几点需要解释一下：

 1. 由于 C 语言的怪癖，`len`字段需要标记为`volatile`。我们会同时使用它作为输入和输出参数，CKB VM 只能在它还保存在内存中时，才可以把它设置输出参数。而`volatile`可以确保 C 编译器将它保存为基于 RISC-V 内存的变量。
 
 2. 在使用`syscall`时，我们需要提供以下功能：一个缓冲区来保存`syscall`提供的数据；一个`len`字段，来表示系统调用返回的缓冲区长度和可用数据长度；一个输入数据缓冲区中的偏移量，以及几个我们在交易中需要获取的确切字段的参数。详情请参阅我们的[RFC](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0009-vm-syscalls/0009-vm-syscalls.md)。

 3. 为了保证最大的灵活性，CKB 使用系统调用的返回值来表示数据抓取状态:0 (or `CKB_SUCCESS`) 意味着成功，1 (or `CKB_INDEX_OUT_OF_BOUND`) 意味着您已经通过一种方式获取了所有的索引，2 (or`CKB_ITEM_MISSING`) 意味着不存在一个实体，比如从一个不包含该 type 脚本的 cell 中获取该 type 的脚本。

概况一下，这个脚本将循环遍历交易中的所有输出 cells，加载每个 cell data 的前6个字节，并测试这些字节是否和`carrot`匹配。如果找到匹配，脚本将返回`-1`，表示错误状态；如果没有找到匹配，脚本将返回`0`退出，表示执行成功。

为了执行该循环，该脚本将保存一个`index`变量，在每次循环迭代中，它将试图让 syscall 获取 cell 中目前采用的`index`值，如果 syscall 返回`CKB_INDEX_OUT_OF_BOUND`，这意味着脚本已经遍历所有的 cell，之后会退出循环；否则，循环将继续，每测试 cell data 一次，`index`变量就会递增一次。
  
这是第一个有用的 CKB 脚本代码！在下一节中，我们将看到我们是如何将其部署到 CKB 中并运行它的。

## 将脚本部署到 CKB 上

首先，我们需要编译上面写的关于胡萝卜的源代码。由于 GCC 已经提供了 RISC-V 的支持，您当然可以使用官方的 GCC 来创建脚本代码。或者你也可以使用我们准备的 [docker 镜像](https://hub.docker.com/r/nervos/ckb-riscv-gnu-toolchain)来避免编译 GCC 的麻烦:

```
$ ls
carrot.c  ckb_consts.h  ckb_syscalls.h
$ sudo docker run --rm -it -v `pwd`:/code nervos/ckb-riscv-gnu-toolchain:xenial bash
root@dc2c0c209dcd:/# cd /code
root@dc2c0c209dcd:/code# riscv64-unknown-elf-gcc -Os carrot.c -o carrot
root@dc2c0c209dcd:/code# exit
exit
$ ls
carrot*  carrot.c  ckb_consts.h  ckb_syscalls.h
```

就是这样，CKB 可以直接使用 GCC 编译的可执行文件作为链上的脚本，无需进一步处理。我们现在可以在链上部署它了。注意，我将使用 CKB 的 Ruby SDK，因为我曾经是一名 Ruby 程序员，当然 Ruby 对我来说是最自然的(但不一定是最好的)。如何设置请参考[官方 Readme 文件](https://github.com/nervosnetwork/ckb-sdk-ruby/blob/develop/README.md)。

要将脚本部署到 CKB，我们只需创建一个新的 cell，把脚本代码设为 cell data 部分:

```
pry(main)> data = File.read("carrot")
pry(main)> data.bytesize
=> 6864
pry(main)> carrot_tx_hash = wallet.send_capacity(wallet.address, CKB::Utils.byte_to_shannon(8000), CKB::Utils.bin_to_hex(data))
```

在这里，我首先要通过向自己发送 token 来创建一个容量足够的新的 cell。现在我们可以创建包含胡萝卜脚本代码的脚本:

```
pry(main)> carrot_data_hash = CKB::Blake2b.hexdigest(data)
pry(main)> carrot_type_script = CKB::Types::Script.new(code_hash: carrot_data_hash, args: [])
```

回忆一下脚本数据结构：

```
pub struct Script {
    pub args: Vec<Bytes>,
    pub code_hash: H256,
    pub hash_type: ScriptHashType,
    }
```

我们可以看到，我们没有直接将脚本代码嵌入到脚本数据结构中，而是只包含了代码的哈希，这是实际脚本二进制代码的 Blake2b 哈希。由于胡萝卜脚本不使用参数，我们可以对`args`部分使用空数组。

注意，这里仍然忽略了 `hash_type`，我们将在后面的文章中通过另一种方式讨论指定代码哈希。现在，让我们尽量保持简单。

要运行胡萝卜脚本，我们需要创建一个新的交易，并将胡萝卜 type 脚本设置为其中一个输出 cell 的 type 脚本:

```
pry(main)> tx = wallet.generate_tx(wallet2.address, CKB::Utils.byte_to_shannon(200))
pry(main)> tx.outputs[0].instance_variable_set(:@type, carrot_type_script.dup)
```

我们还需要进行一个步骤：为了让 CKB 可以找到胡萝卜脚本，我们需要在一笔交易的 deps 中引用包含胡萝卜脚本的 cell：

```
pry(main)> carrot_out_point = CKB::Types::OutPoint.new(cell: CKB::Types::CellOutPoint.new(tx_hash: carrot_tx_hash, index: 0))
pry(main)> tx.deps.push(carrot_out_point.dup)
```

现在我们准备签名并发送交易：

```
[44] pry(main)> tx.witnesses[0].data.clear
[46] pry(main)> tx = tx.sign(wallet.key, api.compute_transaction_hash(tx))
[19] pry(main)> api.send_transaction(tx)
=> "0xd7b0fea7c1527cde27cc4e7a2e055e494690a384db14cc35cd2e51ec6f078163"
```

由于该交易的 cell 中没有任何一个的 cell data 包含`carrot`，因此 type 脚本将验证成功。现在让我们尝试一个不同的交易，它确实含有一个以`carrot`开头的 cell：

```
pry(main)> tx2 = wallet.generate_tx(wallet2.address, CKB::Utils.byte_to_shannon(200))
pry(main)> tx2.deps.push(carrot_out_point.dup)
pry(main)> tx2.outputs[0].instance_variable_set(:@type, carrot_type_script.dup)
pry(main)> tx2.outputs[0].instance_variable_set(:@data, CKB::Utils.bin_to_hex("carrot123"))
pry(main)> tx2.witnesses[0].data.clear
pry(main)> tx2 = tx2.sign(wallet.key, api.compute_transaction_hash(tx2))
pry(main)> api.send_transaction(tx2)
CKB::RPCError: jsonrpc error: {:code=>-3, :message=>"InvalidTx(ScriptFailure(ValidationFailure(-1)))"}
from /home/ubuntu/code/ckb-sdk-ruby/lib/ckb/rpc.rb:164:in `rpc_request'
```

我们可以看到，我们的胡萝卜脚本拒绝了一笔生成的 cell 中包含胡萝卜的交易。现在我可以使用这个脚本来确保所有的 cell 中都不含胡萝卜!

所以，总结一下，部署和运行一个 type 脚本的脚本，我们需要做的是:

 1. 将脚本编译为 RISC-V 可执行的二进制文件  
 2. 在 cell 的 data 部分部署二进制文件
 3. 创建一个 type 脚本数据结构，使用二进制文件的 blake2b 散列作为`code hash`，补齐`args`部分中脚本代码的需要的参数
 4. 用生成的 cell 中设置的 type 脚本创建一个新的交易
 5. 将包含脚本代码的 cell 的 outpoint 写入到一笔交易的 deps 中去

这就是你所有需要的！如果您的脚本遇到任何问题，您需要检查这些要点。

虽然在这里我们只讨论了 type 脚本，但是 lock 脚本的工作方式完全相同。您惟一需要记住的是，当您使用特定的 lock 脚本创建 cell 时，lock 脚本不会在这里运行，它只在您使用 cell 时运行。因此， type 脚本可以用于构造创建 cell 时运行的逻辑，而 lock 脚本用于构造销毁 cell 时运行的逻辑。考虑到这一点，请确保您的 lock 脚本是正确的，否则您可能会在以下场景中丢失 token:

> 您的 lock 脚本有一个其他人也可以解锁您的 cell 的 bug。
> 您的 lock 脚本有一个 bug，任何人(包括您)都无法解锁您的 cell。

在这里我们可以提供的一个技巧是，始终将您的脚本作为一个 type 脚本附加到你交易的一个 output cell 中去进行测试，这样，发生错误时，您可以立即知道，并且您的 token 可以始终保持安全。

## 分析默认 lock 脚本代码

根据已经掌握的知识，让我们看看 CKB 中包含的默认的 lock 脚本代码。 为了避免混淆，我们正在查看 lock 脚本代码在 [这个commit](https://github.com/nervosnetwork/ckb-system-scripts/blob/66e2b3fc4fa3e80235e4b4f94a16e81352a812f7/c/secp256k1_blake160_sighash_all.c)。

默认的 lock 脚本代码将循环遍历与自身具有相同 lock 脚本的所有的 input cell，并执行以下步骤:

* 它通过提供的 syscall 获取当前的交易 hash

* 它获取相应的 witness 数据作为当前输入

* 对于默认 lock 脚本，假设 witness 中的第一个参数包含由 cell 所有者签名的可恢复签名，其余参数是用户提供的可选参数

* 默认的 lock 脚本运行 由交易 hash 链接的二进制程序的 blake2b hash， 还有所有用户提供的参数(如果存在的话)

* 将 blake2b hash 结果用作 secp256k1 签名验证的消息部分。注意，witness 数据结构中的第一个参数提供了实际的签名。

* 如果签名验证失败，脚本退出并返回错误码。否则它将继续下一个迭代。


注意，我们在前面讨论了脚本和脚本代码之间的区别。每一个不同的公钥 hash 都会产生不同的 lock 脚本，因此，如果一个交易的输入 cell 具有相同的默认 lock 脚本代码，但具有不同的公钥 hash(因此具有不同的 lock 脚本)，将执行默认 lock 脚本代码的多个实例，每个实例都有一组共享相同 lock 脚本的 cell。

现在我们可以遍历默认 lock 脚本代码的不同部分:

```
if (argc != 2) {
  return ERROR_WRONG_NUMBER_OF_ARGUMENTS;
}

secp256k1_context context;
if (secp256k1_context_initialize(&context, SECP256K1_CONTEXT_VERIFY) == 0) {
  return ERROR_SECP_INITIALIZE;
}

len = BLAKE2B_BLOCK_SIZE;
ret = ckb_load_tx_hash(tx_hash, &len, 0);
if (ret != CKB_SUCCESS) {
  return ERROR_SYSCALL;
}
```

当参数包含在 `Script`数据结构的 `args`部分， 它们通过 Unix 传统的`arc`/`argv`方式发送给实际运行的脚本程序。
为了进一步保持约定，我们在`argv[0]` 处插入一个伪参数，所以 第一个包含的参数从`argv[1]`开始。
在默认 lock 脚本代码的情况下，它接受一个参数，即从所有者的私钥生成的公钥 hash。

```
ret = ckb_load_input_by_field(NULL, &len, 0, index, CKB_SOURCE_GROUP_INPUT,
                             CKB_INPUT_FIELD_SINCE);
if (ret == CKB_INDEX_OUT_OF_BOUND) {
  return 0;
}
if (ret != CKB_SUCCESS) {
  return ERROR_SYSCALL;
}
```


使用与胡萝卜这个例子相同的技术，我们检查是否有更多的输入 cell 要测试。与之前的例子有两个不同:

* 如果我们只想知道一个 cell 是否存在并且不需要任何数据，我们只需要传入`NULL` 作为数据缓冲区，一个 `len` 变量的值是 0。
通过这种方式，syscall 将跳过数据填充，只提供可用的数据长度和正确的返回码用于处理。

* 在这个 carrot 的例子中，我们循环遍历交易中的所有输入， 但这里我们只关心具有相同 lock 脚本的输入cell。 CKB将具有相同锁定(或类型)脚本的`cell`命名为`group`。 我们可以使用 `CKB_SOURCE_GROUP_INPUT` 代替 `CKB_SOURCE_INPUT`， 来表示只计算同一组中的 cell，举个例子，即具有与当前 cell 相同的 lock 脚本的 cells。

```
len = WITNESS_SIZE;
ret = ckb_load_witness(witness, &len, 0, index, CKB_SOURCE_GROUP_INPUT);
if (ret != CKB_SUCCESS) {
  return ERROR_SYSCALL;
}
if (len > WITNESS_SIZE) {
  return ERROR_WITNESS_TOO_LONG;
}

if (!(witness_table = ns(Witness_as_root(witness)))) {
  return ERROR_ENCODING;
}
args = ns(Witness_data(witness_table));
if (ns(Bytes_vec_len(args)) < 1) {
  return ERROR_WRONG_NUMBER_OF_ARGUMENTS;
}
```

继续沿着这个路径，我们正在加载当前输入的 witness。 对应的 witness 和输入具有相同的索引。
现在 CKB 在 syscalls 中使用`flatbuffer`作为序列化格式，所以如果你很好奇，[flatcc的文档](https://github.com/dvidelabs/flatcc)是你最好的朋友。

```
/* Load signature */
len = TEMP_SIZE;
ret = extract_bytes(ns(Bytes_vec_at(args, 0)), temp, &len);
if (ret != CKB_SUCCESS) {
  return ERROR_ENCODING;
}

/* The 65th byte is recid according to contract spec.*/
recid = temp[RECID_INDEX];
/* Recover pubkey */
secp256k1_ecdsa_recoverable_signature signature;
if (secp256k1_ecdsa_recoverable_signature_parse_compact(&context, &signature, temp, recid) == 0) {
  return ERROR_SECP_PARSE_SIGNATURE;
}
blake2b_state blake2b_ctx;
blake2b_init(&blake2b_ctx, BLAKE2B_BLOCK_SIZE);
blake2b_update(&blake2b_ctx, tx_hash, BLAKE2B_BLOCK_SIZE);
for (size_t i = 1; i < ns(Bytes_vec_len(args)); i++) {
  len = TEMP_SIZE;
  ret = extract_bytes(ns(Bytes_vec_at(args, i)), temp, &len);
  if (ret != CKB_SUCCESS) {
    return ERROR_ENCODING;
  }
  blake2b_update(&blake2b_ctx, temp, len);
}
blake2b_final(&blake2b_ctx, temp, BLAKE2B_BLOCK_SIZE);
```

witness 中的第一个参数是要加载的签名，而其余的参数(如果提供的话)被附加到用于 blake2b 操作的交易 hash 中。

```
secp256k1_pubkey pubkey;

if (secp256k1_ecdsa_recover(&context, &pubkey, &signature, temp) != 1) {
  return ERROR_SECP_RECOVER_PUBKEY;
}
```

然后使用哈希后的 blake2b 结果作为信息，进行 secp256 签名验证。

```
size_t pubkey_size = PUBKEY_SIZE;
if (secp256k1_ec_pubkey_serialize(&context, temp, &pubkey_size, &pubkey, SECP256K1_EC_COMPRESSED) != 1 ) {
  return ERROR_SECP_SERIALIZE_PUBKEY;
}

len = PUBKEY_SIZE;
blake2b_init(&blake2b_ctx, BLAKE2B_BLOCK_SIZE);
blake2b_update(&blake2b_ctx, temp, len);
blake2b_final(&blake2b_ctx, temp, BLAKE2B_BLOCK_SIZE);

if (memcmp(argv[1], temp, BLAKE160_SIZE) != 0) {
  return ERROR_PUBKEY_BLAKE160_HASH;
}
```

最后同样重要的是，我们还需要检查可恢复签名中包含的 pubkey 确实是用于生成 lock 脚本参数中包含的 pubkey hash 的 pubkey。
否则，可能会有人使用另一个公钥生成的签名来窃取你的 token。

简而言之，默认 lock 脚本中使用的方案与现在[比特币中使用的方案](https://bitcoin.org/en/transactions-guide#p2pkh-script-validation)非常相似。

## 介绍 Duktape

我相信你和我现在的感觉一样: 我们可以用 C 语言写合约，这很好，但是 C 语言总是让人觉得有点乏味，而且，让我们面对现实，它很危险。
有更好的方法吗?

当然！ 我们上面提到的 CKB VM 本质上是一台微型计算机，我们可以探索很多解决方案。 我们在这里做的一件事是，使用 JavaScript 编写 CKB 脚本代码。 是的，你说对了，简单的 ES5 (是的，我知道，但这只是一个例子，你可以使用转换器) JavaScript。

这怎么可能呢? 由于我们有 C 编译器,我们只需为嵌入式系统使用一个 JavaScript 实现，在我们的例子中，[duktape](https://duktape.org/)
将它从 C 编译成 RISC-V 二进制文件，把它放在链上，我们就可以在 CKB 上运行 JavaScript 了!因为我们使用的是一台真正的微型计算机，所以没有什么可以阻止我们将另一个 VM 作为 CKB 脚本嵌入到 CKB VM 中，并在 VM 路径上探索这个 VM。

从这条路径展开，我们可以通过 duktape 在 CKB 上使用 JavaScript，我们也可以通过 [mruby](https://github.com/mruby/mruby)在 ckb 上使用 Ruby， 我们甚至可以将比特币脚本或EVM放到链上，我们只需要编译他们的虚拟机，并把它放在链上。这确保了 CKB VM 既能帮助我们保存资产，又能构建一个多样化的生态系统。所有的语言都应该在 CKB 上被平等对待，自由应该掌握在区块链合约的开发者手中。

在这个阶段，你可能想问: 是的，这是可能的，但是 VM 之上的 VM 不会很慢吗? 我相信这取决于你的例子是否很慢。我坚信，基准测试没有任何意义，除非我们将它放在具有标准硬件需求的实际用例中。 所以我们需要有时间检验这是否真的会成为一个问题。 在我看来，高级语言更可能用于 type scripts 来保护 cell 转换，在这种情况下，我怀疑它会很慢。此外，我们也在这个领域努力工作，以优化 CKB VM 和 VMs 之上的 CKB VM，使其越来越快，:P


要在 CKB 上使用 duktape，首先需要将 duktape 本身编译成 RISC-V 可执行二进制文件:

```
$ git clone https://github.com/nervosnetwork/ckb-duktape
$ cd ckb-duktape
$ sudo docker run --rm -it -v `pwd`:/code nervos/ckb-riscv-gnu-toolchain:xenial bash
root@0d31cad7a539:~# cd /code
root@0d31cad7a539:/code# make
riscv64-unknown-elf-gcc -Os -DCKB_NO_MMU -D__riscv_soft_float -D__riscv_float_abi_soft -Iduktape -Ic -Wall -Werror c/entry.c -c -o build/entry.o
riscv64-unknown-elf-gcc -Os -DCKB_NO_MMU -D__riscv_soft_float -D__riscv_float_abi_soft -Iduktape -Ic -Wall -Werror duktape/duktape.c -c -o build/duktape.o
riscv64-unknown-elf-gcc build/entry.o build/duktape.o -o build/duktape -lm -Wl,-static -fdata-sections -ffunction-sections -Wl,--gc-sections -Wl,-s
root@0d31cad7a539:/code# exit
exit
$ ls build/duktape
build/duktape*
```

与 carrot 示例一样，这里的第一步是在 CKB cell 中部署 duktape 脚本代码:

```ruby
pry(main)> data = File.read("../ckb-duktape/build/duktape")
pry(main)> duktape_data.bytesize
=> 269064
pry(main)> duktape_tx_hash = wallet.send_capacity(wallet.address, CKB::Utils.byte_to_shannon(280000), CKB::Utils.bin_to_hex(duktape_data))
pry(main)> duktape_data_hash = CKB::Blake2b.hexdigest(duktape_data)
pry(main)> duktape_out_point = CKB::Types::OutPoint.new(cell: CKB::Types::CellOutPoint.new(tx_hash: duktape_tx_hash, index: 0))
```

与 carrot 的例子不同，duktape 脚本代码现在需要一个参数: 要执行的 JavaScript 源代码:

```ruby
pry(main)> duktape_hello_type_script = CKB::Types::Script.new(code_hash: duktape_data_hash, args: [CKB::Utils.bin_to_hex("CKB.debug(\"I'm running in JS!\")")])
```

注意，使用不同的参数，你可以为不同的用例创建不同的 duktape 支持的 type script：

```ruby
pry(main)> duktape_hello_type_script = CKB::Types::Script.new(code_hash: duktape_data_hash, args: [CKB::Utils.bin_to_hex("var a = 1;\nvar b = a + 2;")])
```

这反映了上面提到的脚本代码与脚本之间的差异：这里 duktape 作为提供 JavaScript 引擎的脚本代码，而不同的脚本利用 duktape 脚本代码在链上提供不同的功能。


现在我们可以创建一个 cell 与 duktape 的 type script 附件:

```ruby
pry(main)> tx = wallet.generate_tx(wallet2.address, CKB::Utils.byte_to_shannon(200))
pry(main)> tx.deps.push(duktape_out_point.dup)
pry(main)> tx.outputs[0].instance_variable_set(:@type, duktape_hello_type_script.dup)
pry(main)> tx.witnesses[0].data.clear
pry(main)> tx = tx.sign(wallet.key, api.compute_transaction_hash(tx))
pry(main)> api.send_transaction(tx)
=> "0x2e4d3aab4284bc52fc6f07df66e7c8fc0e236916b8a8b8417abb2a2c60824028"
```

我们可以看到脚本执行成功，如果在`ckb.toml` 文件中将 `ckb-script`日志模块的级别设置为`debug`，你可以看到以下日志:

```bash
2019-07-15 05:59:13.551 +00:00 http.worker8 DEBUG ckb-script  script group: c35b9fed5fc0dd6eaef5a918cd7a4e4b77ea93398bece4d4572b67a474874641 DEBUG OUTPUT: I'm running in JS!
```

现在您已经成功地在 CKB 上部署了一个 JavaScript 引擎，并在 CKB 上运行基于 JavaScript 的脚本!
你可以在这里尝试认识的 JavaScript 代码。

## 一道思考题

现在你已经熟悉了 CKB 脚本的基础知识，下面是一个思考：
在本文中，您已经看到了一个 always-success 的脚本是什么样子的，但是一个 always-failure 的脚本呢?
一个 always-failure 脚本(和脚本代码)能有多小?

提示：这不是 gcc 优化比赛，这只是一个思考。

## 下集预告

我知道这是一个很长的帖子，我希望你已经尝试过，并成功地部署了一个脚本到 CKB。在下一篇文章中，我们将介绍一个重要的主题:如何在 CKB 定义自己的用户定义 token(UDT)。CKB 上 udt 最好的部分是，每个用户都可以将自己的 udt 存储在自己的 cell 中，这与 Ethereum 上的 ERC20 令牌不同，
在 Ethereum 上，每个人的 token 都必须位于 token 发起者的单个地址中。所有这些都可以通过单独使用 type script 来实现。
如果你感兴趣，请继续关注 :)

本文作者：Xuejie
原文链接：https://xuejie.space/2019_07_13_introduction_to_ckb_script_programming_script_basics/
本文译者：Shooter，Jason，Orange
