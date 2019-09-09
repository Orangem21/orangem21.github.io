---
title: 'CKB脚本编程简介[3]: 自定义代币'
date: 2019-09-09 12:26:56
tags:
- Bitcoin
- Nervos

category:
- Nervos
---

CKB的 Cell 模型和VM支持许多新的用例。然而，这并不意味着我们需要抛弃现有的。如今区块链中的一个常见用途是令牌发行者发布具有特殊目的/意义的新令牌。在以太坊中，我们称之为ERC20令牌，让我们看看我们如何在CKB中构建类似的概念。为了与ERC20区分，在CKB中的令牌我们称之为 `user defined token` , 简称UDT。

本文使用CKB v0.20.0版本来演示，具体来说，我在每个项目中使用以下提交的版本:

* [ckb](https://github.com/nervosnetwork/ckb): 472252ac5333b2b19ea3ec50d54e68b627bf6ac5
* [ckb-duktape](https://github.com/nervosnetwork/ckb-duktape): 55849c20b43a212120e0df7ad5d64b2c70ea51ac
* [ckb-sdk-ruby](https://github.com/nervosnetwork/ckb-sdk-ruby): 1c2a3c3f925e47e421f9e3c07164ececf3b6b9f6

# 数据模型

与以太坊为每个合约账户提供了独特的存储空间不同，CKB是在多个 Cell 之间传递数据。Cell 的 Lock Sript 和 Type Sript 会告知 Cell 属于哪个帐户，以及如何与 Cell 进行交互。其结果是，与ERC20将所有令牌用户的余额存储在ERC20合约的存储空间中不同，在CKB中，我们需要一种新的设计来存储UDT用户的余额。

当然，我们可以构造一个特殊的 Cell 来保存所有UDT用户的余额。这个解决方案看起来很像以太坊的ERC20设计。但是出现了几个问题：

* 令牌发行者必须提供存储空间以保存所有用户的余额。随着用户数量的增长，存储空间也将增长，这在CKB的经济模型中，不是一个高效的设计。
* 考虑到更新CKB中的 Cell 实际上是在销毁旧 Cell 并重新生成新 Cell ，因此保存所有余额的单个 Cell 会产生瓶颈：需要更新UDT余额的每个操作都必须更新唯一的 Cell, 使用过程中将会产生冲突。

虽然有一些解决方案可以缓解甚至解决上述问题，但我们开始质疑这里的基本设计：将所有UDT保存在一个地方真的有意义吗？一旦转账，UDT应该真的属于接受者，为什么余额仍然保持在中心化的地方？

这引出了我们提出的设计：

1. 一个特殊的 Type Script 表示此 Cell 存储UDT。
2. Cell 数据的前4个字节包含当前 Cell 中的UDT数量。

这种设计有几个含义：

* UDT Cell 的存储成本始终是恒定的，与存储在 Cell 中的UDT数量无关。
* 用户可以将 Cell 中的全部或部分UDT转账给其他人。
* 实际上，可能有许多 Cell 包含相同数量的UDT。
* 用于保护UDT的 Lock Script 与UDT本身分离。

每个令牌用户将其UDT保存在自己的 Cell 中。他们负责为UDT提供存储空间，并确保他们自己的令牌是安全的。这样UDT就可以真正属于每个UDT用户。

这里还有一个问题：如果令牌存储在属于每个用户的众多 Cell 中而不是统一存储，我们如何确保令牌确实由令牌发行者创建？如果有人自己伪造代币怎么办？在以太坊中，这可能是一个问题，但正如我们将在本文中看到的，CKB中的 Type Script 可以防止所有这些攻击，确保您的令牌是安全的。

# 编写 UDT 脚本

鉴于上述设计，最小UDT Type Script 应该遵守以下规则：

* 在UDT交易交易中，输出 Cell 中的UDT总和应等于输入 Cell 中UDT的总和。
* 只有令牌发行者可以在初始令牌创建过程中生成新令牌。

这可能听起来有点雄心勃勃，但我们会看到，通过 Type Script 和一些CKB独特的设计模式，一切都可以搞定：P

为简单起见，我们将在纯JavaScript中编写UDT脚本，然而C的版本可能有助于节省 Cells ，但是功能将相同。

首先，我们需要遍历所有输入 Cell 并收集UDT的总和：

```
diff --git a/udt.js b/udt.js
index e69de29..4a20bd0 100644
--- a/udt.js
+++ b/udt.js
@@ -0,0 +1,17 @@
+var input_index = 0;
+var input_coins = 0;
+var buffer = new ArrayBuffer(4);
+var ret = CKB.CODE.INDEX_OUT_OF_BOUND;
+
+while (true) {
+  ret = CKB.raw_load_cell_data(buffer, 0, input_index, CKB.SOURCE.GROUP_INPUT);
+  if (ret === CKB.CODE.INDEX_OUT_OF_BOUND) {
+    break;
+  }
+  if (ret !== 4) {
+    throw "Invalid input cell!";
+  }
+  var view = new DataView(buffer);
+  input_coins += view.getUint32(0, true);
+  input_index += 1;
+}
```

正如前一篇文章中所解释的，CKB要求我们使用循环来迭代同一组中的所有输入 Cell 并获取数据。在C中我们将使用`ckb_load_cell_data`，它被包装到JS函数`CKB.raw_load_cell_data`中。正如ArrayBuffer所示，我们只对单元数据的前4个字节感兴趣，因为这4个字节将包含UDT的数量。

注意，这里我们对`input_coins`执行一个简单的add操作，这非常危险。我们这样做只是为了简单起见。在生产环境中，您应该检查该值是否将保持在32位整数值中。如果需要，应使用更高精度的数字类型。

同样，我们可以获取输出的UDT的总和并进行比较：

```
diff --git a/udt.js b/udt.js
index 4a20bd0..e02b993 100644
--- a/udt.js
+++ b/udt.js
@@ -15,3 +15,23 @@ while (true) {
   input_coins += view.getUint32(0);
   input_index += 1;
 }
+
+var output_index = 0;
+var output_coins = 0;
+
+while (true) {
+  ret = CKB.raw_load_cell_data(buffer, 0, output_index, CKB.SOURCE.GROUP_OUTPUT);
+  if (ret === CKB.CODE.INDEX_OUT_OF_BOUND) {
+    break;
+  }
+  if (ret !== 4) {
+    throw "Invalid output cell!";
+  }
+  var view = new DataView(buffer);
+  output_coins += view.getUint32(0, true);
+  output_index += 1;
+}
+
+if (input_coins !== output_coins) {
+  throw "Input coins do not equal output coins!";
+}
```

这几乎是验证第一条规则所需的全部内容：输出 Cell 中UDT的总和应等于输入 Cell 中UDT的总和。换句话说，现在使用这种 Type Script ，没有人能够伪造新的令牌。这不是很棒吗？

但有一个问题：当我们说没有人能够伪造新的代币时，我们真的意味着没有人，包括代币发行人！这可不太好，我们需要添加一个例外，因此令牌发行者可以先创建令牌，但之后没有人能够这样做。有没有办法做到这一点？

当然有！但答案就像一个谜语，所以请仔细阅读本段：Type Script 由2部分组成：表示实际代码的代码哈希，以及 Type Script 使用的参数。具有不同参数的2种 Type Script 将被视为2种不同 Type Script 。这里的技巧是允许令牌发行者创建一个具有新 Type Script 的 Cell ，没有人能够再次创建，所以如果我们在参数部分放置一些不能再包含的东西，那么问题就被解决了~

现在想想这个问题：什么东西不能包含在区块链中两次？交易输入中的OutPoint！我们第一次将OutPoint作为交易输入包含在内时，引用的 Cell 将被消耗，如果有人稍后再次包含它，则会产生双花错误，这正是我们使用区块链的原因。

我们现在有答案了！ CKB中最小UDT的 Type Script 完整验证流程如下：

1. 首先收集输入 Cell 中所有UDT的总和以及输出 Cell 中所有UDT的总和，如果它们相等，则 Type Script 将以成功状态退出。
2. 检查 Type Script 的第一个参数是否与当前交易中的第一个OutPoint匹配，如果它们匹配，则以成功状态退出。
3. 否则以失败状态退出。

您可以看出：步骤1对应于正常的UDT交易，而步骤2对应于初始令牌创建过程。

这就是我们所说的CKB独特的设计模式：通过使用输入OutPoint作为 Script 参数，我们可以创建一个无法再伪造的独特 Script：

1. 如果攻击者试图使用相同的参数，则 Script 将验证交易中的第一个输入OutPoint与参数不匹配，从而使交易无效;
2. 如果攻击者试图使用相同的参数并填充参数作为第一个输入OutPoint，它将创建一个双花错误，也会使交易无效;
3. 如果攻击者试图使用不同的参数，CKB将识别出不同的参数导致不同的 Type Script，从而生成不同的UDT。

这种简单而强大的模式确保UDT保持安全，同时享受它们可以在许多不同单元之间自由传输的好处。据我们所知，这种模式在其他许多声称灵活或可编程的区块链中是不可能实现的。

现在我们终于可以完成我们UDT脚本了：

```
diff --git a/contract.js b/contract.js
deleted file mode 100644
index e69de29..0000000
diff --git a/udt.js b/udt.js
index e02b993..cd443bf 100644
--- a/udt.js
+++ b/udt.js
@@ -1,3 +1,7 @@
+if (CKB.ARGV.length !== 1) {
+  throw "Requires only one argument!";
+}
+
 var input_index = 0;
 var input_coins = 0;
 var buffer = new ArrayBuffer(4);
@@ -33,5 +37,17 @@ while (true) {
 }
 
 if (input_coins !== output_coins) {
-  throw "Input coins do not equal output coins!";
+  if (!((input_index === 0) && (output_index === 1))) {
+    throw "Invalid token issuing mode!";
+  }
+  var first_input = CKB.load_input(0, 0, CKB.SOURCE.INPUT);
+  if (typeof first_input === "number") {
+    throw "Cannot fetch the first input";
+  }
+  var hex_input = Array.prototype.map.call(
+    new Uint8Array(first_input),
+    function(x) { return ('00' + x.toString(16)).slice(-2); }).join('');
+  if (CKB.ARGV[0] != hex_input) {
+    throw "Invalid creation argument!";
+  }
 }
```

就是这样，一共有53行代码或1372字节，我们就在CKB中完成了一个最小的UDT Type Script 。注意我在这里甚至没有使用压缩工具，使用任何不错的JS压缩工具，我们应该能够获得更紧凑的 Type Script 。当然了，这是一个可用于生产环境的 Type Script ，但它足以显示一个简单的 Type Script 足以处理CKB中的重要任务。

# 部署到CKB网络

我不喜欢[其他一些组织](https://hacks.mozilla.org/2019/09/debugging-webassembly-outside-of-the-browser/) 只向你展示一个视频和一个充满挑战性的帖子其中隐藏了他们如何做到这一点以及伴随的问题。我相信如果没有真正的代码和使用它的步骤，它将是无趣的。以下是如何在CKB上使用上述UDT脚本：

以防万一，这里是没有diff格式的完整UDT脚本：

```
$ cat udt.js
if (CKB.ARGV.length !== 1) {
  throw "Requires only one argument!";
}
var input_index = 0;
var input_coins = 0;
var buffer = new ArrayBuffer(4);
var ret = CKB.CODE.INDEX_OUT_OF_BOUND;
while (true) {
  ret = CKB.raw_load_cell_data(buffer, 0, input_index, CKB.SOURCE.GROUP_INPUT);
  if (ret === CKB.CODE.INDEX_OUT_OF_BOUND) {
    break;
  }
  if (ret !== 4) {
    throw "Invalid input cell!";
  }
  var view = new DataView(buffer);
  input_coins += view.getUint32(0, true);
  input_index += 1;
}
var output_index = 0;
var output_coins = 0;
while (true) {
  ret = CKB.raw_load_cell_data(buffer, 0, output_index, CKB.SOURCE.GROUP_OUTPUT);
  if (ret === CKB.CODE.INDEX_OUT_OF_BOUND) {
    break;
  }
  if (ret !== 4) {
    throw "Invalid output cell!";
  }
  var view = new DataView(buffer);
  output_coins += view.getUint32(0, true);
  output_index += 1;
}
if (input_coins !== output_coins) {
  if (!((input_index === 0) && (output_index === 1))) {
    throw "Invalid token issuing mode!";
  }
  var first_input = CKB.load_input(0, 0, CKB.SOURCE.INPUT);
  if (typeof first_input === "number") {
    throw "Cannot fetch the first input";
  }
  var hex_input = Array.prototype.map.call(
    new Uint8Array(first_input),
    function(x) { return ('00' + x.toString(16)).slice(-2); }).join('');
  if (CKB.ARGV[0] != hex_input) {
    throw "Invalid creation argument!";
  }
}
```

为了能在CKB上运行JavaScript，让我们首先在CKB上部署duktape：

```
pry(main)> data = File.read("../ckb-duktape/build/duktape")
pry(main)> duktape_tx_hash = wallet.send_capacity(wallet.address, CKB::Utils.byte_to_shannon(300000), CKB::Utils.bin_to_hex(duktape_data))
pry(main)> duktape_data_hash = CKB::Blake2b.hexdigest(duktape_data)
pry(main)> duktape_out_point = CKB::Types::CellDep.new(out_point: CKB::Types::OutPoint.new(tx_hash: duktape_tx_hash, index: 0))
```

首先，让我们创建一个包含1000000令牌的UDT：

```
pry(main)> tx = wallet.generate_tx(wallet.address, CKB::Utils.byte_to_shannon(20000))
pry(main)> tx.cell_deps.push(duktape_out_point.dup)
pry(main)> arg = CKB::Utils.bin_to_hex(CKB::Serializers::InputSerializer.new(tx.inputs[0]).serialize)
pry(main)> duktape_udt_script = CKB::Types::Script.new(code_hash: duktape_data_hash, args: [CKB::Utils.bin_to_hex(File.read("udt.js")), arg])
pry(main)> tx.outputs[0].type = duktape_udt_script
pry(main)> tx.outputs_data[0] = CKB::Utils.bin_to_hex([1000000].pack("L<"))
pry(main)> tx.witnesses[0].data.clear
pry(main)> signed_tx = tx.sign(wallet.key, api.compute_transaction_hash(tx))
pry(main)> root_udt_tx_hash = api.send_transaction(signed_tx)
```

如果我们再次尝试提交相同的交易，则双花将阻止我们伪造相同的令牌：

```
pry(main)> api.send_transaction(signed_tx)
CKB::RPCError: jsonrpc error: {:code=>-3, :message=>"UnresolvableTransaction(Dead(OutPoint(0x0b607e9599f23a8140d428bd24880e5079de1f0ee931618b2f84decf2600383601000000)))"}
```

无论我们如何尝试，我们都无法创建另一个想要伪造相同UDT令牌的 Cell 。

现在我们可以尝试将UDT转移到另一个帐户。首先让我们尝试创建一个输出UDT比输入UDT更多的UDT交易：

```
pry(main)> udt_out_point = CKB::Types::OutPoint.new(tx_hash: root_udt_tx_hash, index: 0)
pry(main)> tx = wallet.generate_tx(wallet2.address, CKB::Utils.byte_to_shannon(20000))
pry(main)> tx.cell_deps.push(duktape_out_point.dup)
pry(main)> tx.witnesses[0].data.clear
pry(main)> tx.witnesses.push(CKB::Types::Witness.new(data: []))
pry(main)> tx.outputs[0].type = duktape_udt_script
pry(main)> tx.outputs_data[0] = CKB::Utils.bin_to_hex([1000000].pack("L<"))
pry(main)> tx.inputs.push(CKB::Types::Input.new(previous_output: udt_out_point, since: "0"))
pry(main)> tx.outputs.push(tx.outputs[1].dup)
pry(main)> tx.outputs[2].capacity = CKB::Utils::byte_to_shannon(20000)
pry(main)> tx.outputs[2].type = duktape_udt_script
pry(main)> tx.outputs_data.push(CKB::Utils.bin_to_hex([1000000].pack("L<")))
pry(main)> signed_tx = tx.sign(wallet.key, api.compute_transaction_hash(tx))
pry(main)> api.send_transaction(signed_tx)
CKB::RPCError: jsonrpc error: {:code=>-3, :message=>"InvalidTx(ScriptFailure(ValidationFailure(-2)))"}
```

在这里，我们尝试发送另一个用户1000000 UDT，同时为发送者本身保留1000000 UDT，当然这应该会触发错误，因为我们正在尝试伪造更多令牌。但稍作修改，我们可以证明，如果您遵守总和验证规则，UDT转移交易是有效的：

```
pry(main)> tx.outputs_data[0] = CKB::Utils.bin_to_hex([900000].pack("L<"))
pry(main)> tx.outputs_data[2] = CKB::Utils.bin_to_hex([100000].pack("L<"))
pry(main)> signed_tx = tx.sign(wallet.key, api.compute_transaction_hash(tx))
pry(main)> api.send_transaction(signed_tx)
```

# 灵活规则

这里显示的UDT脚本仅作为示例，实际上，dapps可能更复杂并且需要更多功能。您还可以根据需要为UDT脚本添加更多功能，其中一些示例包括：

* 这里我们严格确保输出UDT的总和等于输入UDT的总和，但在某些情况下，仅仅确保输出UDT的总和不超过输入UDT的总和就足够了。换句话说，当不需要时，用户可以选择为空间 烧毁 UDT。
* 上述UDT脚本不允许在初始创建过程后发出更多令牌，但可能存在另一种类型的UDT，允许来自令牌发行者的增发。这当然也可以在CKB上运行，解决这个任务的实际方法，留在这里作为练习:)
* 在这里，我们将脚本限制为仅在初始令牌创建过程中创建一个 Cell ，也可以创建多个单元格以在初始令牌创建过程中分散使用。
* 虽然我们只在这里介绍ERC20，但ERC721也应该是完全可能的。

请注意这些只是一些例子，使用CKB脚本的实际方法是没有边界的。我们非常高兴看到CKB dapp开发者创造出让我们感到惊讶的使用CKB脚本的有趣用法。
