- 面向对象模型思考：[Object-Capability Model | Cosmos SDK](https://docs.cosmos.network/main/core/ocap)

- ABCI 允许替换共识，目前只有cometBFT

- 开发人员可以自由探索各种权衡（例如，验证器的数量与事务吞吐量、异步中的安全性与可用性等）和设计选择（用于存储的DB或IAVL树、UTXO或帐户模型等） ？？

- 开发人员可以实现代码的自动执行。在Cosmos SDK中，逻辑可以在每个块的开始和结束时自动触发。他们还可以自由选择应用程序中使用的加密库，而不是在虚拟机区块链的情况下受到底层环境的限制 ？？

- 开发人员不受底层虚拟机提供的加密功能的约束。他们可以使用自己的自定义密码，并依赖于经过良好审计的加密库。

- 特定应用程序区块链的主要好处之一是主权。去中心化的应用程序是一个涉及许多参与者的生态系统：用户、开发人员、第三方服务等等。当开发者在许多去中心化应用共存的虚拟机区块链上构建时，应用程序的社区与底层区块链的社区不同，后者在治理过程中取代前者。如果出现错误或需要新功能，应用程序的利益相关者几乎没有升级代码的余地。如果底层区块链的社区拒绝采取行动，什么都不会发生。

  这里的根本问题是应用程序的治理和网络的治理不一致。这个问题可以通过特定于应用程序的区块链来解决。由于特定于应用程序的区块链专门用于操作单个应用程序，因此应用程序的利益相关者可以完全控制整个链。这确保了如果发现漏洞，社区不会陷入困境，并且可以自由选择如何发展。



## Application-Specific Blockchains

```markdown
                ^  +-------------------------------+  ^
                |  |                               |  |   Built with Cosmos SDK
                |  |  State-machine = Application  |  |
                |  |                               |  v
                |  +-------------------------------+
                |  |                               |  ^
Blockchain node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   CometBFT
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v
```



## cometBFT

CometBFT共识算法使用一组称为Validators的特殊节点。验证器负责将交易块添加到区块链中。在任何给定的块上，都有一个验证器集V。**算法选择V中的验证器作为下一个块的提议者**。如果超过三分之二的V在其上签署了prevote和precommit，并且它包含的所有事务都是有效的，则该块被认为是有效的。**验证程序集可以通过在状态机中编写的规则进行更改?????**。

## ABCI 

参考bc_architecture

## APP types 

- **baseApp**:`baseapp` implements most of the core logic for the application, including all the [ABCI methods](https://docs.cometbft.com/v0.37/spec/abci/) and the [routing logic](https://docs.cosmos.network/main/core/baseapp#routing).
- **A list of store keys**: Each module uses one or multiple stores in the multistore to persist their part of the state. These stores can be accessed with specific keys that are declared in the `app` type.
- **A list of module's `keeper`s**：which handles reads and writes for this module's store(s)
- **Appcodec**:used to serialize and deserialize data structures in order to store them, as stores can only persist `[]bytes`. The default codec is [Protocol Buffers](https://docs.cosmos.network/main/core/encoding).
- Moudle manager:The module manager is an object that contains a list of the application's modules.like registering their [`Msg` service](https://docs.cosmos.network/main/core/baseapp#msg-services) and [gRPC `Query` service](https://docs.cosmos.network/main/core/baseapp#grpc-query-services), or setting the order of execution between modules for various functions like [`InitChainer`](https://docs.cosmos.network/main/basics/app-anatomy#initchainer), [`BeginBlocker` and `EndBlocker`](https://docs.cosmos.network/main/basics/app-anatomy#beginblocker-and-endblocker).

### newSimApp 

[initchainer]: 由cometbft 发到App里的消息，主要用来调用各个模块的Initgenesis 函数。

[beginBlocker&endBlocker]:cometbft发到App的两个消息，区块开始前和结束后，添加一些自动执行的任务

## moudles

[Application Module Interface]:需要实现[`AppModuleBasic`](https://docs.cosmos.network/main/building-modules/module-manager#appmodulebasic) and [`AppModule`](https://docs.cosmos.network/main/building-modules/module-manager#appmodule)

[msg services]: 一个Msg服务用于处理消息，另一个gRPC查询服务用于处理查询。如果我们将模块视为一个状态机，那么Msg服务就是一组状态转换RPC方法。每个Protobuf-Msg服务方法都与Protobuf请求类型1:1相关，该请求类型必须实现sdk.Msg接口。请注意，sdk.Msgs捆绑在事务中，每个事务包含一个或多个消息

{diliver tx}: 节点收到有效块，然后cometBFT 通过diliver tx 传给app 去处理，然后1 decode, 2,validate tx , 3 route to module 

一般会在tx.proto中定义自己的Msg服务。

{gRPC `Query` Services}: 了解grpc原理 🚩， rest路由

## Transaction Lifecycle

- 可以指定超时高度

- 由cometbft 发送check tx 给app去检查交易是否正确

- [antehandler]: 

- AnteHandlers: 实践中经常用于执行签名验证、气体计算、费用扣除和其他与区块链交易相关的核心操作。

  例如，身份验证模块AnteHandler检查并递增序列号，检查签名和帐号，并从交易的第一个签名者那里扣除费用——所有状态更改都是使用checkState进行的

「mempool」:验证器节点保留一个内存池以防止重放攻击，就像完整节点一样，但也将其用作未经确认的事务池，为包含块做准备。请注意，即使Tx在这个阶段通过了所有检查，以后仍有可能被发现无效，因为CheckTx并没有完全验证事

「state chhange 」:

**达成共识的下一步是执行事务以完全验证它们**。所有从正确的提议者那里接收到块提议的完整节点都通过**调用ABCI函数BeginBlock**、每个事务的**DeliverTx**和**EndBlock**来执行事务。虽然每个完整节点都在本地运行所有内容，但由于消息的状态转换是确定性的，并且事务在块建议中是明确排序的，因此该过程会产生一个单一的、明确的结果。

```markdown
        -----------------------
        |Receive Block Proposal|
        -----------------------
                  |
              v
        -----------------------
        | BeginBlock          |
        -----------------------
                  |
              v
        -----------------------
        | DeliverTx(tx0)      |
        | DeliverTx(tx1)      |
        | DeliverTx(tx2)      |
        | DeliverTx(tx3)      |
        |   .         |
        |   .         |
        |   .         |
        -----------------------
                  |
              v
        -----------------------
        | EndBlock        |
        -----------------------
                  |
              v
        -----------------------
        | Consensus       |
        -----------------------
                  |
              v
        -----------------------
        | Commit          |
        -----------------------
```

「deliever tx」:

- 解码事务
- Checks和AnteHandler : 事务检查
- MsgServiceRouter： 消息路由到指定模块的处理器
- msg service： Protobuf-Msg服务负责执行Tx中的每个消息，并使状态转换持续到deliveryTxState ？？
- Posthandler:PostHandlers在消息执行之后运行。如果失败，那么runMsgs和PostHandlers的状态更改都将被恢复
- GasMeter: 跟踪交易执行所需要的气体

「commIt」：

最后一步是节点**提交块和状态更改**。**Validator节点执行执行状态转换的前一步，以验证事务，然后对块进行签名以进行确认**。非Validator的完整节点不参与共识，即他们不能投票，但会听取投票，以了解他们是否应该提交状态更改。

当他们收到足够的验证器投票（2/3+按投票权加权的预提交）时，完整的节点提交到要添加到区块链的新块，并在应用层中完成状态转换。生成一个新的状态根，作为状态转换的merkle证明。**应用程序使用从Baseapp继承的Commit ABCI方法；它通过将deliveryState写入应用程序的内部状态来同步所有状态转换**。一旦提交了状态更改，checkState就会从最近提交的状态重新开始，deliveryState会重置为nil，以保持一致并反映更改

请注意，并非所有区块都有相同数量的交易，共识可能会导致**零区块或完全没有区块？？？？？？**。在公共区块链网络中，验证器也可能是拜占庭式的或恶意的，这可能会阻止Tx在区块链中提交。可能的恶意行为包括提议者决定通过将Tx从块中排除来审查Tx，或者验证器投票反对该块。

此时，Tx的事务生命周期已经结束：节点已经验证了它的有效性，通过执行状态更改来传递它，并提交这些更改。Tx本身以[]字节的形式存储在一个块中，并附加到区块链中

# core concepts

## baseApp 

cosmos sdk的基本类型，实现了ABCI接口（所以开发者不需要考虑实现），用于和cometBFT通信

```go
type App struct {
  // reference to a BaseApp
  *baseapp.BaseApp

  // list of application store keys

  // list of application keepers

  // module manager
}
```

## type definition

baseapp 的关键组件如下： 

- CommitMutiStore: 多个存储，对应相应的模块，一笔模块相关的交易会改变相关的存储
- Database: 持久化 commitMutiStore
- Msg service router :  将sdk.msg 路由到适当模块的msg service 处理，不同于ABCI Msg 
- Grpc query router:  路由到适当模块处理查询
- Txdecoder: 解码cometBFT 传过来的事务字节
- AntHandler: 用于在收到交易时处理签名验证、费用支付和其他消息执行前检查。它在CheckTx/RecheckTx和DeliverTx期间执行。
- [`InitChainer`](https://docs.cosmos.network/main/basics/app-anatomy#initchainer), [`BeginBlocker` and `EndBlocker`](https://docs.cosmos.network/main/basics/app-anatomy#beginblocker-and-endblocker):这些是应用程序从底层CometBFT引擎接收**InitChain、BeginBlock和EndBlock ABCI消息**时执行的函数

各个阶段的缓存状态：

- checkstate : 在CheckTx期间更新，并在提交时重置
- Deliverstate:在DeliverTx期间更新，在Commit时设置为nil，并在BeginBlock上重新初始化。
- processProposalState:processProposalState 更新
- prepareProposalState: prepareProposalState更新

## 状态更新 需要重点了解原理

我们通过使用一种称为**存储分支的机制？？？？**（由CacheWrap函数执行）导出四个易失性状态

![Types](https://docs.cosmos.network/assets/images/baseapp_state-c6660bdfda8fa3aeb44239780b465ecc.png)

## InitChain State Updates

在InitChain过程中，通过**分支根？？？**CommitMultiStore来设置四种易失性状态，checkState、prepareProposalState、processProposalState和deliveryState。任何后续的读取和写入都发生在**CommitMultiStore的分支版本上？？？**。为了避免不必要的到主状态的往返，对分支存储的所有读取都被缓存。

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

在InitChain过程中，RequestInitChaine提供ConsensusParams，其中除了证据参数外，还包含与区块执行相关的参数，如最大气体和大小。如果这些参数不是nil，则在BaseApp的ParamStore中进行设置。在幕后，ParamStore由x/consensus_params模块管理。这允许**通过链上治理来调整参数，这是一个很好的去中心化设计**

## Service Routers

当应用程序接收到消息和查询时，必须将它们路由到适当的模块才能进行处理。路由是通过BaseApp完成的，它为消息保存一个msgServiceRouter，为查询保存一个grpcQueryRouter。

### msg service router 

为此，BaseApp持有一个msgServiceRouter，它将完全限定的服务方法（字符串，在每个模块的Protobuf-Msg服务中定义）映射到相应模块的MsgServer实现

应用程序的模块管理器（通过RegisterServices方法）使用所有路由初始化应用程序的msgServiceRouter，模块管理器本身使用应用程序构造函数中的所有应用程序模块初始化.

### grpc query router 

BaseApp拥有一个grpcQueryRouter，它将模块的完全限定服务方法（字符串，在其Protobuf Query gRPC中定义）映射到其QueryServer实现。grpcQueryRouter在查询处理的初始阶段被调用，可以通过直接向gRPC端点发送gRPC查询，也可以通过CometBFT RPC端点上的query ABCI消息。

与msgServiceRouter一样，grpcQueryRouter使用应用程序的模块管理器（通过RegisterServices方法）使用所有查询路由进行初始化，模块管理器本身使用应用程序构造函数中的所有应用程序模块进行初始化。

## ABCI msgs

ABCI 是一个通用接口（由comeBFT定义），通过它连接状态机和共识引擎。共识引擎会发出ABCI消息，供app 采取行动，消息如下： 

- ### Prepare Proposal

- Process Proposal

- CheckTx

- DeliverTx

- Initchain:主要用于初始化参数和状态

- begin block

- End block 

- commit 

- info

- query

## Transaction

### transaction process 

消息（或sdk.Msgs）是特定于模块的对象，用于触发所属模块范围内的状态转换。模块开发人员通过向Protobuf-Msg服务添加方法来定义模块的消息，并实现相应的MsgServer。

每个sdk.Msgs都与一个Protobuf-Msg服务RPC相关，该服务在每个模块的tx.proto文件中定义。SDK应用程序路由器会自动将每个SDK.Msg映射到相应的RPC。Protobuf为每个模块Msg服务生成一个MsgServer接口，模块开发人员需要实现这个接口。这种设计将更多的责任交给了模块开发人员，允许应用程序开发人员重用通用功能，而不必重复实现状态转换逻辑。

要了解有关Protobuf-Msg服务以及如何实现MsgServer的更多信息，请单击此处。

虽然消息包含状态转换逻辑的信息，但事务的其他元数据和相关信息存储在TxBuilder和Context中

## context

包装了go 的context ,设计理念以及他的应用场景？？？？上下文是事务处理不可或缺的一部分，因为它允许模块轻松访问多存储中各自的存储，并检索事务上下文，如块头和gadmeter

### Store branching （context的用途）

 Context包含一个MultiStore，它允许使用CacheMultiStore进行分支和缓存功能（缓存MultiStore中的查询以避免将来的往返）。每个KVStore都在一个安全且隔离的临时存储中进行分支。进程可以自由地将更改写入CacheMultiStore。如果在没有问题的情况下执行状态转换序列，则可以将存储分支提交到序列末尾的底层存储，或者在出现问题时忽略它们。上下文的使用模式如下：

进程从其父进程接收上下文ctx，该上下文ctx提供执行该进程所需的信息。

ctx.ms是一个分支存储，即多存储的一个分支，这样进程就可以在执行时对状态进行更改，而无需更改原始的alctx.ms。这有助于保护底层多存储，以防在执行过程中的某个时刻需要恢复更改。

进程可以在执行时从ctx读取和写入。它可以调用一个子流程，并根据需要将ctx传递给它。

当子流程返回时，它会检查结果是成功还是失败。如果发生故障，则无需执行任何操作——只需丢弃分支ctx。如果成功，可以通过Write（）将对CacheMultiStore所做的更改提交到原始ctx.ms。

## store 

cosmos sdk 的存储设计架构：

```markdown
+-----------------------------------------------------+
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 1 - Manage by keeper of Module 1  |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 2 - Manage by keeper of Module 2  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 3 - Manage by keeper of Module 2  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 4 - Manage by keeper of Module 3  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 5 - Manage by keeper of Module 4  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|                    Main Multistore                  |
|                                                     |
+-----------------------------------------------------+

                   Application's State
```

### multistore 

multistore 是 store的存储，Cosmos SDK中使用的Multistore的主要类型是CommitMultiStore，它是Multistore接口的扩展

！！！！？？？？[I AVL Tree 详解](https://github.com/cosmos/iavl)

[ival doc ](https://github.com/cosmos/iavl/blob/master/docs/overview.md)

Grpc 网关。rest路由

## Object-Capability Model

对象能力系统的结构特性有利于代码设计中的模块化，并确保代码实现中的可靠封装。

这些结构属性有助于分析对象能力程序或操作系统的一些安全属性。其中一些属性，特别是信息流属性，可以在对象引用和连接级别进行分析，而不依赖于确定对象行为的代码的任何知识或分析。

因此，在存在包含未知且可能是恶意代码的新对象时，可以建立和维护这些安全属性。

这些结构特性源于管理对现有对象的访问的两个规则：

只有当对象A包含对B的引用时，对象A才能向B发送消息。

只有当对象A接收到包含对C的引用的消息时，对象A才能获得对C的参考。由于这两个规则，一个对象只能通过预先存在的引用链获得对另一个对象的引用。简而言之，“只有连接才能产生连接。

[对象模型](https://en.wikipedia.org/wiki/Object-capability_model)

## p ro to bu f

https://buf.build/cosmos/cosmos-sdk/docs/main

## In-Place Store Migrations

必须非常理解，比较危险，手动升级模块

[这个需要自己动手尝试](https://docs.cosmos.network/main/core/upgrade)

## 构建模块

Cosmos SDK可以被认为是区块链开发的Ruby on Rails。它的核心提供了每个区块链应用程序所需的基本功能，如ABCI的样板实现，以与底层共识引擎通信，多存储持久状态，形成完整节点的服务器和处理查询的接口。

Cosmos SDK模块可以被视为状态机中的小型状态机。它们通常使用主多存储中的一个或多个KVStores以及消息类型的子集来定义状态的子集。这些消息由Cosmos SDK核心的主要组件之一BaseApp路由到定义它们的模块Protobuf-Msg服务

### 如何处理构建模块

设计原则如下： 

- **Composability**：思考是否要和核心模块或其他模块集成
- **Specialization**：模块应该专门化的，不得将多个功能集成到一个模块中，关注点分离设计思想，便于升级
- **Capabilities**：大多数模块需要读取和/或写入其他模块的存储区。然而，在开源环境中，某些模块可能是恶意的。这就是为什么模块开发人员不仅需要仔细考虑他们的模块如何与其他模块交互，还需要仔细考虑如何访问模块的存储。Cosmos SDK采用了一种面向功能的方法来实现模块间安全。这意味着由模块定义的每个存储都由一个密钥访问，该密钥由模块的管理员持有。该管理员定义了如何访问商店以及在什么条件下访问。访问模块的存储是通过将引用传递给模块的keeper来完成的。



### Application Module Interfaces

应用程序模块接口的存在有助于将模块组合在一起，形成一个功能强大的Cosmos SDK应用程序。有4个主要的应用程序模块接口：

- Appmoudlebasic:用于独立的模块功能。
- Appmoudle:用于相互依赖的模块功能
- appmoudleGenesis:用于相互依赖的genesis相关模块功能
- GenesisOnlyAppModule:定义仅具有导入/导出功能的AppModule

#### appmoudlebasic

```go
// AppModuleBasic is the standard form for basic non-dependant elements of an application module.
type AppModuleBasic interface {
	HasName
	RegisterLegacyAminoCodec(*codec.LegacyAmino)
	RegisterInterfaces(codectypes.InterfaceRegistry)

	// client functionality
  //为模块注册gRPC路由
	RegisterGRPCGatewayRoutes(client.Context, *runtime.ServeMux)
	GetTxCmd() *cobra.Command
	GetQueryCmd() *cobra.Command
}
```

#### appmoudle

```go
// AppModule is the form for an application module. Most of
// its functionality has been moved to extension interfaces.
type AppModule interface {
	AppModuleBasic
}
```



### m s g SERVICES   时序图怎么看？？？？？

此图显示了Protobuf-Msg服务的典型结构，以及消息如何在模块中传播。

![probuf-msg structure](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/transaction_flow.svg)

## keeper



### module Genesis

每当进行状态导出时，都会执行ExportGenesis方法。它获取模块管理的状态子集的最新已知版本，并从中创建一个新的GenesisState。这主要用于需要通过硬分叉升级链时

grpc网关将REST调用转换为grpc调用，这对于不使用grpc的客户端可能很有用。 auto cli 包的使用

## 升级模块，需要实际操作



## Modules depinject-ready  新加的内容

