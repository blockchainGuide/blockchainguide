## 组件

<img src="https://github.com/ethereum-optimism/optimism/raw/develop/specs/assets/components.svg" alt="image-20230203112545498" style="zoom:100%;" />

### L1组件

- **OptimismPortal**：L2事务的提要，它起源于 L1状态下的智能契约调用
  
  - OptimmPortal 契约会发出 TransactionDeposited 事件，rollup驱动程序读取这些事件以处理存款。
  
  - 保证Deposits在序列窗口中反映在 L2状态中。
  
  - 注意事务是存储的，而不是令牌。然而，存款交易是实现令牌存款的关键部分(令牌锁定在 L1上，然后通过**存款交易**在 L2上生成)。
- **BatchInbox**：Batch Submitter 向其提交交易批次的 **L1 地址**。
  
  - 交易批次包括 L2 交易calldata、时间戳和排序信息。
  - BatchInbox 是一个常规的 EOA 地址。这让我们可以通过不执行任何 EVM 代码来节省 gas 成本。

### L2组件

- rollup节点(node)
  - 一个独立的、无状态的二进制文件。
  - 接收来自用户的 L2 交易。
  - 同步并验证 L1 上的rollup数据。
  - 应用特定于 rollup 的块生产规则从 L1 合成块。
  - 使用引擎 API 将块附加到 L2 链。
  - 处理 L1 重组。
  - 将未提交的块分发到其他rollup节点。
- 执行引擎(EE)
  - 一个普通的 Geth 节点，经过少量修改以支持 Optimism。
  - 保持 L2 状态。
  - 将状态同步到其他 L2 节点以实现快速入职。
  - 将引擎 API 提供给rollup节点。

### 批量提交(**Batch Submitter**)

- 将交易批次提交到 BatchInbox 地址的后台进程。

### 输出提交器(**Output Submitter**)

- 向 L2OutputOracle 提交 L2 输出承诺的后台进程

## 交易区块广播

由于 EE 在底层使用 Geth，Optimism 使用 Geth 的内置点对点网络和交易池来传播交易。同一网络还可用于传播提交的块并支持快照同步。

然而，未提交的块将使用单独的 Rollup 节点对等网络传播。然而，这是可选的，并且是为了降低验证者及其 JSON-RPC 客户端的延迟而提供的。

下图说明了排序器和验证器如何组合在一起：

<img src="https://github.com/ethereum-optimism/optimism/raw/develop/specs/assets/propagation.svg" style="zoom:100%;" />

## 深入的关键交互

### 存款

Optimism支持两种类型的质押：用户质押和L1 attributes deposits。为了执行用户存款，用户调用 OptimismPortal 合约上的 depositTransaction 方法。这反过来会发出 TransactionDeposited 事件，rollup 节点会在区块派生期间读取这些事件。

L1 属性存款用于通过调用 L1 属性预部署在 L2 上注册 L1 块属性（数字、时间戳等）。它们不能由用户发起，而是由 rollup 节点自动添加到 L2 块中。

两种存款类型都由 L2 上的单个自定义 EIP-2718 交易类型表示。

### 区块推导

给定 L1 以太坊链，可以确定性地导出rollup链。整个 rollup 链可以基于 L1 块派生这一事实使 Optimism 成为 rollup。这个过程可以表示为：

```go
derive_rollup_chain(L1_blockchain) -> rollup_blockchain
```

Optimism 的区块推导函数是这样设计的：

- 除了可以使用 L1 和 L2 执行引擎 API 轻松访问的状态外，不需要任何状态。
- 支持sequencer和sequencer共识
- 对sequencer审查具有弹性

#### Epochs and the Sequencing Window

rollup 链被细分为 epoch。 L1 区块编号和纪元编号之间存在 1:1 的对应关系。

对于编号为 n 的 L1 区块，有一个相应的 rollup epoch n，它只能在经过一个排序窗口值的区块后得出，即在编号为 n + SEQUENCING_WINDOW_SIZE 的 L1 区块被添加到 L1 链之后。

每个纪元至少包含一个块。纪元中的每个块都包含L1信息事务，该事务包含关于L1的上下文信息，例如块散列和时间戳。纪元中的第一个区块还包含通过L1上的OptimimPortal合同启动的所有矿床。所有L2块也可以包含排序的事务，即直接提交给排序器的事务。

每当sequencer为给定历元创建新的L2块时，它必须在历元的定序窗口内将其作为批的一部分提交给L1（即，**批必须在L1块n+sequencing_window_SIZE之前着陆**）。这些批处理（连同TransactionDeposited L1事件）允许从L1链派生L2链。

定序器不需要将一个L2块批量提交给L1以在其上构建。事实上，批量通常包含多个L2块的有序事务。这就是为什么能够在sequencer上快速确认交易的原因。

由于给定epoch的事务批次可以在排序窗口内的任何位置提交，因此验证器必须在该窗口内的所有块中搜索事务批次。这可以防止L1的事务包含的不确定性。这种不确定性也是我们首先需要测序窗口的原因：否则测序器可能会向旧的历元中追溯添加块，而验证器不知道何时可以完成历元。

测序窗口还防止了测序器的审查：在序列号为window_SIZE的L1块通过后，在给定L1块上进行的沉积最坏情况下将被包括在L2链中。

下图描述了这种关系，以及L2块是如何从L1块派生的（L1信息事务已被忽略）：

<img src="https://github.com/ethereum-optimism/optimism/raw/develop/specs/assets/sequencer-block-gen.svg" style="zoom:100%;" />

#### Block Derivation Loop

汇总节点的一个子组件称为 *rollup driver*，实际上负责执行块派生。 rollup driver 本质上是一个运行区块推导函数的无限循环。对于每个纪元，块推导函数执行以下步骤：

1. 在排序窗口中下载每个区块的存款和交易批量数据。

2. 将存款和交易批处理数据转换为引擎 API 的payload attributes 。

3. 将payload attributes 提交给引擎 API，在那里它们被转换成块并添加到规范链中。

然后以递增的纪元重复此过程，直到到达 L1 的尖端。

## Engine API

**汇总驱动程序实际上并不创建块**。相反，它指示执行引擎通过引擎 API 执行此操作。对于上述块派生循环的每次迭代，rollup 驱动程序将创建一个**有效负载属性**对象并将其发送到执行引擎。执行引擎然后将有效负载属性对象转换为块，并将其添加到链中。汇总驱动程序的基本顺序如下：

1. 使用负载属性对象调用 engine_forkChoiceUpdatedV1。我们现在将跳过分叉选择状态参数的细节——只知道它的字段之一是 L2 链的 headBlockHash，并且它被设置为 L2 链顶端的块哈希。引擎 API 返回有效负载 ID。
2. 使用步骤 1 中返回的有效载荷 ID 调用 engine_getPayloadV1。引擎 API 返回一个有效载荷对象，其中包含块哈希作为其字段之一。
3. 使用步骤 2 中返回的有效负载调用 engine_newPayloadV1。
4. 调用 engine_forkChoiceUpdatedV1，并将分叉选择参数的 headBlockHash 设置为步骤 2 中返回的区块哈希值。L2 链的顶端现在是步骤 1 中创建的区块。

<img src="https://github.com/ethereum-optimism/optimism/blob/develop/specs/assets/engine.svg" style="zoom:100%;" />