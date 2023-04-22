## 组件

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203112545498.png" alt="image-20230203112545498" style="zoom:25%;" />

### L1组件

- **DepositFeed**：源自 L1 状态下智能合约调用的 L2 交易的提要。
  - DepositFeed 合约发出 TransactionDeposited 事件，rollup驱动程序读取这些事件以处理存款。
  - 存款保证在排序窗口内反映在 **L2 状态中**
  - 请注意，交易是存入的，而不是代币。然而，存款交易是实现代币存款的关键部分（代币在 L1 上锁定，然后通过存款交易在 L2 上铸造）。
- **BatchInbox**：Batch Submitter 向其提交交易批次的 **L1 地址**。
  - 交易批次包括 L2 交易调用数据、时间戳和排序信息。
  - BatchInbox 是一个常规的 EOA 地址。这让我们可以通过不执行任何 EVM 代码来节省 gas 成本。

### L2组件

- rollup节点
  - 一个独立的、无状态的二进制文件。
  - 接收来自用户的 L2 交易。
  - 同步并验证 L1 上的rollup数据。
  - 应用特定于 rollup 的块生产规则从 L1 合成块。
  - 使用引擎 API 将块附加到 L2 链。
  - 处理 L1 重组。
  - 将未提交的块分发到其他rollup节点。
- 执行引擎
  - 一个普通的 Geth 节点，经过少量修改以支持 Optimism。
  - 保持 L2 状态。
  - 将状态同步到其他 L2 节点以实现快速入职。
  - 将引擎 API 提供给rollup节点。

### 批量提交

- 将交易批次提交到 BatchInbox 地址的后台进程。

### 输出提交器

- 向 L2OutputOracle 提交 L2 输出承诺的后台进程

## 交易区块广播

由于 EE 在底层使用 Geth，Optimism 使用 Geth 的内置点对点网络和交易池来传播交易。同一网络还可用于传播提交的块并支持快照同步。

然而，未提交的块将使用单独的 Rollup 节点对等网络传播。然而，这是可选的，并且是为了降低验证者及其 JSON-RPC 客户端的延迟而提供的。

下图说明了排序器和验证器如何组合在一起：

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203135658918.png" alt="image-20230203135658918" style="zoom:25%;" />

## 深入的关键交互

### 存款

Optimism支持两种类型的押金：用户押金和L1属性押金。为了执行用户存款，用户调用 DepositFeed 合约上的 depositTransaction 方法。这反过来会发出 TransactionDeposited 事件，rollup 节点会在区块派生期间读取这些事件。

L1 属性存款用于通过调用 L1 属性预部署在 L2 上注册 L1 块属性（数字、时间戳等）。它们不能由用户发起，而是由 rollup 节点自动添加到 L2 块中。

两种存款类型都由 L2 上的单个自定义 EIP-2718 交易类型表示。

### 区块推导

给定 L1 以太坊链，可以确定性地导出汇总链。整个 rollup 链可以基于 L1 块派生这一事实使 Optimism 成为 rollup。这个过程可以表示为：

```go
derive_rollup_chain(l1_blockchain) -> rollup_blockchain
```

Optimism 的区块推导函数是这样设计的：

- 除了可以使用 L1 和 L2 执行引擎 API 轻松访问的状态外，不需要任何状态。
- 支持sequencer和sequencer共识
- 对sequencer审查具有弹性

#### Epochs and the Sequencing Window

rollup 链被细分为 epoch。 L1 区块编号和纪元编号之间存在 1:1 的对应关系。

对于编号为 n 的 L1 区块，有一个相应的 rollup epoch n，它只能在经过一个排序窗口值的区块后得出，即在编号为 n + SEQUENCING_WINDOW_SIZE 的 L1 区块被添加到 L1 链之后。

每个纪元至少包含一个区块。 epoch 中的每个块都包含一个 L1 信息事务，其中包含有关 L1 的上下文信息，例如块哈希和时间戳。该时期的第一个区块还包含通过 L1 上的 DepositFeed 合约发起的所有存款。所有 L2 块还可以包含排序交易，即直接提交给排序器的交易。

每当定序器为给定的纪元创建新的 L2 块时，它必须在纪元的排序窗口内将其作为批次的一部分提交给 L1（即批次必须在 L1 块 n + SEQUENCING_WINDOW_SIZE 之前落地）。这些批次（连同 TransactionDeposited L1 事件）允许从 L1 链派生 L2 链。

排序器不需要将 L2 块批量提交到 L1 以在其之上构建。事实上，批次通常包含多个 L2 块价值的有序交易。这就是在排序器上实现快速交易确认的原因。

由于给定时期的交易批次可以在排序窗口内的任何位置提交，因此验证者必须在窗口内的所有块中搜索交易批次。这可以防止 L1 交易包含的不确定性。这种不确定性也是我们首先需要排序窗口的原因：否则排序器可能会追溯性地将块添加到旧时期，而验证者将不知道他们何时可以完成一个时期。

排序窗口还可以防止排序器进行审查：在 SEQUENCING_WINDOW_SIZE L1 块通过后，在给定的 L1 块上进行的存款最坏情况下将包含在 L2 链中。

下图描述了这种关系，以及 L2 块如何从 L1 块派生（L1 信息交易已被省略）：

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203140757601.png" alt="image-20230203140757601" style="zoom:25%;" />

#### Block Derivation Loop

汇总节点的一个子组件称为汇总驱动程序，实际上负责执行块派生。 rollup driver 本质上是一个运行区块推导函数的无限循环。对于每个纪元，块推导函数执行以下步骤：

1. 在排序窗口中下载每个区块的存款和交易批量数据。

2. 将存款和交易批处理数据转换为引擎 API 的有效负载属性。

3. 将有效负载属性提交给引擎 API，在那里它们被转换成块并添加到规范链中。

然后以递增的纪元重复此过程，直到到达 L1 的尖端。

## Engine API

汇总驱动程序实际上并不创建块。相反，它指示执行引擎通过引擎 API 执行此操作。对于上述块派生循环的每次迭代，rollup 驱动程序将创建一个有效负载属性对象并将其发送到执行引擎。执行引擎然后将有效负载属性对象转换为块，并将其添加到链中。汇总驱动程序的基本顺序如下：

1. 使用负载属性对象调用 engine_forkChoiceUpdatedV1。我们现在将跳过分叉选择状态参数的细节——只知道它的字段之一是 L2 链的 headBlockHash，并且它被设置为 L2 链顶端的块哈希。引擎 API 返回有效负载 ID。
2. 使用步骤 1 中返回的有效载荷 ID 调用 engine_getPayloadV1。引擎 API 返回一个有效载荷对象，其中包含块哈希作为其字段之一。
3. 使用步骤 2 中返回的有效负载调用 engine_newPayloadV1。
4. 调用 engine_forkChoiceUpdatedV1，并将分叉选择参数的 headBlockHash 设置为步骤 2 中返回的区块哈希值。L2 链的顶端现在是步骤 1 中创建的区块。

<img src="/Users/carver/Library/Application Support/typora-user-images/image-20230203141114067.png" alt="image-20230203141114067" style="zoom:25%;" />