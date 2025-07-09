> 文章参考官方文档，对官方文档进行了解读，并对主要内容进行了记录
>
> https://developer.offchainlabs.com/

- 词汇表： https://developer.offchainlabs.com/intro/glossary

## arbitrum 交易的生命周期

### 1. Sequencer接收交易

sequencer 从客户端接收交易并排序，它通过以下方式接收：

1. #####  Directly / Offchain

   - 对于L2原生dapp, 客户端会将他们的钱包连接到 L2 节点并直接交付签名交易

2. ##### or from L1 (via the Delayed Inbox)

   - 或者客户端可以通过在 Arbitrum 链的延迟收件箱中签署和发布 L1 交易来向sequencer发送消息。此功能最常用于通过桥存入 ETH 或代币

**See**:

- [Retryables](https://developer.offchainlabs.com/arbos/l1-to-l2-messaging)
- [The Sequencer](https://developer.offchainlabs.com/sequencer)
- [Token Bridge](https://developer.offchainlabs.com/asset-bridging)

### 2.Sequencer 对离线交易排序

一旦接收交易，Sequencer会做以下事情：

- 在链下收件箱中订购
- 使用 Arbitrum Nitro VM 在本地执行（包括收集/分配 L1 和 L2 费用等）
- “即时”向客户提供交易收据（“即时”是因为它不需要任何额外的链上确认，通常不会超过一两秒）

**See**:

- [ArbOS](https://developer.offchainlabs.com/arbos/)
- [Geth](https://developer.offchainlabs.com/arbos/geth)
- [L1 pricing](https://developer.offchainlabs.com/arbos/l1-pricing) / [L2 Gas](https://developer.offchainlabs.com/arbos/gas)

在此阶段，客户端对最终性的接受依赖于对 Sequencer 的信任。即，恶意/错误的 Sequencer 可能会偏离它在交易收据中承诺的内容和最终批量发布的内容（参见第 3 阶段）

> 即使是恶意/有故障的 Sequencer 也只能在最坏的情况下重新排序或暂时延迟交易；例如，它不能伪造客户的交易或提出无效的状态更新。考虑到第 2 阶段对 Sequencer 的信任程度，我们有时将 Sequencer 提供的“即时”收据称为“软确认”。

### 3. Sequencer 批量发布交易（链上）

Sequencer 最终会发布一批 L2 交易，其中包括我们客户的交易到基础 L1（作为CALLDATA）；在正常情况下，Sequencer 将每隔几分钟发布一次批次。

**3a: 如果 Sequencer 从未包含我们的交易怎么办？**

即使 Sequencer 从未将我们的交易包含在一个批次中，客户也可以通过在延迟的收件箱中发布并将其包含在 L2 中，然后在一段延迟时间（目前在 Arbitrum One 上大约 24 小时）后“强制包含”它。

Sequencer 被迫按照它们出现在链上的排队顺序包含**来自延迟收件箱的消息**，即它使用“先进先出”方法处理消息。因此，它不能在包含其他消息的同时选择性地延迟特定消息；即，延迟队列前面的消息意味着也延迟它后面的所有消息。

**See:**

- ["The Sequencer / Censorship Resistance."](https://developer.offchainlabs.com/sequencer)

在这个阶段，假设客户相信至少有一个行为良好的活跃 Arbitrum 验证者（回想一下在 Arbitrum Rollup 中，验证是无需许可的），客户可以将他们的交易的最终性视为等同于普通的以太坊交易。换句话说，他们的 L2 交易与批量记录它的 L1 交易具有相同的最终性。这意味着客户应该使用他们用于常规以太坊交易的任何最终性启发式（即等待 L1 块确认等），应用于 L1 批量发布交易。这也意味着客户对 Sequencer 的软确认（第 2 阶段）的信任模型感到不舒服，可以简单地等待 Sequencer 批量发布他们的交易（第 3 阶段）

- 一旦 Sequencer 发布了一个 batch，它的交易顺序完全由 L1 决定；实际上，Sequencer 在我们的交易生命周期中根本没有发言权。
- L1 上的收件箱合约确保当 Sequencer 发布批次时，它发布的数据足以让任何 Arbitrum 节点重建和验证 L2 链的状态；即，此“输入”数据的可用性由以太坊本身保证。
- Arbitrum 上的执行是完全确定的；即，当前链状态和新输入数据足以计算新链状态；因此，一旦此输入数据可用（即，当 Sequencer 发布批次时），就可以计算 L2 链的状态。
- Arbitrum防错系统完善；也就是说，如果任何验证者（后来）试图偏离有效的 L2 状态，诚实的验证者最终将能够挑战并获胜。因为我们已经知道有效状态最终会胜出，所以我们现在可以将我们的交易视为 L1 最终确定的。

### 4.Validator 断言包含交易的 RBlock

然后，一个抵押的、活跃的验证者将在收件箱中的输入上运行 Arbitrum VM（就像 Sequencer 之前所做的一样，除了现在只在 L1 上发布的交易上）并对链的最新状态做出链上断言，即汇总块或“RBlock”。 RBlocks 通常每 30-60 分钟被断言一次。

> RBlock 断言包括关于发件箱状态的声明；如果我们的交易触发了任何 L2 到 L1 消息，RBlock 将包含对发件箱的更新以反映其包含。

**See**:

- [The Outbox](https://developer.offchainlabs.com/arbos/l2-to-l1-messaging)

4a:RBlock 有效/不受挑战

在快乐/常见的情况下，验证者断言了一个有效的 RBlock，并且在争议窗口的过程中——在 Arbitrum One 上 1 周——没有其他验证者质疑它。

4b:断言受到挑战！

如果两个验证者断言不同的 RBlocks，则只有（至多）其中一个可以有效，因此他们会陷入争议。

争议包括两个质押验证者将他们的分歧分解为单个 L2 块，然后将该块内的 VM 指令序列分解为单个操作码，最后执行该单个操作。 Arbitrum 使用的底层 VM 是 WebAssembly (Wasm)，或者更准确地说，是“WAVM”。这都由 L1 上的合约引用。

**See:**

- [Challenges](https://developer.offchainlabs.com/proving/challenge-manager)
- [Wasm/WAVM](https://developer.offchainlabs.com/proving/wasm-to-wavm)

L1 合约还跟踪所有断言树；即，有多少利益相关者存在分歧，目前谁在与谁争论等等。我们将 Arbitrum 设计架构的这一级别称为其“断言树协议”。

**See:**

- [Assertion Tree Protocol](https://developer.offchainlabs.com/assertion-tree)

还记得在阶段 3 中说过一旦 L1 承诺输入，我们就可以保证 L2 输出吗？我们是认真的！即使在争议期间，Arbitrum 节点也会继续执行，并且活跃的验证者会继续对状态树中的有效叶子做出断言；在第 4 阶段可能发生的任何事情都不会影响我们在第 3 阶段已经锁定的 L1 级最终确定性。

### 5.RBlock 在 L1 上被确认

一旦所有争议都得到解决并且经过了足够的时间，我们的 RBlock 就可以在 L1 上得到确认（L1 上的任何以太坊账户都可以确认）。确认后，L1 上的发件箱根目录得到更新。

甚至在阶段 5 之前，客户端对他们的 L2 到 L1 消息的结果具有 L1 最终确定性，他们只是还不能执行它；也就是说，他们可以保证他们最终能够，例如，完成他们的提款，他们只是不能在 RBlock 被确认之前在 L1 上领取他们的资金。



