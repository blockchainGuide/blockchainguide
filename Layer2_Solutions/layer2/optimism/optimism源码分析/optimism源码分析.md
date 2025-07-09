> https://godorz.info/2022/04/optimism-notes/
>
> https://github.com/ethereum-optimism/optimism-tutorial 社区基于optimism的开发教程
>
> layer2交易费对比 https://www.theblockbeats.info/news/29797



本文基于https://github.com/ethereum-optimism/optimism/commit/b1cc033b6d3827df47b606e3f89fdbf2fa2cccc4 撰写

> 学习目的，有意将此扩展方案适配到本公司基于以太坊更改的项目中以提高扩展性，更好的支持应用生态。

## Optimistic Rollup 概述

乐观汇总是一种第2层可扩展性技术，可在不牺牲安全性或分散性的情况下增加以太坊的计算和存储容量。交易数据在链上提交，但在链下执行。如果链外执行中存在错误，可以在链上提交错误证明，以纠正错误并保护用户资金。同样，除非有争议，否则你不会上法庭，除非有错误，否则你也不会在链上执行交易。

## 系统概览

Optimism 协议中的智能合约可以分为几个关键组件。我们将在下面更详细地讨论每个组件。

- 链：第 1 层上的合约，它持有第 2 层交易的顺序，以及对相关第 2 层状态根的承诺
- 验证：第 1 层上的合约，它实现了对交易结果提出质疑的过程。
- bridge：促进第 1 层和第 2 层之间消息传递的合约
- Predeploys：一组基本合约，在系统的创世状态下部署并可用。这些合约类似于以太坊的预编译，但它们是用 Solidity 编写的，并且可以在前缀为 0x42 的地址中找到。

## 链合约

该链由一组运行在以太坊主网上的合约组成。这些合约存储以下有序列表：

- 应用于 L2 状态的所有事务的有序列表。
- 提议的状态根将由每笔交易的应用产生。
- 从 L1 发送到 L2 的交易，等待包含在有序列表中。

该链由以下合约构成：

CanonicalTransactionChain (opens new window) (CTC)：

Canonical Transaction Chain (CTC) 合约是一个只能附加的交易日志，**必须应用于 OVM 状态**。它通过将交易写入链存储容器的 CTC:batches 实例来定义交易的顺序。 CTC 还允许任何帐户 enqueue() 一个 L2 交易，Sequencer 最终必须将其附加到汇总状态。

### [`StateCommitmentChain` (opens new window)](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L1/rollup/StateCommitmentChain.sol)(SCC)

状态承诺链 (SCC) 合约包含一个提议的状态根列表，提议者断言这些状态根是规范交易链 (CTC) 中每笔交易的结果。这里的元素与 CTC 中的交易是 1:1 对应的，应该是链下通过规范交易逐一应用计算出来的唯一状态根。

### [`ChainStorageContainer`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L1/rollup/ChainStorageContainer.sol)

以“环形缓冲区”数据结构的形式提供可重复使用的存储，它将覆盖不再需要的存储槽。部署了三个 Chain Storage Container，两个由 CTC 控制，一个由 SCC 控制。

### 验证合约

在上一节中，我们提到链包括每个交易产生的建议状态根列表。在这里，我们将更多地解释这些建议是如何产生的，以及我们如何信任它们。

简而言之：如果提议的状态根不是执行交易的正确结果，那么验证者（运行 Optimism“全节点”的任何人）可以发起交易结果挑战。如果交易结果被成功证明是错误的，Verifier 将从资金中获得奖励，Sequencer 必须将其作为保证金。

::: 注意该系统仍在编写中，因此这些细节可能会发生变化:::

### bondmanager合约

bondmanager合约以 ERC20 代币的形式处理来自担保提议者的存款。它还处理验证者在挑战过程中花费的 gas 成本的核算。如果挑战成功，有问题的提议者的保证金将被削减，验证者的 gas 费用将被退还

### bridge合约

主要实现了L1和L2消息传递。

### [`L1CrossDomainMessenger`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L1/messaging/L1CrossDomainMessenger.sol)

L1 Cross Domain Messenger (L1xDM) 合约将消息从 L1 发送到 L2，并将消息从 L2 中继到 L1。如果从 L1 发送到 L2 的消息因超过 L2 时期的气体限制而被拒绝，可以通过该合约的重放功能重新提交。

### [`L2CrossDomainMessenger`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/messaging/L2CrossDomainMessenger.sol)

L2 Cross Domain Messenger (L2xDM) 合约将消息从 L2 发送到 L1，并且是通过 L1 Cross Domain Messenger 发送的 L2 消息的入口点。

## 标准桥

消息传递的一种常见情况是在 L1 和 Optimism 之间“传输”ERC-20 代币或 ETH。为了将代币存入 Optimism，桥将它们锁定在 L1 上并在 Optimism 中铸造等价代币。为了提取代币，桥会销毁 Optimism 代币并释放锁定的 L1 代币。[更多细节在这里](https://community.optimism.io/docs/developers/bridge/standard-bridge/)

### [`L1StandardBridge`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L1/messaging/L1StandardBridge.sol)

标准桥的 L1 部分。负责完成 L2 的提款并开始将 ETH 和兼容的 ERC20 存入 L2。

### [`L2StandardBridge`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/messaging/L2StandardBridge.sol)

标准网桥的 L2 部分。负责完成 L1 的存款并启动 L2 的 ETH 和合规 ERC20 提款。

### [`L2StandardTokenFactory`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/messaging/L2StandardTokenFactory.sol)

用于创建与标准桥兼容并在标准桥上工作的 L1 ERC20 的标准 L2 令牌表示的工厂合同。请参阅此处了解更多信息（打开新窗口）。

### 预部署合约

“Predeploys”是一组基本的 L2 合约，它们在系统的创世状态下部署并可用。这些合约类似于以太坊的预编译，但它们是用 Solidity 编写的，可以在 OVM 中以 0x42 为前缀的地址找到。

在 Solidity 库 Lib_PredeployAddresses (opens new window) 以及 @eth-optimism/contracts 包中作为预部署导出可以查找预部署。

预先部署了以下具体合同：

### [`OVM_L1MessageSender`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/predeploys/iOVM_L1MessageSender.sol)

L1MessageSender 是在 L2 上运行的预部署合约。在执行从 L1 到 L2 的跨域交易期间，它返回通过 Canonical Transaction Chain 的 enqueue() 函数将消息发送到 L2 的 L1 帐户（EOA 或合约）的地址。

请注意，该合约不是用 Solidity 编写的。但是，上面链接的界面仍然可以正常工作。通过这种方式，它类似于 EVM 的预部署。

\#

### [`OVM_L2ToL1MessagePasser`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/predeploys/OVM_L2ToL1MessagePasser.sol)

L2 到 L1 消息传递器是一个实用程序合约，它有助于 L2 上消息的 L1 证明。 L1 Cross Domain Messenger 在其 _verifyStorageProof 函数中执行此证明，该函数验证此合约的 sentMessages 映射中交易哈希的存在

### [`OVM_SequencerFeeVault`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/L2/predeploys/OVM_SequencerFeeVault.sol)

该合约持有支付给定序器的费用，直到有足够的交易成本证明将它们发送到 L1 的交易成本是合理的，在那里它们被用来支付 L1 交易成本（主要是将所有 L2 交易数据作为 CALLDATA 在 L1 上发布的成本）。

### [`Lib_AddressManager`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/libraries/resolver/Lib_AddressManager.sol)

这是一个存储名称与其地址之间的映射的库。它由 L1CrossDomainMessenger 使用。







## sequencer

目前，Optimism 在 Optimism 上运行唯一的排序器。这并不意味着 Optimism 可以审查用户交易。然而，随着时间的推移，仍然希望将排序器去中心化，完全消除 Optimism 的作用，以便任何人都可以作为区块生产者参与网络。

目前sequencer只有一个，后面会通过经济机制和治理机制以一定频率轮换或者通过BFT来支持多个sequencer

[optimism spec仓库](https://github.com/ethereum-optimism/optimistic-specs)



## 如何工作的

Optimism 块存储在合约（CanonicalTransactionChain）的列表中（以太坊链上），（https://etherscan.io/address/0x5E4e65926BA27467555EB562121fac00D24E9dD2），他把这个合约叫做链，如果以太坊重组，Optimism也会重组，目前配置50个区块

## 区块的产生

sequencer 产生区块，它做以下事情：

- 提供即时交易确认和状态更新
- 构造并执行L2区块
- 向L1提交用户交易

sequencer没有内存池，事务按照接收顺序立即接受或拒绝。当用户将他们的交易发送到sequencer时，sequencer检查交易是否有效（即支付足够的费用），然后将交易应用到其本地状态作为pending块（多少笔交易作为一个块？？？？）。这些待处理的区块会定期以大批量提交给以太坊进行最终确定。该批处理过程通过将固定成本分摊到给定批内的所有交易上，显著降低了总交易费用。sequencer还应用了一些基本的压缩技术，以最小化发布到以太坊的数据量。

因为sequencer被赋予了对L2链的优先写访问权，所以sequencer可以提供一个强有力的保证，即一旦它决定了一个新的待决块，什么状态将被最终确定。结果，L2状态可以非常快速地可靠地更新。这样做的好处包括快速、即时的用户体验，以及近乎实时的Uniswap价格更新。

或者，用户可以完全跳过sequencer，通过以太坊交易将其交易直接提交到CanonicalTransactionChain。这通常更昂贵，因为提交此交易的固定成本完全由用户支付，并且不会在许多不同的交易中摊销。然而，这种替代（不发给sequencer，直接发到合约中）的提交方法有一个优点，即可以抵抗sequencer的审查。即使sequencer正在主动审查用户，用户也可以继续在Optimism上发送事务。

目前的sequencer 其实还是官方在操作，唯一一个能发布交易的角色。

## 区块的执行

optimism 节点直接从CanonicalTransactionChain合约中保存的块的附加列表中直接下载块, 这个列表中的区块是由sequencer提供的，然后全网的optimism节点去挑战这个区块数据.

Optimism 节点由两个主要组件组成，即以太坊数据索引器和 Optimism 客户端软件.即以太坊数据索引器(搜索上面合约中的区块数据，根据合约发出的事件）和Optimism客户端软件。以太坊数据索引器，也称为“数据传输层”（https://github.com/ethereum-optimism/optimism/tree/develop/packages/data-transport-layer）（或DTL），从发布到CanonicalTransactionChain合约的区块重建Optimism区块链。**DTL搜索由CanonicalTransactionChain发出的事件**，该事件表示已发布新的Optimism块。然后，它检查发出这些事件的事务，以标准以太坊块格式重建已发布的块（https://ethereum.org/en/developers/docs/blocks/#block-anatomy）。

Optimism节点的第二部分，Optimism客户端软件，是Geth的一个几乎完全普通的版本（https://github.com/ethereum/go-ethereum）。这意味着乐观主义几乎等同于以太坊。特别是，Optimism共享相同的以太坊虚拟机（https://ethereum.org/en/developers/docs/evm/），相同的帐户和状态结构（https://ethereum.org/en/developers/docs/accounts/），以及相同的燃气计量机制和收费时间表（https://ethereum.org/en/developers/docs/gas/）。我们将此架构称为“EVM等效”（https://medium.com/ethereum-optimism/introducing-evm-equivalence-5c2021deb306），这意味着大多数以太坊工具（即使是最复杂的工具）“只与乐观主义一起工作”。

乐观客户端软件不断监视DTL的新索引块。当索引新块时，客户端软件将下载并执行其中包含的交易。在客户端上执行交易的过程与以太坊上的交易相同：我们加载optimism状态，将交易应用于该状态，然后记录所得状态的变化。然后，针对由DTL索引的每个新块重复此过程。

我的理解，sequencer接收交易，组成区块扔到L1 合约上，optimism节点 监听合约上的区块，再下载下来根据以太坊的格式重组这些块，实际还会执行里面的交易，就是验证的过程。如果不对就可以挑战他。



## 在层之间桥接资产

Optimism的设计使得用户可以在**Optimism和以太坊上**的智能合约之间发送任意消息。这使得在两个网络之间转移包括ERC20代币在内的资产成为可能。这种通信发生的确切机制取决于消息发送的方向。

Optimism在标准网桥中使用此功能，允许用户将资产（ERC20s和ETH）从以太坊存入Optimism，并允许将资产从Optimism提取回以太坊。开发者如何使用标准桥（https://community.optimism.io/docs/developers/bridge/standard-bridge/）



## 从Ethereum转向Optimism

要从以太坊向Optimism发送消息，**用户只需在以太坊上触发CanonicalTransactionChain合约**，即可在Optimism块上创建新块。有关更多上下文，请参见上面关于块生产的章节。用户创建的块可以包括看起来来自生成块的地址的事务。

## 从optimism 回到 ethereum 

Optimism上的合约不可能像以太坊合约在Optimism中生成交易一样，在以太坊上轻松生成交易。因此，将数据从Optimism发送回以太坊的过程稍微复杂一些。我们必须能够对以太坊上的合约**做出关于乐观状态的可证明声明**，而不是自动生成经过验证的交易。

关于Optimism状态的可证明声明需要以Optimism的状态trie根（https://medium.com/@eiki1212/ethereum-state-trie-architecture-explained-a30237009d4e）的形式进行加密承诺（https://en.wikipedia.org/wiki/Commitment_scheme）。乐观主义的状态在每个区块之后都会更新，因此这种承诺也会在每个区块后发生变化。承诺定期发布（大约每小时一次或两次）到以太坊上名为StateCommitmentChain的智能合约（https://etherscan.io/address/0xBe5dAb4A2e9cd0F27300dB4aB94BeE3A233AEB19）



用户可以使用这些承诺来生成关于乐观状态的Merkle树证明（https://en.wikipedia.org/wiki/Merkle_tree）。这些证明可以通过以太坊上的智能合约进行验证。Optimism维护一个方便的跨链通信合同，即L1CrossDomainMessenger（https://etherscan.io/address/0x25ace71c97B33Cc4729CF772ae268934F7ab5fA1），它可以代表其他合同验证这些证明。

这些证明可用于在特定块高度上对Optimism上的任何合约的存储中的数据进行可验证的陈述。然后可以使用此基本功能使Optimism上的合约向以太坊上的合约发送消息。L2ToL1MessagePasser（https://explorer.optimism.io/address/0x4200000000000000000000000000000000000000）契约（预先部署到Optimism网络）可由Optimism上的契约用于在Optimism状态下存储消息。然后，用户可以向以太坊上的合约证明，Optimism上的给定合约实际上意味着通过显示该消息的哈希已存储在L2ToL1MessagePasser合约中来发送某些给定消息



## 故障证明

在乐观汇总中，状态承诺被发布到以太坊，没有任何直接证明这些承诺的有效性。相反，这些承诺在一段时间内被视为待定（称为“挑战窗口”）。如果提议的国家承诺在挑战窗口期（目前设定为7天）内未受到挑战，则该承诺被视为最终承诺。一旦承诺被视为最终承诺，以太坊上的智能合约可以安全地接受基于该承诺的乐观主义状态的证明

当状态承诺受到质疑时，可以通过“错误证明”(以前称为“欺诈证明”(https://github.com/ethereum-optimism/optimistic-specs/discussions/53))过程使其无效。如果该承诺成功地受到挑战，那么它将被从状态承诺链中删除，最终被另一个拟议的承诺所取代。值得注意的是，一个成功的挑战并不会让乐观主义本身倒退，而只会让乐观主义者对链条的状态做出公开的承诺。事务的顺序和乐观的状态是不变的，由一个错误证明挑战。

作为11月11日EVM等效（https://medium.com/ethereum-optimism/introducing-evm-equivalence-5c2021deb306）更新的副作用，目前正在对故障预防流程进行重大重新开发，因此在当前Optimism系统中不活跃。有关更多信息，请参阅Optimism的安全模型（https://community.optimism.io/docs/security-model/）。您还可以在本网站的“协议规范”部分中阅读更多关于一般故障证明过程的信息



## l2上面的交易fee

### L2执行费用 

> ```text
> l2_execution_fee = transaction_gas_price * l2_gas_used
> ```

可以在这个面板上查看 gasprice https://public-grafana.optimism.io/d/9hkhMxn7z/public-dashboard?orgId=1&refresh=5m

### L1 数据费用

Optimism 上的所有交易也都发布到以太坊，这部分数据费用由optimism 用户承担，也是主要的费用 ，基于以下4个因素：

- 以太坊当前的 gas price
- 将交易发布到以太坊的 gas 成本。这大致与**事务的大小**（以字节为单位）成比例。
- 以 gas 计价的固定间接费用。当前设置为 2100
- 一种动态管理费用，按固定数量按比例缩放 L1 费用。当前设置为 1.0。

> ```text
> l1_data_fee = l1_gas_price * (tx_data_gas + fixed_overhead) * dynamic_overhead
> 
> tx_data_gas = count_zero_bytes(tx_data) * 4 + count_non_zero_bytes(tx_data) * 16
> ```

具体的参数值在这个合约里：https://optimistic.etherscan.io/address/0x420000000000000000000000000000000000000F#readContract

同时L1的数据费用无法通过交易类型设置上限，但是最高多不超过25% （这里说明了为什么不超过25）https://help.optimism.io/hc/en-us/articles/4416677738907-What-happens-if-the-L1-gas-price-spikes-while-a-transaction-is-in-process

### 关键事

- 发送交易：
  - 在 Optimism 上发送交易的过程与在以太坊上发送交易的过程相同。发送交易时，您应提供大于或等于当前 L2 gas 价格的 gas 价格。与在以太坊上一样，您可以使用 eth_gasPrice RPC 方法查询此 gas 价格。同样，您应该以与在以太坊上设置交易气体限制相同的方式设置您的交易气体限制（例如通过 eth_estimateGas）。
- 响应价格更新
  - L2 上的 Gas 价格默认为 0.001 Gwei，但如果网络拥堵可以动态增加。发生这种情况时，网络将接受的最低费用会增加。与以太坊不同，Optimism 目前没有内存池来持有费用过低的交易。取而代之的是，Optimism 节点将拒绝交易，消息 Fee too low。您可能需要明确处理这种情况，并在发生这种情况时以新的汽油价格重试交易。
- 向用户显示费用
  - 许多以太坊应用程序通过将 gas 价格乘以 gas 限制来向用户显示预估费用。然而，如前所述，Optimism 的用户需要支付 L2 执行费和 L1 数据费。因此，您应该显示这两种费用的总和，以便为用户提供对交易总成本最准确的估计。
    - 您可以使用 SDK（https://github.com/ethereum-optimism/optimism-tutorial/tree/main/sdk-estimate-gas）。或者，您可以使用位于 0x420000000000000000000000000000000000000F（打开新窗口）的 GasPriceOracle 预部署智能合约估算 L1 数据费用。 GasPriceOracle 合约（打开新窗口）位于每个 Optimism 网络（主网和测试网）的相同地址。为此，调用 GasPriceOracle.getL1Fee()。

-------

## 和optimism合约交互

### 查找合约地址

- https://community.optimism.io/docs/useful-tools/networks/#api-options-2 
- [测试网部署的合约](https://github.com/ethereum-optimism/optimism/tree/master/packages/contracts/deployments/goerli)
- [合约的完整包](https://github.com/ethereum-optimism/optimism/tree/master/packages/contracts)

-----

## 运行本地开发环境



-----

## 组件







## 疑问

1. 标准网桥 和 自定义网桥的概念
2. 

> reference
>
> https://github.com/curryxbo/Arbitrum_Doc_CN。arbitrum 中文资料
>
> https://github.com/guoshijiang/layer2 最全面的layer2跨链学习资料
>
> https://community.optimism.io/docs/how-optimism-works/#moving-from-optimism-to-ethereum
>
> 



