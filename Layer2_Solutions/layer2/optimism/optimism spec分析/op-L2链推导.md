# L2 链推导规范

**目录**

- 概述
  - [渴望区块推导](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#eager-block-derivation)
- 批量提交
  - [排序和批量提交概述](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#sequencing--batch-submission-overview)
  - 批量提交电汇格式
    - [批处理交易格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batcher-transaction-format)
    - [帧格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#frame-format)
    - [频道格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#channel-format)
    - [批次格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-format)
- 建筑学
  - L2链衍生管道
    - [L1遍历](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#l1-traversal)
    - [L1检索](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#l1-retrieval)
    - [帧队列](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#frame-queue)
    - 渠道银行
      - [修剪](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#pruning)
      - [超时](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#timeouts)
      - [阅读](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#reading)
      - [加载帧](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#loading-frames)
    - [通道读取器（批量解码）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#channel-reader-batch-decoding)
    - [批量队列](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-queue)
    - [负载属性推导](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#payload-attributes-derivation)
    - 引擎队列
      - [引擎API使用](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#engine-api-usage)
      - [Forkchoice同步](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#forkchoice-synchronization)
      - [L1-consolidation：负载属性匹配](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#l1-consolidation-payload-attributes-matching)
      - [L1-sync：有效负载属性处理](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#l1-sync-payload-attributes-processing)
      - [处理不安全的有效负载属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#processing-unsafe-payload-attributes)
    - 重置管道
      - [寻找同步起点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#finding-the-sync-starting-point)
      - [重置推导阶段](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#resetting-derivation-stages)
      - [关于合并后重组](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#about-reorgs-post-merge)
- 派生有效负载属性
  - [导出交易列表](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#deriving-the-transaction-list)
  - [构建单独的有效负载属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#building-individual-payload-attributes)

# 概述

> 请注意，以下内容假设单个定序器和批处理器。将来，该设计将进行调整以容纳多个此类实体。

[L2 链推导](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-derivation)——从 L1 数据推导 L2[区块——是](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block)[rollup 节点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-node)的主要职责之一，无论是在验证器模式还是在排序器模式下（其中推导充当排序的健全性检查，并能够检测 L1 链重组[）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#chain-re-organization)）。

L2链源自L1链。具体地，每个L1块被映射到包括多个L2块的L2[排序时期](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing-epoch)。纪元号被定义为等于相应的L1区块号。

为了导出 epoch 中的 L2 区块`E`，我们需要以下输入：

- epoch 的L1

  排序窗口

  

  ```
  E
  ```

  ： 范围内的 L1 块，

  ```
  [E, E + SWS)
  ```

  其中

  ```
  SWS
  ```

   是排序窗口大小（请注意，这意味着 epoch 是重叠的）。特别是，我们需要：

  - 排序窗口中包含的

    批处理事务

    。这些允许我们重建包含要包含在 L2 块中的交易的

    定序器批次

    （每个批次包含 L2 块的列表）。

    - 请注意，批处理交易不可能包含与`E`L1 区块上的 纪元相关的批次`E`，因为该批次必须包含 L1 区块的哈希值`E`。

  - L1 区块中的存款（以[存款合约发出](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposit-contract)[的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposits)`E`事件的形式）。

  - L1区块属性来自L1区块`E`（以导出[存入交易的L1属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-attributes-deposited-transaction)）。

- 纪元最后一个 L2 区块之后 L2 链的状态

  ```
  E - 1
  ```

  ，或者（如果纪元

  ```
  E - 1
  ```

  不存在） 

  L2 创世状态

  。

  - [如果L2 链起始点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-inception)在哪里，则epoch`E`不存在。`E <= L2CI``L2CI`

[为了从头开始推导整个 L2 链，我们只需从L2 创世状态](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l2-genesis-block)开始，并将[L2 链起始](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-inception)作为第一个 epoch，然后按顺序处理所有排序窗口。有关我们如何在实践中实现这一点的更多信息，请参阅 [架构部分。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#architecture)L2 链可能包含基岩前的历史，但这里的 L2 起源指的是第一个基岩 L2 区块。

每个时期可能包含可变数量的 L2 块（每一个`l2_block_time`，乐观时为 2 秒），由 [排序器](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer)自行决定，但每个块受到以下约束：

- ```
  min_l2_timestamp <= block.timestamp <= max_l2_timestamp
  ```

  ， 在哪里

  - 所有这些值均以秒为单位

  - ```
    min_l2_timestamp = l1_timestamp
    ```

    - 这可确保 L2 时间戳不落后于 L1 原始时间戳。

  - ```
    block.timestamp = prev_l2_timestamp + l2_block_time
    ```

    - `prev_l2_timestamp`是上一个纪元的最后一个L2块的时间戳
    - `l2_block_time`是 L2 块之间时间的可配置参数（乐观时为 2 秒）

  - ```
    max_l2_timestamp = max(l1_timestamp + max_sequencer_drift, min_l2_timestamp + l2_block_time)
    ```

    - `l1_timestamp`是与 L2 区块纪元关联的 L1 区块的时间戳
    - `max_sequencer_drift`是排序器允许领先于 L1 的最大程度

总而言之，这些约束意味着每秒必须有一个 L2 块`l2_block_time`，并且纪元的第一个 L2 块的时间戳绝不能落后于与该纪元匹配的 L1 块的时间戳。

合并后，以太坊的固定出[块时间](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block-time)为 12 秒（尽管可以跳过某些时隙）。因此，预计在 2 秒的 L2 区块时间下，大多数情况下，每个 epoch 将包含`12/2 = 6`L2 区块。然而，定序器可以延长或缩短纪元（受上述限制）。其基本原理是在 L1 上跳过时隙或暂时失去与 L1 的连接（这需要更长的 epoch）的情况下保持活跃性。然后需要更短的纪元来避免 L2 时间戳越来越领先于 L1。

请注意，`min_l2_timestamp + l2_block_time`即使超出，也可确保始终可以处理新的 L2 批次 `max_sequencer_drift`。但是，当超过 时`max_sequencer_drift`，将强制执行到下一个 L1 源，但有一个例外，以确保在下一个 L2 批次中可以满足最小时间戳界限（基于下一个 L1 源），并且在超过时继续强制`len(batch.transactions) == 0`执行`max_sequencer_drift`。更多详情请参见[批处理队列]。

## 渴望区块推导

在实践中，通常不需要等待 L1 块的完整排序窗口就可以开始导出一个 epoch 中的 L2 块。事实上，只要我们能够重建连续的批次，我们就可以开始推导相应的 L2 块。我们称之为*急切块派生*。

然而，在最坏的情况下，我们只能通过读取测序窗口的最后一个 L1 块来重建该纪元中第一个 L2 块的批次。当该批次的某些数据包含在窗口的最后一个 L1 块中时，就会发生这种情况。在这种情况下，我们不仅无法导出该纪元中的第一个 L2 块，而且在此之前我们也无法导出该纪元中的任何其他 L2 块，因为它们需要应用该纪元的第一个 L2 块所产生的状态。（请注意，这仅适用于*块*派生。批次仍然可以派生并暂时排队，我们只是无法从中创建块。）

------

# 批量提交

## 排序和批量提交概述

排序[器](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer)接受来自用户的 L2 事务。它负责构建这些块。对于每个这样的块，它还会创建一个相应的[定序器批次](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)。它还负责将每个批次提交给[数据可用性提供者](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#data-availability-provider)（例如以太坊calldata），这是通过其[批处理](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher)程序组件完成的。

L2 块和批处理之间的区别很微妙但很重要：块包含 L2 状态根，而批处理仅在给定的 L2 时间戳（相当于：L2 块号）提交事务。块还包括对前一个块的引用 (*)。

(*) 这在某些边缘情况下很重要，其中会发生 L1 重组，并且批次将被重新发布到 L1 链，但不是前一个批次，而 L2 块的前身不可能改变。

这意味着即使定序器错误地应用了状态转换，批次中的交易仍将被视为规范 L2 链的一部分。批次仍然需要接受有效性检查（即它们必须正确编码），批次内的各个交易也是如此（例如签名必须有效）。无效批次和有效批次中无效的单个交易将被正确的节点丢弃。

如果定序器错误地应用状态转换并发布[输出根](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l2-output-root)，则该输出根将不正确。错误的输出根将受到[故障证明的挑战，然后由](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#fault-proof)**现有定序器批次的**正确输出根替换。

有关更多信息，请参阅[批量提交规范。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/batcher.md)

## 批量提交电汇格式

批量提交与 L2 链派生密切相关，因为派生过程必须对为了批量提交而编码的批次进行解码。

[批处理程序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher)将[批处理程序事务](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)提交给[数据可用性提供者](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#data-availability-provider)。这些事务包含一个或多个[通道帧](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)，它们是属于某个[通道](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel)的数据块。

通道是压缩在一起的一系列定[序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel)[器批次](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)（对于任何 L2 块）。将多个批次分组在一起的原因很简单，就是为了获得更好的压缩率，从而降低数据可用性成本。

通道可能太大而无法容纳单个[批处理器事务](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)，因此我们需要将其分成称为[通道帧的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)块。单个批处理器事务还可以携带多个帧（属于相同或不同的通道）。

这种设计为我们如何将批次聚合到通道以及如何通过批次事务拆分通道提供了最大的灵活性。它特别允许我们在批处理事务中最大化数据利用率：例如，它允许我们将窗口的最终（小）帧与下一个窗口的大帧打包。

将来，这一通道识别功能还允许[批处理程序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher)使用多个签名者（私钥）并行提交一个或多个通道 (1)。

(1) 这有助于缓解以下问题：由于交易随机数值影响 L2 交易池并因此包含在内：同一签名者进行的多个交易陷入等待包含先前交易的状态。

另请注意，我们使用流压缩方案，并且当我们启动通道时，甚至当我们发送通道中的第一帧时，我们不需要知道通道最终将包含多少个块。

通过跨多个数据事务分割通道，L2 可以拥有比数据可用性层可支持的更大的块数据。

所有这些都如下图所示。解释如下。

[![批次推导链图](https://github.com/ethereum-optimism/optimism/raw/develop/specs/assets/batch-deriv-chain.svg)](https://github.com/ethereum-optimism/optimism/blob/develop/specs/assets/batch-deriv-chain.svg)

第一行代表 L1 块及其编号。L1 区块下方的方框代表区块内包含的[批处理交易。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)L1 区块下方的波浪线代表 [存款](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposits)（更具体地说，是[存款合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposit-contract)发出的事件）。

框中的每个彩色块代表一个[通道框架](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)。所以`A`和`B`是 [通道](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel)，而`A0`, `A1`, `B0`, `B1`,`B2`是帧。请注意：

- 多个通道交错
- 帧不需要按顺序传输
- 单个批处理事务可以携带来自多个通道的帧

在下一行中，圆形框代表从通道中提取的各个[测序器批次。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)蓝色/紫色/粉色四种颜色来自通道`A`，其他颜色来自通道`B`。这些批次在这里按照它们从批次中解码的顺序表示（在本例中`B`是首先解码）。

> 注意此处的标题显示“首先看到通道 B，并将首先将其解码为批次”，但这不是必需的。例如，对于一种实现来说，查看通道并首先解码包含最旧批次的通道同样是可以接受的。

该图的其余部分在概念上与第一部分不同，并说明了通道重新排序后的 L2 链推导。

第一行显示批处理交易。请注意，在这种情况下，存在批次排序，使得通道内的所有帧连续出现。一般来说，情况并非如此。`A1`例如，在第二笔交易中，和的位置`B0`可以颠倒以获得完全相同的结果 - 图中的其余部分不需要进行任何更改。

第二行以正确的顺序显示重建的通道。第三行显示从通道中提取的批次。由于通道是有序的并且通道内的批次是连续的，这意味着批次也是有序的。第四行显示了从每个批次导出的[L2 块](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block)。请注意，我们在这里有一个 1-1 批次到块的映射，但是，正如我们稍后将看到的，如果在 L1 上发布的批次中存在“间隙”，则可以插入未映射到批次的空块。

第五行显示了[L1 属性存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-attributes-deposited-transaction)，该交易在每个 L2 区块内记录了与 L2 区块的纪元相匹配的 L1 区块的信息。第一个数字表示纪元/L1x 编号，而第二个数字（“序列号”）表示纪元内的位置。

最后，第六行显示了前面提到的[存款合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposit-contract)事件衍生的[用户存款交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#user-deposited-transaction)。

请注意`101-0`该图右下角的 L1 属性事务。`B2`仅当帧指示它是通道内的最后一个帧并且(2) 不得插入空块时，它才可能存在。

该图没有指定使用的排序窗口大小，但从中我们可以推断它必须至少为 4 个块，因为通道的最后一帧`A`出现在块 102 中，但属于 epoch 99。

至于“安全类型”的注释，它解释了 L1 和 L2 上使用的块的分类。

- [不安全的 L2 块](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#unsafe-l2-block)：
- [安全 L2 块](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#safe-l2-block)：
- [最终确定的 L2 块：指从最终确定的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#finalized-l2-head)L1 数据导出的块。

这些安全级别映射到与[执行引擎 API](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md)交互时传输的`headBlockHash`、`safeBlockHash`和值。`finalizedBlockHash`

### 批处理交易格式

批处理事务被编码为`version_byte ++ rollup_payload`（其中`++`表示串联）。

| `version_byte` | `rollup_payload`                |
| -------------- | ------------------------------- |
| 0              | `frame ...`（一帧或多帧，串联） |

未知版本使批处理器事务无效（必须被汇总节点忽略）。批处理事务中的所有帧都必须是可解析的。如果任何一帧无法解析，则事务中的所有帧都将被拒绝。

通过验证交易`to`地址是否与批量收件箱地址匹配，以及该地址是否与读取交易数据的 L1 块时[系统配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#system-configuration)`from`中的批量发送者地址匹配，来对批量交易进行身份验证。

### 帧格式

通道[帧](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)编码为：

```text
frame = channel_id ++ frame_number ++ frame_data_length ++ frame_data ++ is_last

channel_id        = bytes16
frame_number      = uint16
frame_data_length = uint32
frame_data        = bytes
is_last           = bool
```



其中`uint32`和`uint16`都是大端无符号整数。类型名称应根据[Solidity ABI](https://docs.soliditylang.org/en/v0.8.16/abi-spec.html)进行解释和编码。

帧中的所有数据都是固定大小的，除了`frame_data`. 固定开销为`16 + 2 + 4 + 1 = 23 bytes`。固定大小的帧元数据避免了与目标总数据长度的循环依赖，以简化具有不同内容长度的帧的打包。

在哪里：

- `channel_id`是通道的不透明标识符。不宜重复使用，建议随机；但是，在超时规则之外，不会检查有效性
- `frame_number`标识通道内帧的索引
- `frame_data_length`是以字节为单位的长度`frame_data`。它的上限为 1,000,000 字节。
- `frame_data`是属于通道的字节序列，逻辑上位于前一帧的字节之后
- `is_last`是一个单字节，如果该帧是通道中的最后一个帧，则值为 1；如果通道中存在帧，则值为 0。任何其他值都会使帧无效（汇总节点必须忽略它）。

### 频道格式

通道编码为`channel_encoding`，定义为：

```text
rlp_batches = []
for batch in batches:
    rlp_batches.append(batch)
channel_encoding = compress(rlp_batches)
```



在哪里：

- `batches`是输入，按照下一节（“批量编码”）进行字节编码的批次序列
- `rlp_batches`是 RLP 编码批次的串联
- `compress`是一个执行压缩的函数，使用 ZLIB 算法（如[RFC-1950](https://www.rfc-editor.org/rfc/rfc1950.html)中指定），没有字典
- `channel_encoding`是压缩版本`rlp_batches`

在解压缩通道时，我们将解压缩的数据量限制为`MAX_RLP_BYTES_PER_CHANNEL`（当前为 10,000,000 字节），以避免“zip-bomb”类型的攻击（其中小的压缩输入解压缩为巨大的数据量）。如果解压缩的数据超出限制，则处理过程就像通道仅包含第一个`MAX_RLP_BYTES_PER_CHANNEL`解压缩的字节一样。`MAX_RLP_BYTES_PER_CHANNEL`RLP 解码设置了限制，因此即使通道的大小大于 ， 所有可以解码的批次也将被接受`MAX_RLP_BYTES_PER_CHANNEL`。确切的要求是`length(input) <= MAX_RLP_BYTES_PER_CHANNEL`。

虽然上述伪代码意味着所有批次都是预先已知的，但可以对 RLP 编码批次执行流式压缩和解压缩。这意味着在我们知道通道将包含多少个批次（以及多少个帧）之前，可以开始在 [批处理器事务中包含通道帧。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)

### 批次格式

回想一下，批次包含要包含在特定 L2 块中的交易列表。

批次编码为`batch_version ++ content`，其中`content`取决于`batch_version`：

| `batch_version` | `content`                                                    |
| --------------- | ------------------------------------------------------------ |
| 0               | `rlp_encode([parent_hash, epoch_number, epoch_hash, timestamp, transaction_list])` |

在哪里：

- `batch_version`是一个单字节，在 RLP 内容之前添加前缀，类似于事务类型。
- `rlp_encode`是一个根据[RLP 格式](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)对批次进行编码的函数，并`[x, y, z]`表示包含项目的列表`x`，`y`以及`z`
- `parent_hash`是前一个L2块的块哈希
- `epoch_number`和是与L2 区块的[排序纪元](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing-epoch)`epoch_hash`对应的 L1 区块的编号和哈希值
- `timestamp`是L2块的时间戳
- `transaction_list`[是EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)编码交易的 RLP 编码列表。

未知版本会使批次无效（汇总节点必须忽略它），格式错误的内容也是如此。

和`epoch_number`还必须遵守[“批处理队列”](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-queue)`timestamp`部分中列出的约束 ，否则该批处理将被视为无效并将被忽略。

------

# 建筑学

上面主要描述了L2链推导中使用的通用编码，主要是如何在[批量交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)中对批量进行编码。

本节介绍如何使用管道架构从 L1 批次生成 L2 链。

验证者可以以不同的方式实现这一点，但必须在语义上等效，以免偏离 L2 链。

## L2链衍生管道

我们的架构将推导过程分解为由以下阶段组成的管道：

1. L1遍历
2. L1检索
3. 帧队列
4. 渠道银行
5. 通道读取器（批量解码）
6. 批量队列
7. 负载属性推导
8. 引擎队列

数据从管道的起点（外部）流向终点（内部）。从最里面的阶段，数据是从最外面的阶段拉取的。

然而，数据以相反的顺序*处理。*意思是如果最后一个阶段有数据要处理，就会先处理。处理按每个阶段可以采取的“步骤”进行。我们尝试在最后（最内部）阶段采取尽可能多的步骤，然后再在其外部阶段采取任何步骤，等等。

这确保了我们在提取更多数据之前使用已有的数据，并最大限度地减少数据穿过派生管道的延迟。

每个阶段都可以根据需要维持自己的内部状态。特别是，每个阶段都维护对最新 L1 块的 L1 块引用（数字 + 哈希），以便源自先前块的所有数据都已完全处理，并且来自该块的数据正在或已经被处理。这使得最里面的阶段能够最终确定用于生成 L2 链的 L1 数据可用性，从而在 L2 链输入变得不可逆时反映在 L2 链分叉选择中。

让我们简要描述一下管道的每个阶段。

### L1遍历

在*L1遍历*阶段，我们只需读取下一个L1块的头部。[在正常操作中，这些在创建时将是新的 L1 块，尽管我们也可以在同步时读取旧块，或者在 L1重组](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#chain-re-organization)的情况下读取旧块。

遍历 L1 块时，L1 检索阶段使用的[系统配置副本会更新，以便批量发送方身份验证始终准确到该阶段读取的确切 L1 块。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#system-configuration)

### L1检索

在*L1检索*阶段，我们读取从外部阶段（L1遍历）获得的块，并从中提取数据。默认情况下，对于每个事务，汇总都会对从块中的[批处理器事务](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#batcher-transaction)中检索到的调用数据进行操作：

- 接收者必须是配置的批处理程序收件箱地址。
- 发送方必须匹配从系统配置加载的批处理地址，该地址与数据的 L1 块相匹配。

每个数据事务都有版本控制，并包含一系列由帧队列读取的[通道帧，请参阅](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)[批量提交线路格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-submission-wire-format)。

### 帧队列

帧队列一次缓冲一个数据事务，解码为[通道帧](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame)，供下一阶段使用。请参阅[批处理程序事务格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batcher-transaction-format)和[帧格式](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#frame-format)规范。

### 渠道银行

通道*库*阶段负责管理由 L1 检索阶段写入的通道库的缓冲。通道组阶段中的一个步骤尝试从“就绪”的通道读取数据。

通道当前完全缓冲，直到读取或丢弃，ChannelBank 的未来版本可能会支持流通道。

为了限制资源使用，通道库根据通道大小进行修剪，并使旧通道超时。

*通道按 FIFO 顺序记录在称为通道队列的*结构中。当第一次看到属于该通道的帧时，该通道就会被添加到通道队列中。

#### 修剪

成功插入新帧后，ChannelBank 会被修剪：通道按 FIFO 顺序丢弃，直到`total_size <= MAX_CHANNEL_BANK_SIZE`，其中：

- `total_size`是每个通道的大小之和，即通道的所有缓冲帧数据的总和，以及`200`每帧字节的额外帧开销。
- `MAX_CHANNEL_BANK_SIZE`是 100,000,000 字节的协议常量。

#### 超时

通道打开的 L1 原点通过通道 as 进行跟踪`channel.open_l1_block`，并确定在修剪之前保留通道数据的 L1 块的最大跨度。

如果出现以下情况，则通道超时：`current_l1_block.number > channel.open_l1_block.number + CHANNEL_TIMEOUT`，其中：

- `current_l1_block`是舞台当前经过的 L1 原点。
- `CHANNEL_TIMEOUT`是可汇总配置的，以 L1 块的数量表示。

超时通道的新帧将被丢弃而不是被缓冲。

#### 阅读

通道组只能从第一个打开的通道输出数据。

读取后，当第一个打开的通道超时时，将其从通道库中删除。

一旦第一个打开的通道（如果有）未超时且准备就绪，就会读取该通道并将其从通道组中删除。

如果满足以下条件，则通道已准备就绪：

- 通道已关闭
- 通道具有连续的帧序列，直到关闭帧

如果没有通道准备就绪，则读取下一帧并将其摄取到通道组中。

#### 加载帧

当帧引用的通道 ID 尚未存在于通道库中时，将打开一个新通道，用当前 L1 块进行标记，并将其附加到通道队列中。

帧插入条件：

- 与尚未从通道库中修剪的超时通道相匹配的新帧将被丢弃。
- 尚未从通道库中修剪的帧的重复帧（按帧号）将被丢弃。
- 重复的关闭（新框架`is_last == 1`，但通道已经看到关闭框架并且尚未从通道库中修剪）被丢弃。

如果帧正在关闭 ( `is_last == 1`)，则任何现有的编号较高的帧都会从通道中删除。

请注意，虽然这允许通道 ID 在从通道库中删除后可以重复使用，但建议批处理程序实现使用唯一的通道 ID。

### 通道读取器（批量解码）

在这个阶段，我们解压缩从最后一个阶段拉出的通道，然后 从解压缩的字节流中解析[批次。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)

有关解压缩和解码规范，请参阅[批处理格式。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-format)

### 批量队列

在*批量缓冲*阶段，我们按时间戳对批次重新排序。[如果某些时间段](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#time-slot)缺少批次，并且存在具有较高时间戳的有效批次，则此阶段还会生成空批次来填补空白。

只要有一个连续批次直接跟在当前[安全 L2 头](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#safe-l2-head)（可以从规范 L1 链派生的最后一个块）的时间戳之后，批次就会被推送到下一阶段。该批次的父哈希也必须与当前安全 L2 头的哈希相匹配。

请注意，从 L1 派生的批次中存在任何间隙意味着该阶段需要缓冲整个 [测序窗口](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing-window)，然后才能生成空批次（因为丢失的批次可能在最后一个 L1 块中包含数据）最坏情况下的窗口）。

一个批次可以有 4 种不同形式的有效性：

- `drop`：该批次无效，并且将来一直如此，除非我们重新组织。可以将其从缓冲区中删除。
- `accept`：该批次有效，应进行处理。
- `undecided`：在我们可以进行批量过滤之前，我们缺乏 L1 信息。
- `future`：该批次可能有效，但尚无法处理，应稍后再次检查。

批次按照包含在 L1 上的顺序进行处理：如果可以进行多个批次，`accept`则应用第一个批次。实现可以推迟`future`批次稍后的推导步骤以减少验证工作。

批次有效性计算如下：

定义：

- `batch`[如批处理格式部分](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-format)中所定义。
- `epoch = safe_l2_head.l1_origin`与批次耦合的 [L1 源](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-origin)，具有以下属性： `number`（L1 块编号）、`hash`（L1 块哈希）和`timestamp`（L1 块时间戳）。
- `inclusion_block_number``batch`是第一次*完全*推导时的L1块号，即由前一级解码和输出时的L1块号。
- `next_timestamp = safe_l2_head.timestamp + block_time`是下一批应该具有的预期 L2 时间戳，请参阅[块时间信息](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block-time)。
- `next_epoch`可能还不知道，但`epoch`如果可用的话将是 L1 块。
- `batch_origin`是 或`epoch`，`next_epoch`取决于验证。

请注意，批次的处理可以推迟到`batch.timestamp <= next_timestamp`，因为`future`无论如何都必须保留批次。

规则，按验证顺序：

- `batch.timestamp > next_timestamp`-> `future`：即批次必须准备好处理。

- `batch.timestamp < next_timestamp`-> `drop`：即批次不能太旧。

- `batch.parent_hash != safe_l2_head.hash`-> `drop`：即父哈希必须等于L2安全头块哈希。

- `batch.epoch_num + sequence_window_size < inclusion_block_number`-> `drop`：即该批次必须及时包含。

- `batch.epoch_num < epoch.number`-> `drop`：即批次来源不早于L2安全头的来源。

- `batch.epoch_num == epoch.number`: 定义`batch_origin`为`epoch`.

- ```
  batch.epoch_num == epoch.number+1
  ```

  ：

  - 如果`next_epoch`未知 -> `undecided`：即在我们拥有 L1 原始数据之前，无法处理更改 L1 原点的批次。
  - 如果已知，则定义`batch_origin`为`next_epoch`

- `batch.epoch_num > epoch.number+1`-> `drop`：即每个 L2 块的 L1 原点不能更改超过一个 L1 块。

- `batch.epoch_hash != batch_origin.hash`-> `drop`：即批次必须引用规范的 L1 来源，以防止批次被重播到意外的 L1 链上。

- `batch.timestamp < batch_origin.time`-> `drop`：强制执行最小 L2 时间戳规则。

- ```
  batch.timestamp > batch_origin.time + max_sequencer_drift
  ```

  ：强制执行 L2 时间戳漂移规则，但有例外情况以保留高于最小 L2 时间戳不变性：

  - ```
    len(batch.transactions) == 0
    ```

    ：

    - ```
      epoch.number == batch.epoch_num
      ```

      ：这意味着该批次尚未提前至 L1 原点，因此必须对照 进行检查

      ```
      next_epoch
      ```

      。

      - 如果`next_epoch`未知 -> `undecided`：如果没有下一个 L1 原点，我们还无法确定是否可以保持时间不变。
      - 如果`batch.timestamp >= next_epoch.time`-> `drop`：批次可以采用下一个 L1 原点而不破坏`L2 time >= L1 time`不变量。

  - `len(batch.transactions) > 0`: -> `drop`: 当超过定序器时间漂移时，绝不允许定序器包含事务。

- ```
  batch.transactions
  ```

  ：

  ```
  drop
  ```

  如果

  ```
  batch.transactions
  ```

  列表中包含无效交易或仅通过其他方式衍生的交易：

  - 任何空交易（零长度字节字符串）
  - 任何[存入的交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposited-transaction-type)（由交易类型前缀字节标识）

如果没有批次可以被`accept`-ed，并且该阶段已完成可以从高度为 L1 块完全读取的所有批次的缓冲`epoch.number + sequence_window_size`，并且`next_epoch`可用，则可以派生出具有以下属性的空批次：

- `parent_hash = safe_l2_head.hash`

- `timestamp = next_timestamp`

- `transactions`为空，即没有定序器事务。存入交易可能会在下一阶段添加。

- 如果

  ```
  next_timestamp < next_epoch.time
  ```

  ：重复当前的 L1 原点，以保持 L2 时间不变。

  - `epoch_num = epoch.number`
  - `epoch_hash = epoch.hash`

- 如果批次是纪元的第一批，则使用该纪元而不是推进纪元，以确保每个纪元至少有一个 L2 块。

  - `epoch_num = epoch.number`
  - `epoch_hash = epoch.hash`

- 否则，

  - `epoch_num = next_epoch.number`
  - `epoch_hash = next_epoch.hash`

### 负载属性推导

在*有效负载属性导出*阶段，我们将从前一阶段获得的批次转换为结构的实例[`PayloadAttributes`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#payload-attributes)。这种结构对需要放入区块的交易以及其他区块输入（时间戳、费用接收者等）进行编码。[有效负载属性派生在下面的派生有效负载属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#deriving-payload-attributes)部分中详细介绍。

该阶段维护自己的[系统配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#system-configuration)副本，独立于 L1 检索阶段。每当批量输入引用的 L1 纪元发生变化时，系统配置就会使用 L1 日志事件进行更新。

### 引擎队列

在*引擎队列*阶段，先前导出的`PayloadAttributes`结构被缓冲并发送到 [执行引擎](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#execution-engine)执行并转换为适当的L2块。

该阶段维护对三个 L2 块的引用：

- 最终[确定的 L2 头](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#finalized-l2-head)：直到并包括该块的所有内容都可以完全源自 L1 链的[最终确定（即规范且永远不可逆）部分。](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/#finality)
- 安全[的 L2 头](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#safe-l2-head)：直到并包括该块的所有内容都可以完全源自当前规范的 L1 链。
- 不安全[的L2头](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#unsafe-l2-head)：安全头和不安全头之间的块是不是从L1派生的[不安全块。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#unsafe-l2-block)这些块要么来自排序（在定序器模式下），要么来自与定序器的[不安全同步（在验证器模式下）。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#unsafe-sync)这也称为“最新”头。

此外，它还缓冲最近处理的安全 L2 块的引用的简短历史记录，以及每个 L1 块派生的引用。该历史不必是完整的，但可以使以后的 L1 最终信号能够转换为 L2 最终信号。

#### 引擎API使用

为了与引擎交互，使用[执行引擎 API ，通过以下 JSON-RPC 方法：](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md)

- [`engine_forkchoiceUpdatedV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine_forkchoiceupdatedv1)— 如果不同，则更新 forkchoice（即链头）`headBlockHash`，如果有效负载属性参数不是，则指示引擎开始构建执行有效负载`null`。
- [`engine_getPayloadV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine_getpayloadv1)— 检索先前请求的执行负载构建。
- [`engine_newPayloadV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine_newpayloadv1)— 执行执行负载来创建块。

执行有效负载是类型的对象[`ExecutionPayloadV1`](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md#executionpayloadv1)。

#### Forkchoice同步

如果在派生或处理其他输入之前要应用任何 forkchoice 更新，那么这些更新将首先应用于引擎。

这种同步可能会在以下情况下发生：

- L1 最终信号最终确定一个或多个 L2 块：更新“最终确定”的 L2 块。
- 成功整合不安全的 L2 块：更新“安全”L2 块。
- 派生管道重置后的第一件事是确保执行引擎 forkchoice 状态一致。

新的 forkchoice 状态通过 来应用`engine_forkchoiceUpdatedV1`。在出现 forkchoice-state 有效性错误时，必须重置派生管道以恢复到一致状态。

#### L1-consolidation：负载属性匹配

如果不安全头位于安全头之前，则尝试合并，验证现有不安全 L2 链是否与从规范 L1 数据导出的导出 L2 输入相匹配[。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#unsafe-block-consolidation)

在合并期间，我们考虑最旧的不安全L2块，即紧接在安全头之后的不安全L2块。如果有效负载属性与这个最旧的不安全 L2 块匹配，则该块可以被视为“安全”并成为新的安全头。

检查导出的 L2 有效负载属性的以下字段是否与 L2 块相等：

- `parent_hash`
- `timestamp`
- `randao`
- `fee_recipient`
- `transactions_list`（首先是长度，然后是每个编码交易的相等性，包括存款）

如果合并成功，forkchoice 更改将按照上一节所述进行同步。

如果合并失败，将立即处理 L2 有效负载属性，如下节所述。有效负载属性的选择有利于先前不安全的 L2 区块，从而在当前安全区块之上创建 L2 链重组。立即处理新的替代属性使像 go-ethereum 这样的执行引擎能够实施更改，因为可能不支持链末端的线性倒带。

#### L1-sync：有效负载属性处理

如果安全和不安全的 L2 头相同（无论是否由于合并失败），我们将 L2 有效负载属性发送到执行引擎以构造成正确的 L2 块。这个 L2 块将成为新的 L2 安全头和不安全头。

如果由于验证错误（即块中存在无效交易或状态转换）而无法将从批次创建的有效负载属性插入到链中，则应删除该批次并且不应提前安全头。引擎队列将尝试使用批次队列中该时间戳的下一个批次。如果未找到有效批次，则汇总节点将创建仅存款批次，该批次应始终通过验证，因为存款始终有效。

与执行引擎[通信部分详细介绍了通过执行引擎 API 与执行引擎](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine-api)进行交互。

然后使用以下序列处理有效负载属性：

- ```
  engine_forkchoiceUpdatedV1
  ```

  具有当前阶段的 forkchoice 状态，以及开始块构建的属性。

  - 必须禁用非确定性来源（例如交易池）才能重建预期的块。

- `engine_getPayload`通过上一步结果中的有效负载 ID 检索有效负载。

- `engine_newPayload`将新的有效负载导入到执行引擎中。

- `engine_forkchoiceUpdatedV1`为了使新的有效负载规范化，现在更改了`safe`和`unsafe`字段以引用有效负载，并且没有有效负载属性。

引擎API错误处理：

- 对于 RPC 类型错误，应在以后的步骤中重新尝试有效负载属性处理。
- 在有效负载处理错误时，必须删除属性，并且必须保持 forkchoice 状态不变。
  - 最终，派生管道将产生替代的有效负载属性，无论是否有批次。
  - 如果有效负载属性仅包含存款，那么如果这些属性无效，则这是一个严重的推导错误。
- 在出现 forkchoice-state 有效性错误时，必须重置派生管道以恢复到一致状态。

#### 处理不安全的有效负载属性

如果没有分叉选择更新或 L1 数据需要处理，并且如果下一个可能的 L2 块已经可以通过不安全的来源（例如通过 p2p 网络发布的定序器）获得，那么它会被乐观地处理为“不安全”块。这将后续的推导工作减少到在满意的情况下仅与 L1 合并，并且使用户能够比 L1 确认 L2 批次更快地看到 L2 链的头部。

要处理不安全的有效负载，有效负载必须：

- 具有比当前安全L2头更高的块号。
  - 安全的 L2 头只能因 L1 重组而被重组。
- 有一个与当前不安全的 L2 头匹配的父块哈希。
  - 这可以防止执行引擎单独同步不安全的 L2 链中的较大间隙。
  - 这可以防止不安全的 L2 块重组其他先前验证的 L2 块。
  - 此检查可能会在未来版本中更改以采用例如 L1 快照同步协议。

然后通过以下序列处理有效负载：

- `engine_newPayloadV1`：处理有效负载。它还没有成为规范。
- `engine_forkchoiceUpdatedV1`：将有效负载设置为规范的不安全L2头，并保留安全/最终确定的L2头。

引擎API错误处理：

- 对于 RPC 类型的错误，应在以后的步骤中重新尝试有效负载处理。
- 当有效负载处理错误时，必须删除有效负载，并且不能将其标记为规范。
- 在出现 forkchoice-state 有效性错误时，必须重置派生管道以恢复到一致状态。

### 重置管道

[例如，如果我们检测到 L1重组（重组）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#chain-re-organization)，则可以重置管道。 **这使得汇总节点能够处理 L1 链重组事件。**

重置会将管道恢复到产生与完整 L2 推导过程相同的输出的状态，但从现有的 L2 链开始，向后遍历足以与当前的 L1 链协调。

请注意，该算法涵盖了几个重要的用例：

- 初始化管道而不从 0 开始，例如当汇总节点使用现有引擎实例重新启动时。
- 如果管道与执行引擎链不一致（例如，当引擎同步/更改时），则恢复管道。
- 当 L1 链重组时恢复管道，例如，较晚的 L1 块被孤立，或者更大的证明失败。
- 初始化管道，以在防错程序内使用先前的 L1 和 L2 历史记录派生有争议的 L2 块。

处理这些情况还意味着节点可以配置为通过 0 次确认立即同步 L1 数据，因为如果 L1 稍后将数据识别为规范数据，它可以撤消更改，从而实现安全的低延迟使用。

首先重置引擎队列，以确定继续推导的 L1 和 L2 起点。此后，其他阶段彼此独立地重置。

#### 寻找同步起点

要找到起点，相对于链头向后移动有几个步骤：

1. 查找当前L2 forkchoice状态

   - 如果找不到`finalized`区块，则从基岩创世区块开始。
   - 如果`safe`找不到块，则回退到该`finalized`块。
   - 该`unsafe`块应始终可用并与上述内容一致（在罕见的发动机损坏恢复情况下可能不会出现，这一点正在审查中）。

2. 找到第一个具有合理 L1 参考的 L2 块作为新的

   ```
   unsafe
   ```

   起点，从前一个 开始

   ```
   unsafe
   ```

   ，回到

   ```
   finalized
   ```

   ，不再进一步。

   - 合理的 iff：L2 块的 L1 起源已知且规范，或者未知且块号领先于 L1。

3. 找到第一个 L2 块，其 L1 参考早于测序窗口，作为新的

   ```
   safe
   ```

   起点，从上面看似合理的

   ```
   unsafe
   ```

   头部开始，返回到

   ```
   finalized
   ```

   不再进一步。

   - 如果在任何时候 L1 起源已知但不规范，则`unsafe`头部将被修改为当前的父级。
   - 具有已知规范 L1 起源的最高 L2 块被记住为`highest`。
   - 如果在任何时候块中的 L1 原点破坏了推导规则，则会出错。腐败包括：
     - L1 起源块编号或父哈希与父 L1 起源不一致
     - `0`L1 序列号不一致（对于 L1 原点更改，始终更改为，`1`如果不是则递增）
   - 如果 L2 块的 L1 原点`n`比 的 L1 原点早`highest`超过一个序列窗口，并且`n.sequence_number == 0`，则 的父 L2 块`n`将是`safe`起始点。

4. L2`finalized`块仍然作为`finalized`起点。

5. 查找 L1 引用早于通道超时的第一个 L2 块

   - 我们称之为该块引用的 L1 原点`l2base`将是`base`L2 管道派生的源：通过从这里开始，各个阶段可以缓冲任何必要的数据，同时丢弃不完整的派生输出，直到 L1 遍历赶上实际的 L2 安全头。

在遍历 L2 链时，实现可能会进行健全性检查，确保与现有的 forkchoice 状态相比，起点永远不会设置得太远，以避免由于配置错误而导致的密集重组。

实施者注意：步骤 1-4 称为`FindL2Heads`。第 5 步当前是引擎队列重置的一部分。这可能会改变以将起点搜索与裸重置逻辑隔离。

#### 重置推导阶段

1. L1 遍历：从 L1 开始，`base`作为下一阶段拉取的第一个块。
2. L1 检索：清空之前的数据，并获取`base`L1 数据，或者将获取工作推迟到稍后的管道步骤。
3. 帧队列：清空队列。
4. 通道库：清空通道库。
5. 通道读取器：重置任何批量解码状态。
6. 批处理队列：清空批处理队列，用作`base`初始 L1 参考点。
7. 有效负载属性推导：清空任何批次/属性状态。
8. 引擎队列：
   - 使用同步起始点状态初始化 L2 forkchoice 状态。( `finalized`// `safe`) `unsafe`_
   - 将平台的 L1 参考点初始化为`base`。
   - 需要将 forkchoice 更新作为第一个任务
   - 重置所有最终数据

如有必要，从 开始的阶段`base`可以根据块中编码的数据初始化其系统配置`l2base`。

#### 关于合并后重组

请注意，[合并](https://ethereum.org/en/upgrades/merge/)后，重组的深度将受到[L1 最终确定延迟的](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/#finality)限制 （2 个 L1 信标周期，或大约 13 分钟，除非超过 1/3 的网络始终不同意）。新的 L1 区块可能会在每个 L1 信标周期（大约 6.4 分钟）完成，并且根据这些最终信号和批量包含，派生的 L2 链也将变得不可逆转。

请注意，这种形式的最终确定仅影响输入，然后节点可以通过从这些不可逆输入以及设置的协议规则和参数再现链来主观地认为该链是不可逆的。

然而，这与 L1 上发布的输出完全无关，L1 需要诸如防错或 zk 证明之类的证明形式才能最终确定。像 L1 提款这样的乐观汇总输出只有在经过一周且没有争议（故障证明挑战窗口）后才会被标记为“最终确定”，这是与权益证明最终确定的名称冲突。

------

# 派生有效负载属性

对于从 L1 数据派生的每个 L2 块，我们需要构建[有效负载属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#payload-attributes)，由对象的[扩展版本](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#extended-payloadattributesv1)表示[`PayloadAttributesV1`](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md#executionpayloadv1)，其中包括附加`transactions`和`noTxPool`字段。

此过程发生在验证者节点运行的有效负载属性队列期间，以及排序器节点运行的块生产期间（如果交易是批量提交的，排序器可能会启用交易池使用）。

## 导出交易列表

对于定序器要创建的每个 L2 块，我们从与目标 L2 块编号匹配的[定序器批次开始。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencer-batch)如果 L1 链不包含目标 L2 区块编号的批次，则这可能是一个空的自动生成批次。[请记住](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#batch-format)，该批次包括[排序纪元](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing-epoch)号、L2 时间戳和事务列表。

该块是[测序 epoch](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing-epoch)的一部分，其编号与 L1 块的编号（其*[L1 origin](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-origin)*）匹配。该 L1 区块用于派生 L1 属性和（对于该纪元中的第一个 L2 区块）用户存款。

因此，一个[`PayloadAttributesV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#extended-payloadattributesv1)对象必须包含以下事务：

- 一笔或多笔

  存入交易

  ，有两种：

  - 单一*[L1属性存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-attributes-deposited-transaction)*，源于L1本源。
  - 对于纪元中的第一个 L2 区块，零个或多个*[用户存款交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#user-deposited-transaction)*，源自L1 来源的[收据。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#receipt)

- 零个或多个*[排序交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#sequencing)*：由 L2 用户签名的常规交易，包含在排序器批次中。

事务**必须**按此顺序出现在有效负载属性中。

L1 属性是从 L1 区块头读取的，而存款是从 L1 区块的[收据](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#receipt)中读取的。有关如何将存款编码为日志条目的详细信息，请参阅[**存款合约规范。**](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-contract)

## 构建单独的有效负载属性

导出交易列表后，rollup 节点构造[`PayloadAttributesV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#extended-payloadattributesv1)如下：

- `timestamp`设置为批次的时间戳。
- `random`被设置为`prev_randao`L1块属性。
- `suggestedFeeRecipient`设置为 Sequencer Fee Vault 地址。请参阅[费用库](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#fee-vaults)规范。
- `transactions`是派生交易的数组：存款交易和排序交易，全部用[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)编码。
- `noTxPool`设置为，以在构造块时`true`使用上面的列表。`transactions`
- `gasLimit`设置为此负载的[系统配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#system-configuration)`gasLimit`中的当前值。