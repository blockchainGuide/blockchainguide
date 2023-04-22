> 本系列基于 **Pre-Bedrock** 分析

## 介绍

Optimism 是一种layer2可扩展性技术，可在不牺牲安全性或分散性的情况下增加以太坊的计算和存储容量。交易数据在链上提交但在链下执行。如果链下执行出现错误，可以在链上提交欺诈证明来纠正错误，保护用户资金。同样，除非有争议，否则你不会上法庭，除非出现错误，否则你不会在链上执行交易。 它利用其父链（ethereum）的共识机制（如 PoW 或 PoS）保证安全性，而不是提供自己的共识机制。

整体架构如下：

![image-20230305124747665](https://p.ipic.vip/1jtt0z.png)

## 组件

### L1组件

- **DepositFeed**
  - 合约`DepositFeed`发出`TransactionDeposited`事件，汇总驱动程序读取这些事件以处理存款。
  - *存款保证在排序窗口*内反映在 L2 状态中
  - 存入的是*交易*，而不是代币。然而，存款交易是实现代币存款的关键部分（代币在 L1 上锁定，然后通过存款交易在 L2 上铸造）
- **BatchInbox**：Batch Submitter 向其提交交易批次的 L1 地址。
  - 交易批次包括 L2 交易调用数据、时间戳和order信息。
  - BatchInbox 是一个常规的 EOA 地址。这让我们可以通过不执行任何 EVM 代码来节省 gas 成本
- **L2OutputOracle**：一种智能合约，存储[L2 输出根](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l2-output)以用于取款和故障证明。

## L2组件

- rollup 节点
  - 一个独立的、无状态的二进制文件。
  - 接收来自用户的 L2 交易。
  - 同步并验证 L1 上的汇总数据。
  - 应用特定于 rollup 的块生产规则从 L1 合成块。
  - 使用引擎 API 将块附加到 L2 链。
  - 处理 L1 重组。
  - 将未提交的块分发到其他汇总节点
- 执行引擎EE：
  - 修改过的Geth节点
  - 保持L2状态
  - 同步状态到其他L2节点
  - 将引擎 API 提供给rollup节点
- 批量交易提交者
  - [将交易批次](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)提交到地址的后台进程`BatchInbox`
- 输出提交器
  - 将 L2 输出承诺提交给`L2OutputOracle`.



sequencer 和 verifier 交互：

![image-20230307154045630](https://p.ipic.vip/xzrcao.png)

## 参与者

Optimistic Ethereum 中有三个参与者：用户、sequencer和验证者。

![image-20230305113538334](https://p.ipic.vip/5y5vfo.png)



### 用户

1. 可以直接向以太坊主网发送交易来存入或取出代币
2. 通过将交易发送到sequencer，在layer2 使用 EVM 智能合约
3. 使用网络验证者提供的区块浏览器查看交易状态

#### 入金

用户通常会通过从 L1 存入 ETH 来开始他们的 L2 旅程。一旦他们有 ETH 来支付费用，他们就会开始在 L2 上发送交易。下图演示了这种交互以及使用的所有关键 Optimistic Ethereum 组件：

![image-20230305124421674](https://p.ipic.vip/gposg6.png)

#### 提现

与存款一样重要的是，用户可以从汇总中退出是至关重要的。提款由 L2 上的正常交易发起，但在争议期结束后使用 L1 上的交易完成。

![image-20230305124335514](https://p.ipic.vip/cp0ip6.png)

### sequencer

块生产者，目前只支持单个sequencer。

1. 接受用户链下交易（公开`eth_sendRawTransaction`、验证费用……）
2. 观察链上交易（主要是来自 L1 的存款事件），由DTL扫描存储
3. 将两种交易合并到具有特定顺序的 L2 块中。
4. 将两个东西作为调用数据提交给 L1，将合并的 L2 块传播到 L1：
   - 用户在L2发送的pending链下交易
   - 关于链上事务的排序的足够信息，以从步骤3成功地重建块
5. sequencer还提供早在步骤 3 中访问块数据的权限，以便用户可以选择在 L1 确认之前访问实时状态

### 验证者

1. 向用户提供rollup数据
2. 验证rollup完整性并争论无效断言

![image-20230305120240964](https://p.ipic.vip/agbyde.png)

--------

## 区块存储

所有 Optimism 区块都存储在以太坊上一个特殊的智能合约中，称为[`CanonicalTransactionChain` ](https://etherscan.io/address/0x5E4e65926BA27467555EB562121fac00D24E9dD2)。Optimism 块保存在 CTC 内的一个列表中，这个列表形成了 Optimism 区块链，此列表只能添加。

包括`CanonicalTransactionChain`保证现有区块列表不能被新的以太坊交易修改的代码。然而，如果以太坊区块链本身被重组并且过去以太坊交易的顺序发生变化，这种保证可能会被打破。Optimism **主网被配置为能够抵抗多达 50 个以太坊区块的区块重组**。如果以太坊经历了比这更大的重组，Optimism 也会重组。optimism可以借助以太坊尽可能不进行重大重组的目标来获得一定的安全属性，从而抵御大型的区块重组。

## 区块产生

Optimism 区块生产主要由sequencer的单一方管理，它可以提供以下服务：

- 交易确认和状态更新
- 构造和执行L2块
- 向L1提交用户交易

sequencer会按照收到的顺序立即接受或拒绝并检查用户发送的交易是否有效。然后将交易作为待处理块应用于其本地状态。这些pending区块会定期大批量提交给以太坊（L1）进行最终确定。此批处理过程通过将固定成本分摊到给定批次中的所有交易来显着降低总体交易费用。sequencer还应用了一些基本的压缩技术来最小化发布到以太坊的数据量。

## 区块执行

Optimism 节点直接从合约`CanonicalTransactionChain`中保存的区块列表中下载区块。

Optimism 节点由两个主要组件组成，即以太坊数据索引器和 Optimism 客户端软件。以太坊数据索引器，也称为[“数据传输层” ](https://github.com/ethereum-optimism/optimism/tree/develop/packages/data-transport-layer)（或 DTL），从发布到CanonicalTransactionChain合约的区块中重建Optimism区块链。DTL搜索由CanonicalTransactionChain发出的事件，该事件表示已发布新的Optimism块。然后，它检查发出这些事件的事务，[以标准以太坊块格式重建已发布的块](https://ethereum.org/en/developers/docs/blocks/#block-anatomy)。

Optimism 节点的第二部分，即 Optimism 客户端软件。基于Geth，EVM、账户状态结构、gas计量机制和fee和以太坊一致，此为EVM等效，所以很多以太坊工具可以和Optimism一起使用。

Optimism 客户端软件持续监控新索引块的 DTL。当一个新块被索引时，客户端软件将下载它**并执行其中包含的交易**。在 Optimism 上执行交易的过程与在以太坊上相同：我们加载 Optimism 状态，针对该状态应用交易，然后记录由此产生s的状态变化。然后对 DTL 索引的每个新块重复此过程。



## 参考

[1] : [optimism protocol](https://community.optimism.io/docs/protocol/)

[2] : [evm Equivalence](https://medium.com/ethereum-optimism/introducing-evm-equivalence-5c2021deb306)

[3] : [spec](https://github.com/ethereum-optimism/optimistic-specs)





