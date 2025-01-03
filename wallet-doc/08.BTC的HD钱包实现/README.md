# BTC的HD钱包实现

## 比特币的特性

* utxo与账户模型

## 比特币的版本迭代

### 版本升级介绍

* 2012年，BIP-0016升级，支持复杂脚本，支持多签，使得比特币的交易更加灵活和安全，地址格式为P2SH（Pay-to-Script-Hash），使用验证脚本来验证
* 2015年，BIP-0066 规范了交易中DER格式的签名，解决了交易签名的一致性问题，增强了网络的安全性。
* 2016年CheckSequenceVerify（CSV），BIP-0112 增加了相对时间锁功能，使得交易可以在指定的时间或块高度之后才生效，这为更复杂的支付通道铺平了道路。
* 2017年**Segregated Witness**（SegWit）即隔离见证，BIP-0141, BIP-0143,
  BIP-0144，分离交易签名数据，提高了区块的有效容量，减小了交易体积，降低了交易费用。还解决了交易的可塑性问题，使得闪电网络等侧链解决方案成为可能。
* 2017年隔离见证2x（SegWit2x），这次升级是对 SegWit 的后续提案，旨在进一步增加区块大小。然而，由于社区共识未达成，这次升级最终未能实现。
* 2021年**Taproot**，BIP-0340, BIP-0341, BIP-0342，引入了 Schnorr
  签名和默克尔化抽象语法树（MAST），增强了隐私性和执行脚本验证的功能，进一步提高了交易的灵活性和效率。这是自2017年 SegWit
  以来最重要的一次升级。催生rgb和铭文
* 2021年MAST（Merkelized Abstract Syntax Trees），作为 Taproot 的一部分，允许将多种条件的智能合约合并为一个，使得只有满足条件的部分才被公开，提高了隐私性和效率。
* 2021年**Schnorr Signatures**，作为 Taproot 的一部分，提供了一种更高效和安全的签名算法，允许多重签名聚合，提高了交易的隐私性和可扩展性。

### 比特币隔离见证升级为什么会减少手续费【减少交易体积减少手续费】

1. 减小交易大小：SegWit 将交易中的签名数据从主区块中分离出来，放入一个独立的见证数据区。虽然签名仍然存在，但它不再占据主区块中的空间。因此，整体上，SegWit
   交易的有效大小减小了，减少了对区块空间的占用。
2. 引入“虚拟字节”概念：SegWit 使用了虚拟大小（vsize）计算交易的字节数。即使见证数据仍然需要传输，但它对区块大小的影响较小，因为见证数据的权重为
   1，而非见证部分的权重为 4。通过这种方式，相同体积的交易数据实际占用的区块空间更少。
3. 提高交易吞吐量：SegWit 允许更多交易被打包进同一个区块中，提高了比特币网络的交易处理效率。由于区块能容纳更多的交易，需求压力降低，用户支付的手续费可以因此降低。
4. 减少交易拥堵：SegWit 优化了区块使用，降低了交易的体积，减少了区块链拥堵现象。在区块空间较为充裕时，用户可以用更低的手续费成功广播交易。

### 不同地址及引入原因

1. P2PKH（Pay to Public Key Hash）地址
   开头：以 1 开头。
   引入：比特币网络从一开始就使用的地址格式，源自比特币白皮书的设计。
   作用：这种地址代表了对公钥哈希（Public Key Hash, PKH）的支付，属于经典的交易脚本形式。通过 P2PKH 地址支付时，需要提供公钥和签名来解锁资金。
   示例地址：`1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa`（比特币创世块奖励地址）
   使用地址协议：BIP-44，（m/44'/0'/0'/0/index）
2. P2SH（Pay to Script Hash）地址
   开头：以 3 开头。
   引入：2012 年，比特币软分叉升级中的 BIP 16 提议引入了 P2SH 地址。
   作用：P2SH 允许用户通过复杂的条件（脚本）进行支付。它的主要用途是多重签名地址（multisig），同时也可以用作时间锁定交易等场景。用户将支付发送到脚本哈希（而不是公钥哈希），在花费资金时需要提供满足条件的解锁脚本。
   示例地址：`3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy`
   使用地址协议：BIP-49，（m/49'/0'/0'/0/index）
3. Bech32（SegWit 原生地址，也称为 P2WPKH 或 P2WSH）
   开头：以 bc1 开头。
   引入：2017 年，随着隔离见证（SegWit）的升级，BIP 173 提出了 Bech32 地址格式。
   作用：Bech32 是一种基于 SegWit 的地址类型，使用更紧凑的编码方式，可以减少交易数据的大小，从而降低手续费。SegWit
   地址有助于解决比特币交易的可延展性问题，并提升了网络的效率。此外，使用 SegWit 原生地址的交易手续费更低，因为交易数据占用的空间更少。Bech32
   地址还支持多种验证功能，减少了用户输入错误的概率。
   示例地址：`bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080`
   使用地址协议：BIP-84，（m/84'/0'/0'/0/index）
4. Taproot 地址（P2TR - Pay to Taproot）
   开头：以 bc1p 开头。
   引入：Taproot 升级是在 2021 年 11 月激活的，这是比特币一次重要的软分叉升级，BIP 341 和 BIP 342 共同构成了这一升级的主要内容。
   作用： 提升隐私性：Taproot 允许多签名交易（例如多方合作的交易）在链上表现得像普通的单签名交易，从而提高了隐私性。无论一个交易背后的条件多么复杂，Taproot 都可以将其隐藏起来，使其与普通交易无法区分。
   提高效率：Taproot 引入了 Schnorr 签名算法，它比之前的 ECDSA 签名更高效。Schnorr 签名支持批量验证多个签名，提高了交易验证效率，并减少了交易数据的大小。
   更灵活的智能合约支持：Taproot 允许更复杂的脚本条件和智能合约，但只有在必要时才会公开这些条件（通过 Taproot 中的 MAST 结构，即 Merkleized Abstract Syntax Tree）。这使得复杂交易在链上的数据更加简洁，且不需要在链上暴露所有条件。
   使用地址协议：BIP-86，（m/86'/0'/0'/0/index）
总之：

* P2PKH：基础的比特币支付地址，简单而广泛应用。 以 1 开头。
* P2SH：增强了比特币的脚本功能，支持多重签名、智能合约等复杂交易。 以 3 开头。
* Bech32（SegWit 原生地址）：优化了区块空间利用率，减少交易手续费，提升网络性能，解决了交易可延展性问题。以 bc1 开头。
* Taproot 地址（以 bc1p 开头）是在比特币 Taproot 升级中引入的，它通过 Schnorr 签名和 MAST 提升了隐私性、交易效率，并支持更灵活的脚本功能。以 bc1p 开头。

## UTXO模型

比特币使用的 **UTXO（未花费交易输出，Unspent Transaction Output）模型** 与账户模型（如以太坊的模型）相比，有其独特的优势。下面是
UTXO 模型的主要好处：

### 1. **增强隐私性**

在 UTXO 模型中，每笔交易的输出可以生成新的地址，用户可以将交易中的找零部分发送到不同的地址。这意味着同一个用户可以拥有多个地址，每个地址存储不同的
UTXO。这种特性使得跟踪资金流向更加困难，有助于保护用户隐私。

- **多个地址和 UTXO**：用户可以通过不断生成新地址来接收和存储比特币，从而提高匿名性和隐私性。
- **难以跟踪**：与账户模型中的余额变化不同，UTXO 模型中的输出是离散的、不可修改的，使得外部观察者难以判断资金的确切流动。

### 2. **并行性高，便于分片**

UTXO 模型可以让多个交易在不同的 UTXO 上并行处理，不会因为单一账户的余额更新而发生冲突。这使得 UTXO
模型在处理并行交易时有更高的效率，并更适合未来可能的分片设计或扩展方案。

- **并行性**：每个 UTXO 是独立的，多个交易可以在不同 UTXO 上同时进行，而不会影响其他交易的执行。
- **分片潜力**：UTXO 可以被轻松地分配到不同的区块链片段中处理，能够更有效地进行扩展。

### 3. **不可变性和安全性**

UTXO 模型中的输出是不可修改的，一旦 UTXO 被消费（花费），就不会再被使用。这种设计消除了修改交易历史的可能性，从而增强了比特币的安全性。

- **不可变性**：每个 UTXO 只能被消费一次，防止双花问题。
- **防篡改**：因为 UTXO 只能被消费，不能被修改，所以区块链上的交易记录更加安全且不可篡改。

### 4. **简化的状态管理**

在 UTXO 模型下，网络中的每个节点只需追踪未花费的输出，而不需要维护每个用户的账户余额状态。这简化了状态管理，节点只需存储
UTXO 集合，减少了内存和计算的负担。

- **轻量级状态**：只需追踪 UTXO，状态不需要像账户模型那样频繁更新和计算，从而减轻了区块链节点的存储和处理负担。
- **历史简洁**：UTXO 模型的账本结构较为简洁，因为每笔交易都可以追溯到之前的 UTXO 来源。

### 5. **防止双重支付问题**

UTXO 模型的不可变性使得双重支付问题更容易检测和防范。每个 UTXO 只能被消费一次，网络会拒绝那些尝试使用已消费 UTXO 的交易。

- **双花防护**：因为 UTXO 模型的设计，双花（double spending）攻击更容易被检测和阻止，保证了系统的安全性。
- **确定性消费**：一旦 UTXO 被花费，交易是不可逆的，链上记录清晰透明。

### 6. **灵活性强**

UTXO 模型支持复杂的脚本语言，可以为每个 UTXO 设定特定的花费条件。这为比特币的扩展性提供了强大的灵活性，支持多签名、时间锁等复杂的交易条件。

- **可编程性**：UTXO 允许在其上附加花费条件，通过比特币的脚本系统，可以实现条件支付、多重签名、智能合约等功能。
- **多签名和时间锁**：用户可以灵活设置多个参与者共同签名或设置一定时间之后才能花费 UTXO，从而增强资金管理的灵活性和安全性。

### 7. **交易验证简单**

UTXO 模型让每笔交易只需要验证输入是否未被花费即可，无需复杂的状态管理。这使得验证交易变得相对简单和高效，节点只需检查 UTXO
集合是否包含某个输入即可确认其有效性。

- **验证效率高**：每笔交易可以通过检查是否存在对应的 UTXO 来快速验证，而不需要像账户模型那样进行余额检查。
- **便于轻节点**：因为只需追踪 UTXO 集合，轻节点也能够快速验证交易的有效性，而不需要维护整个链上的状态。

### 8. **离线安全**

在 UTXO 模型中，由于交易可以离线生成，用户可以在不连接网络的情况下生成包含签名的交易，然后在线广播。这增强了资金管理的安全性，尤其适用于冷存储等场景。

- **离线签名**：用户可以生成离线交易，在没有连接网络的情况下确保私钥的安全。
- **冷存储支持**：UTXO 模型便于在冷存储中进行资金管理，防止私钥暴露在线攻击风险。

## 不同格式离线签名的生成

## 不同格式交易的验证过程

## 比特币扫链相关接口(Rosetta,rpc)

## electrumX和比特币节点结合，提供UTXO及交易记录
获取建议的手续费
* https://www.oklink.com/zh-hans/api-plans
* https://www.blockchain.com/explorer/api/blockchain_api

## Rosetta使用
* https://thewebthree.xyz/1/course_article?act_id=6

## 扫链
* 根据块儿高获取hash
  * developer.bitcoin.org/reference/rpc/getblockhash.html
* 获取交易信息
  * 
* 
