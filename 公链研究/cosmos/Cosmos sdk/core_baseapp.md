## 概述

BaseApp是实现Cosmos SDK应用程序核心的基本类型，用于状态机与底层共识引擎（例如CometBFT）通信。

BaseApp的目标是提供Cosmos SDK应用程序的基本层，开发人员可以轻松扩展该层以构建自己的自定义应用程序。通常，开发人员会为他们的应用程序创建一个自定义类型.

## 初始化参数

- [`CommitMultiStore`](https://docs.cosmos.network/main/core/store#commitmultistore):
- [`Msg` Service Router](https://docs.cosmos.network/main/core/baseapp#msg-service-router): 
- [gRPC Query Router](https://docs.cosmos.network/main/core/baseapp#grpc-query-router): 
- [`TxDecoder`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/types#TxDecoder): 
- [`AnteHandler`](https://docs.cosmos.network/main/core/baseapp#antehandler): 
- [`InitChainer`](https://docs.cosmos.network/main/basics/app-anatomy#initchainer)
-  BeginBlocker
- [EndBlocker`](https://docs.cosmos.network/main/basics/app-anatomy#beginblocker-and-endblocker):

## 缓存状态参数

- checkState ：此状态在CheckTx期间更新，并在提交时重置。
- deliverState ：此状态在DeliverTx期间更新，在Commit时设置为nil，并在BeginBlock上重新初始化
- processProposalState
- prepareProposalState

### 状态更新

BaseApp维护四个主要的易失性状态和一个根或主状态。主状态是应用程序的规范状态，而不稳定状态checkState、deliveryState、prepareProposalState、processPreposalState用于处理提交过程中主状态之间的状态转换。

在内部，只有一个CommitMultiStore，我们称之为主状态或根状态。从这个根状态，我们通过使用一种称为存储分支的机制（由CacheWrap函数执行）导出四个易失性状态。类型如下所示：

![Types](https://docs.cosmos.network/assets/images/baseapp_state-c6660bdfda8fa3aeb44239780b465ecc.png)

## InitChain State Updates

在InitChain过程中，通过分支根CommitMultiStore来设置四种易失性状态，checkState、prepareProposalState、processProposalState和deliveryState。任何后续的读取和写入都发生在CommitMultiStore的分支版本上。为了避免不必要的到主状态的往返，对分支存储的所有读取都被缓存。

![InitChain](https://docs.cosmos.network/assets/images/baseapp_state-initchain-62da1a79d5dd67a6d1ab07f2805040da.png)

### CheckTx State Updates

在CheckTx期间，基于根存储中最后一个提交状态的checkState用于任何读取和写入。在这里，我们只执行AnteHandler，并验证事务中每个消息都存在服务路由器。注意，当我们执行AnteHandler时，我们对已经分支的checkState进行分支。这有一个副作用，即如果AnteHandler失败，状态转换将不会反映在checkState中——即checkState只在成功时更新。

![CheckTx](https://docs.cosmos.network/assets/images/baseapp_state-checktx-5bb98c17c37b2b93e98cc681b6c1c9d6.png)

## PrepareProposal State Updates

在PrepareProposal期间，通过分支根CommitMultiStore来设置prepareProposalState。prepareProposalState用于在PrepareProposal阶段发生的任何读取和写入。该函数使用内存池的Select（）方法来迭代事务。然后调用runTx，它对每个事务进行编码和验证，然后执行AnteHandler。如果成功，将返回有效的事务，包括在执行建议期间生成的事件、标记和数据。所描述的行为是默认处理程序的行为，应用程序可以灵活地定义自己的自定义内存池处理程序。

![ProcessProposal](https://docs.cosmos.network/assets/images/baseapp_state-prepareproposal-bc5c8099ad94b823c376d1bde26d584a.png)

## ProcessProposal State Updates

在ProcessProposal过程中，processProposalState是根据根存储中的最后一个提交状态设置的，用于处理从验证器接收的签名提案。在这种状态下，调用runTx并执行AnteHandler，并且在这种状态中使用的上下文是用来自标头和主状态的信息构建的，包括也设置的最低天然气价格。我们再次强调，所描述的行为是默认处理程序的行为，应用程序可以灵活地定义自己的自定义内存池处理程序

![ProcessProposal](https://docs.cosmos.network/assets/images/baseapp_state-processproposal-486265a078da51c6f72ce7248e8021b3.png)

### BeginBlock State Updates

在BeginBlock期间，deliveryState被设置为在随后的DeliverTx ABCI消息中使用。deliveryState基于根存储中最后一个提交的状态，并且是分支的。请注意，deliveryState在提交时设置为nil。

![BeginBlock](https://docs.cosmos.network/assets/images/baseapp_state-begin_block-dfbb5abb42a34744e7fe08df2f8d01e2.png)

### DeliverTx State Updates

DeliverTx的状态流几乎与CheckTx相同，只是deliveryState上发生状态转换，并且执行事务中的消息。与CheckTx类似，状态转换发生在双分支状态deliveryState上。成功执行消息会导致写入被提交到deliveryState。请注意，如果消息执行失败，来自AnteHandler的状态转换将被持久化。

![DeliverTx](https://docs.cosmos.network/assets/images/baseapp_state-deliver_tx-5999f54501aa641d0c0a93279561f956.png)

### Commit State Updates

在提交过程中，deliveryState中发生的所有状态转换最终都会写入根CommitMultiStore，然后提交到磁盘并生成新的应用程序根哈希。这些状态转换现在被认为是最终的。最后，checkState设置为新提交的状态，deliveryState设置为nil以在BeginBlock上重置

![Commit](https://docs.cosmos.network/assets/images/baseapp_state-commit-247373784511c1db3ed2175551b22abb.png)

## ParamStore

在InitChain过程中，RequestInitChaine提供ConsensusParams，其中除了证据参数外，还包含与区块执行相关的参数，如最大气体和大小。如果这些参数不是nil，则在BaseApp的ParamStore中进行设置。在幕后，ParamStore由x/consensus_params模块管理。这允许通过链上治理来调整参数

## Service Routers

当应用程序接收到消息和查询时，必须将它们路由到适当的模块才能进行处理。路由是通过BaseApp完成的，它为消息保存一个msgServiceRouter，为查询保存一个grpcQueryRouter。

