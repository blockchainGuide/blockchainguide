# L2 输出根提案规范

**目录**

- 提出 L2 产出承诺
  - [L2OutputOracle v1.0.0](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2outputoracle-v100)
  - [L2OutputOracle v2.0.0](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2outputoracle-v200)
- [L2输出承诺构建](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2-output-commitment-construction)
- L2输出预言机智能合约
  - [配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#configuration)
- 安全考虑
  - [L1 重组](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l1-reorgs)

处理一个或多个区块后，输出需要与结算层 (L1) 同步，以便无需信任地执行 L2 到 L1 消息传递（例如提款）。这些输出建议充当桥接器进入 L2 状态的视图。被称为“提议者”的参与者将输出根提交到结算层（L1），并且可以通过故障证明进行争议，如果证明错误，则将面临风险。提议者的此类实现中的[反对提议者](https://github.com/ethereum-optimism/optimism/blob/develop/op-proposer)。

*注意*：目前尚未完全指定 Optimism 的故障证明。[尽管在 Cannon 中实现了](https://github.com/ethereum-optimism/cannon)防错构造和验证，但防错游戏规范以及将输出根挑战者集成到[汇总节点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-node)中 是后来规范里程碑的一部分。

## 提出 L2 产出承诺

提议者的角色是构建并提交输出根，即对 L2 状态的承诺，以及`L2OutputOracle`L1（结算层）上的合约。为此，提议者定期查询汇总[节点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md)以获取从最新 [最终确定的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md#finalization-guarantees)L1 块派生的最新输出根。然后，它获取输出根并将其提交给`L2OutputOracle`结算层（L1）上的合约。

### L2OutputOracle v1.0.0

输出提案的提交仅限于单个帐户。预计该帐户会随着时间的推移继续提交输出提案，以确保用户提款不会停止。

[L2 输出提议者](https://github.com/ethereum-optimism/optimism/blob/develop/op-proposer)预计会根据`SUBMISSION_INTERVAL`中的配置以确定性间隔提交输出根`L2OutputOracle`。越大`SUBMISSION_INTERVAL`，需要将 L1 交易发送到合约的频率就越低`L2OutputOracle` ，但 L2 用户需要等待更长的时间才能将输出根包含在 L1（结算层）中，其中包括他们从系统中提款的意图。

诚实`op-proposer`算法假设与合约有连接，`L2OutputOracle`以了解与必须提交的下一个输出提案相对应的 L2 区块号。它还假设连接到`op-node`能够查询`optimism_syncStatus`RPC 端点的连接。

```
import time

while True:
    next_checkpoint_block = L2OutputOracle.nextBlockNumber()
    rollup_status = op_node_client.sync_status()
    if rollup_status.finalized_l2.number >= next_checkpoint_block:
        output = op_node_client.output_at_block(next_checkpoint_block)
        tx = send_transaction(output)
    time.sleep(poll_interval)
```



一个账户可以通过调用该函数并指定要删除的第一个输出的索引来`CHALLENGER`删除多个输出根，这也将删除所有后续输出。`deleteL2Outputs()`

### L2OutputOracle v2.0.0

产出提案的提交无需许可，并且没有必须提交产出提案的时间间隔。预计用户会“及时”提出输出提案，方便自己提现。必须与输出提案一起放置保证金，以抑制恶意输出的提案。如果可以通过故障证明或证明证明证明输出是恶意的，那么可以削减债券并将其用作支付给支付gas以消除恶意输出的用户的费用。

仍可`op-proposer`用于提交输出提案。一个简单的实现 `op-proposer`将定期提交输出建议。然而，这不是必需的，其他提议者实现可以随时提交有效的输出。更理想的实现将使用启发式方法，例如上次提交的时间或尚未包含在输出提案中的待处理提款数量。

该提议者的单次迭代（将一个输出根发布到 L1）如下所示：



<iframe role="presentation" class="render-viewer" sandbox="allow-scripts allow-same-origin allow-top-navigation" src="https://viewscreen.githubusercontent.com/markdown/mermaid?docs_host=https%3A%2F%2Fdocs.github.com&amp;color_mode=light#62127905-065a-48aa-bb51-0fdbc20c5ad2" name="62127905-065a-48aa-bb51-0fdbc20c5ad2" data-content="{&quot;data&quot;:&quot;sequenceDiagram\n    participant L1\n    participant Rollup Node\n    participant Proposer\n\n    L1-&amp;gt;&amp;gt;L1: L1 block finalized\n    L1-&amp;gt;&amp;gt;Rollup Node: L1 block finalized\n    Proposer-&amp;gt;&amp;gt;Rollup Node: optimism_syncStatus\n    Rollup Node-&amp;gt;&amp;gt;Proposer: sync status { finalized L1 block num }\n    Proposer-&amp;gt;&amp;gt;Rollup Node: optimism_outputAtBlock\n    Rollup Node-&amp;gt;&amp;gt;Proposer: output root\n    Proposer-&amp;gt;&amp;gt;L1: Query L2OutputOracle for this output root\n    L1-&amp;gt;&amp;gt;Proposer: output root or nil\n    Proposer-&amp;gt;&amp;gt;Proposer: stop if the current output is already proposed\n    Proposer-&amp;gt;&amp;gt;L1: L2OutputOracle.proposeOutputRoot\n&quot;}" style="box-sizing: border-box; display: block; width: 1012px; height: 851px; border: 0px;"></iframe>

由于启用无需许可的输出提案时可能有多个提案者同时运行，因此反对者[提案者](https://github.com/ethereum-optimism/optimism/blob/develop/op-proposer)将在发送提案交易之前检查其输出根是否尚未针对给定的 L2 区块号发布。`Proposer` 当查询`L2OutputOracle`输出根时，这如上面的序列图中所示。如果它接收到的输出根等于从汇总节点接收到的输出根，则它不会**在**事务中将此输出根发送到`L2OutputOracle`.

另请注意，虽然[op-proposer实现](https://github.com/ethereum-optimism/optimism/blob/develop/op-proposer)*仅*基于最终确定或安全的 L2 块提交输出，但其他提议者实现可能会提交与不安全（未最终确定）L2 块相对应的输出。[这会带来风险，因为批处理程序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/batcher.md)可能会提交与已提供的输出不对应的 L2 块（意味着这些输出现在*无效*）。

版本`v2.0.0`包括对 ABI 的重大更改`L2OutputOracle`。

## L2输出承诺构建

这`output_root`是一个 32 字节的字符串，它是基于版本化方案派生的：

```pseudocode
output_root = keccak256(version_byte || payload)
```



在哪里：

1. `version_byte`( `bytes32`) 一个简单的版本字符串，只要输出根的构造发生更改，该版本字符串就会递增。
2. `payload`( `bytes`) 是任意长度的字节串。

在输出承诺构造的初始版本中，版本为`bytes32(0)`，有效负载定义为：

```pseudocode
payload = state_root || withdrawal_storage_root || latest_block_hash
```



在哪里：

1. ( ) 是最新 L2 块的块哈希`latest_block_hash`。`bytes32`
2. ( `state_root`)是所有执行层账户的`bytes32`Merkle-Patricia-Trie ( [MPT ) 根。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#merkle-patricia-trie)该值被频繁使用，因此升高到更接近 L2 输出根，从而无需证明其包含在 的原像中`latest_block_hash`。这减少了 Merkle 证明的深度和访问 L1 上的 L2 状态根的成本。
3. ( `withdrawal_storage_root`)提升[Message Passer 合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#the-l2tol1messagepasser-contract)`bytes32`存储的Merkle-Patricia-Trie ( [MPT](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#merkle-patricia-trie) ) 根。我们可以直接针对 L2toL1MessagePasser 的存储根进行证明，而不是针对状态根进行提款的 MPT 证明（首先针对状态根证明 L2toL1MessagePasser 的存储根，然后针对该存储根证明提款），从而减少验证L1 上的提款成本。

## L2输出预言机智能合约

L2 块以恒定速率`L2_BLOCK_TIME`（2 秒）生成。必须将新的 L2 输出添加到链一次，每个`SUBMISSION_INTERVAL`链基于多个块。确切的数量尚未确定，并将取决于故障证明游戏的设计。

L2输出预言机合约实现了以下接口：

```
/**
 * @notice The number of the first L2 block recorded in this contract.
 */
uint256 public startingBlockNumber;

/**
 * @notice The timestamp of the first L2 block recorded in this contract.
 */
uint256 public startingTimestamp;

/**
 * @notice Accepts an L2 outputRoot and the timestamp of the corresponding L2 block. The
 * timestamp must be equal to the current value returned by `nextTimestamp()` in order to be
 * accepted.
 * This function may only be called by the Proposer.
 *
 * @param _l2Output      The L2 output of the checkpoint block.
 * @param _l2BlockNumber The L2 block number that resulted in _l2Output.
 * @param _l1Blockhash   A block hash which must be included in the current chain.
 * @param _l1BlockNumber The block number with the specified block hash.
*/
  function proposeL2Output(
      bytes32 _l2Output,
      uint256 _l2BlockNumber,
      bytes32 _l1Blockhash,
      uint256 _l1BlockNumber
  )

/**
 * @notice Deletes all output proposals after and including the proposal that corresponds to
 *         the given output index. Only the challenger address can delete outputs.
 *
 * @param _l2OutputIndex Index of the first L2 output to be deleted. All outputs after this
 *                       output will also be deleted.
 */
function deleteL2Outputs(uint256 _l2OutputIndex) external

/**
 * @notice Computes the block number of the next L2 block that needs to be checkpointed.
 */
function nextBlockNumber() public view returns (uint256)
```



### 配置

必须`startingBlockNumber`至少是第一个基岩块的编号。必须`startingTimestamp`与起始块的时间戳相同。

因此，第一个`outputRoot`提议将处于高度`startingBlockNumber + SUBMISSION_INTERVAL`

## 安全考虑

### L1 重组

如果 L1 在生成并提交输出后进行重组，则 L2 状态和正确输出可能会发生变化，从而导致错误的提案。通过允许提议者在附加新输出时向输出预言机提交 L1 块号和哈希值可以缓解这种情况；如果发生重组，块哈希将与具有该编号的块的哈希不匹配，并且调用将恢复。