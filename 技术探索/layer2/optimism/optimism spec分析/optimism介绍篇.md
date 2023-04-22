Optimism 是一个 EVM 等效的乐观汇总协议，旨在扩展以太坊，同时保持与现有以太坊基础设施的最大兼容性。本文档概述了协议，为规范的其余部分提供上下文。



## 功能

### 扩展性

扩展以太坊意味着增加以太坊网络可以处理的有用交易的数量。以太坊有限的资源，特别是带宽、计算和存储，限制了网络上可以处理的交易数量。在这三种资源中，计算和存储是目前最显着的瓶颈。这些瓶颈限制了交易的供应，导致极高的费用。可以通过更好地利用带宽、计算和存储来实现扩展以太坊和降低费用。

### 什么是乐观rollup

Optimistic rollup 是一种第 2 层可扩展性技术，可在不牺牲安全性或分散性的情况下增加以太坊的计算和存储容量。交易数据在链上提交但在链下执行。如果链下执行出现错误，可以在链上提交错误证明来纠正错误，保护用户资金。同样，除非有争议，否则你不会上法庭，除非出现错误，否则你不会在链上执行交易。

### EVM 等效

EVM Equivalence 完全符合以太坊黄皮书描述的状态转换函数，协议的正式定义。通过在 EVM 等效汇总中符合以太坊标准，智能合约开发人员可以编写一次并部署到任何地方。

## 网络参与者

users, sequencers, and verifiers.

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203105146447.png" alt="image-20230203105146447" style="zoom:25%;" />

-----

user：

- 通过将数据发送到以太坊主网上的合约，在 L2 上存入或提取任意交易。
- 通过将交易发送到sequencer，在第 2 层使用 EVM 智能合约。
- 使用网络验证者提供的区块浏览器查看交易状态。

Sequencer:

Sequencer是主要的块生产者。可能有一个或多个使用共识协议的Sequencer。对于 1.0.0，只有一个排序器（目前在 Optimism Foundation 的监督下运行）。通常，规范可能会使用“定序器”作为由多个定序器操作的共识协议的替代术语。



🚩 这部分无法解释清楚🚩

1. 接受用户链下交易
2. 观察链上交易（主要是来自 L1 的存款事件）
3. 将两种交易合并到具有特定顺序的 L2 块中。
4. 通过将两个东西作为calldata提交给 L1，将合并的 L2 块传播到 L1：
   - 在步骤 1 中接受的pending链下交易。
   - 关于链上交易顺序的足够信息，以成功重建步骤 3 中的块，完全通过观察 L1

排序器还提供早在步骤 3. 中访问块数据的权限，以便用户可以选择在 L1 确认之前访问实时状态。

Verifiers:

- 向用户提供rollup数据；和
- 验证rollup完整性并争论无效断言。

为了让网络保持安全，必须**至少有一个诚实的验证者**能够验证汇总链的完整性并为用户提供区块链数据。

### 关键交互图

下图演示了在关键用户交互期间如何使用协议组件，以便在深入研究任何特定组件规范时提供上下文

存钱和发送交易：

用户通常会通过从 L1 存入 ETH 来开始他们的 L2 旅程。一旦他们有 ETH 来支付费用，他们就会开始在 L2 上发送交易。下图演示了这种交互以及所有已使用或应该使用的关键 Optimism 组件：

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203111801122.png" alt="image-20230203111801122" style="zoom:25%;" />

这个图中涉及到的组件链接

- [Rollup Node](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md)
- [Execution Engine](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md)

- [L2 Output Oracle](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2-output-oracle-smart-contract)
- [L2 Output Submitter](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#proposing-l2-output-commitments)

提款：

与存款一样重要的是，用户可以从rollup中退出是至关重要的。提款由 L2 上的正常交易发起，但在争议期结束后使用 L1 上的交易完成。

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203112021810.png" alt="image-20230203112021810" style="zoom:25%;" />

这个图中涉及到的组件链接

- [L2 Output Oracle](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2-output-oracle-smart-contract)