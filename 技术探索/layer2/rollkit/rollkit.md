## 优势

1. 共享DA层安全性
2. 可扩展性：rollkit部署在专门的DA层，如celestia,且有自己的专用计算资源
3. 可定制：自定义DA或者执行环境
4. 主权：可以部署主权rollup
5. 支持ABCI兼容的状态机，以及自定义的执行环境，包括EVM兼容



## 如何根据cosmos SDk构建主权rollup

1. 使用cosmos sdk 创建与rollkit兼容的rollup链
2. 使用现有的基于cosmos sdk构建的APP链，将其部署rollkit 汇总

## 构建结算层

结算层非常适合那些希望避免部署主权汇总的开发人员。它们为汇总提供了一个平台，以验证证据并解决争议。此外，它们还充当了汇总的中心，以促进共享同一结算层的汇总之间的最小化信任的代币转移和流动性共享。将结算层视为一种特殊类型的执行层。[目前流行celestia](https://celestia.org/learn/modular-settlement-layers/settlement-in-the-modular-stack/)或者restaking（eginelayer）或者 polygon aval

## 主要组件

Roll-up sequencer节点从用户那里收集事务，将它们聚合成块，并将块发布到数据可用性（DA）层（如Celestia）上以进行排序和最终确定。完整节点执行和验证汇总块，在**乐观汇总**的情况下，在需要时传播欺诈证据。轻型客户端将接收标头，验证证据（欺诈、zk等），并验证有关状态的信任最小化查询。

![image-20230403133035905](/Users/carver/Library/Application Support/typora-user-images/image-20230403133035905.png)



## 改造

您想将Cosmos SDK应用程序更改为Rollkit汇总吗？没问题！您需要将Cosmos SDK Go依赖项替换为启用了Rollkit的版本，该版本可以在Rollkit/Cosmos SDK存储库中找到。

请注意，rollkit/cosmos sdk存储库遵循上游cosmos sdk的发布分支，但额外的好处是使用rollkit而不是Tendermint作为ABCI客户端。

别忘了用rollkit/tendermin替换另一个依赖项tendermin，它有一个增强的ABCI接口，包括州欺诈证明所需的方法。



## 数据可用性层

可以使用[通用接口](https://github.com/rollkit/rollkit/tree/main/da)访问数据可用性（DA）。这种设计允许与任何DA层无缝集成。新的实现可以通过编程方式插入，而不需要派生Rollkit。

DataAvailabilityLayerClient接口包括基本的生命周期方法（Init、Start、Stop）以及数据可用性方法（SubmitBlock、CheckBlockAvailability）。

BlockRetriever接口用于从数据可用性层同步完整节点。重要的是要记住，DA层块高度和汇总高度之间没有直接相关性。每个DA层块可以包含任意数量的汇总块。

### celestia 

Celestia是为Rollkit实现的数据可用性集成的一个示例。它通过celestiaorg/go-cnc软件包使用Celestia Node网关API。要在Celestia上部署Rollkit汇总，您还必须运行Celestia轻节点



## 节点组件

### 内存池

内存池的灵感来源于Tendermint内存池。默认情况下，事务是以先到先得（FCFS）的方式处理的。交易的排序可以在应用程序级别上实现；目前，这可以通过在CheckTx上返回优先级来实现，一旦我们支持ABCI++，也可以通过PrepareProposal和应用程序内存池来实现。

## 块管理器

块管理器包含通过Go通道进行通信的Go协程AggregationLoop、RetrieveLoop和SyncLoop。这些Go例程在Rollkit节点启动（OnStart）时运行。只有sequencer节点运行AggregationLoop，它根据BlockManager中的BlockTime控制块生产的频率，以便使用计时器进行汇总。

所有节点都运行SyncLoop，它会查找以下操作：

- 接收块标头：通过通道HeaderInCh接收块标头，Rollkit节点尝试用相应的块数据验证块。

- 接收块数据：通过通道blockInCh接收块体，Rollkit节点尝试验证块。

- 接收状态欺诈证明：状态欺诈证明是通过通道接收的FraudProofInCh和Rollkit节点试图验证它们。请注意，我们计划对完整节点进行配置，因为完整节点本身也会产生状态欺诈证明。

- 根据BlockManager中的DABlockTime，具有计时器的信号RetrieveLoop。

所有节点还运行RetrieveLoop，它负责与**数据可用性层**交互。它检查最后更新的DAHeight以检索具有SyncLoop发出的计时器DABlockTime的块。请注意，用于汇总的DA层的起始高度DAStartHeight在BlockManager中是可配置的。

## 节点类型

完整节点验证所有块，并为乐观汇总生成欺诈证据。由于他们完全验证了所有汇总块，因此他们不依赖于欺诈或有效性证明来确保安全。

轻节点（正在进行的工作）

轻型节点是对块头进行身份验证的轻型汇总节点，可以通过欺诈证明或有效性证明进行保护。建议低资源设备上的普通用户使用。运行轻型节点的用户可以对汇总的状态进行信任最小化查询。目前，Rollkit灯光节点仍在开发中。

序列器节点

汇总可以使用序列器节点。序列器是汇总的块生产者，负责将事务聚合为块，通常执行事务以生成状态根，供汇总的轻型客户端使用。

Rollkit计划支持多种不同的可插拔sequencer方案：

| Deploy in one-click            | Faster soft-confirmations than L1    | Control over rollup's transaction ordering | Atomic composability with other rollups | Censorship resistance | Implementation Status |                |
| ------------------------------ | ------------------------------------ | ------------------------------------------ | --------------------------------------- | --------------------- | --------------------- | -------------- |
| Centralized sequencer          | Requires spinning up a sequencer     | Yes ✅                                      | Yes ✅                                   | No ❌                  | Eventual ⏳*           | ✅ Implemented! |
| Decentralized sequencer        | Requires spinning up a sequencer set | Yes ✅                                      | Yes ✅                                   | No ❌                  | Real-time ⚡️           | Planned        |
| Shared decentralized sequencer | Yes ✅                                | Yes ✅                                      | No ❌                                    | Yes ✅                 | Real-time ⚡️           | Planned        |
| Pure fork-choice rule          | Yes ✅                                | No ❌                                       | Maybe 🟡                                 | Maybe 🟡               | Eventual ⏳            | Planned        |

## 状态有效性模式

**悲观（仅完整节点）**

悲观的汇总是一个汇总，仅支持完整的节点，该节点重新汇总汇总中的所有交易以检查其有效性。Rollkit默认情况下支持悲观的汇总。

悲观的汇总类似于Tether如何将比特币用作通过Omnilayer用作数据可用性层。

**乐观（欺诈证明）**（正在进行的工作）

Rollkit的当前设计由一个单序器组成，该测序器将张贴到DA层的单个序列和多个（可选）完整节点。音序器八卦块标题到完整节点，完整节点从DA层中获取块。然后，完整的节点在这些块中执行交易以更新其状态，然后在P2P网络上进行八卦块标题以进行Rollkit Light节点。

一旦启用了国家欺诈证明，当一个块包含欺诈状态过渡时，Rollkit Full节点可以通过在交易之间比较中间状态根（ISR）来检测到它，并生成可以在P2P网络上闲聊的状态欺诈证明节点。然后，这些Rollkit Light节点可以使用此状态欺诈证据来验证欺诈性状态过渡是否自行发生。

总体而言，只要系统中至少有一个诚实的完整节点将产生州欺诈证明，则州欺诈证明将在完整节点和轻节点之间实现信任最小化。

请注意，Rollkit州欺诈证明仍在进行中，并且需要在ABCI之上的新方法，特别是生成违法，验证违法和Getapphash。

您可以找到当前的详细设计以及将州[欺诈证明](https://github.com/rollkit/rollkit/blob/manav/state_fraud_proofs_adr/docs/lazy-adr/adr-009-state-fraud-proofs.md)推向此架构决策记录（ADR）中所需的剩余工作。

计划进行有效性（ZK）汇总，但Rollkit目前不支持。

## 交易流程

交易流程

汇总用户使用轻型节点与汇总P2P网络通信，主要原因有两个：

- 提交交易

- 八卦标题和欺诈证明

![image-20230403135148279](/Users/carver/Library/Application Support/typora-user-images/image-20230403135148279.png)

为了进行交易，用户向他们的轻节点提交一个交易，轻节点将交易传给一个完整的节点。在将事务添加到其内存池之前，完整节点会检查其有效性。有效的事务被包括在内存池中，而无效的事务被拒绝，并且用户的事务将不会被处理。

如果事务是有效的并且已经包含在内存池中，那么定序器可以将其添加到汇总块中，然后将其提交到数据可用性（DA）层。这将为用户带来成功的事务流，并相应地更新汇总的状态。

在块被提交到DA层之后，完整的节点下载并验证块。然而，有一种可能性是，定序器可能会恶意地将具有无效事务或状态的块提交给DA层。在这种情况下，汇总链的完整节点将认为该块无效。在乐观汇总的情况下，如果他们发现块无效，他们会生成欺诈证据，并在P2P网络中与其他完整节点和轻型节点进行八卦。

因此，上链将停止，网络将决定通过社会共识来分叉。未来，当去中心化测序仪方案到位时，将提供额外的选项，例如削减测序仪或选择另一个完整节点作为测序仪。但是，在任何情况下，都必须创建一个新的块并将其提交给DA层。您可以在这里阅读更多关于测序仪节点的信息。

