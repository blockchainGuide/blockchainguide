## cosmos sdk 简介

Cosmos SDK是一个开源框架，用于构建多资产公共权益证明（PoS）区块链，以及许可的授权证明（PoA）区块链。

Cosmos SDK的目标是让开发人员能够轻松地从头开始创建自定义区块链，从而与其他区块链进行本地互操作。基于SDK的区块链是由可组合模块构建的，构建在**CometBFT**之上。其中大多数是开源的，任何开发者都可以随时使用。任何人都可以为Cosmos SDK创建一个模块，集成已经构建的模块就像将它们导入区块链应用程序一样简单。

## cosmos 应用链设计架构

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

## 为什么要用cosmos sdk 构建链

1. Cosmos SDK是开源的，随着开源Cosmos SDK模块生态系统的发展，使用它构建复杂的去中心化平台将变得越来越容易。
2. 已经在生产环境中被许多知名项目使用
3. 灵活
   - 可以使用多种不同语言构建状态机，cosmos sdk 使用最广泛
   - 开发人员可以自行根据自己需求权衡，比如是否使用SDK中提供的账户模型或者DB等
   - 可以自己选择加密库，不受像拥有虚拟机区块链底层环境限制
4. 性能方面的优势
   - 基于cometBFT「**在整个行业中被广泛使用，并被认为是构建风险证明系统的黄金标准共识引擎**」共识，吞吐量远超Pow
   - 构建的特定应用链不会争抢资源和计算能力，比如以太坊的智能合约，经常会遇到交易堵塞，即使在layer2 ，也会受限。
   - 基于虚拟机来解释事务会显著增加计算复杂性
5. 安全
   - dis 不成熟的智能合约语言以及不安全的虚拟机
   - [基于对象能力模型](https://docs.cosmos.network/main/core/ocap)设计思想，使得cosmos sdk 提供了一个非常安全的环境 

----------

## cosmos sdk 架构设计

开发者可以直接复用Cosmos-SDK提供的功能模块，也可以基于业务需求实现新的功能模块。每个模块都有自己的存储空间，并可以定义新的消息和相应的处理逻辑。一个消息对应一个链上操作，而一笔交易中可以包含多个消息。从模块化设计的视角来看，一个区块链应用的状态被拆解到多个模块中，每个模块单独管理自己的存储空间，并通过适当的方式与其他模块实现交互。模块的代码组织如下(nft为例)：

```markdown
nft
├── CHANGELOG.md
├── README.md
├── client
│   └── cli // 本模块支持的命令行方法和REST方法
│       ├── tx.go // 从命令行构建本模块支持的交易请求
│       └── tx_test.go 
├── codec.go
├── errors.go
├── event.pb.go
├── expected_keepers.go
├── genesis.go // 导入模块的初始状态以及导出模块的状态
├── genesis.pb.go
├── go.mod
├── go.sum
├── internal
│   └── conv
│       ├── doc.go
│       ├── string.go
│       └── string_test.go
├── keeper 
│   ├── class.go
│   ├── genesis.go
│   ├── grpc_query.go
│   ├── grpc_query_test.go
│   ├── keeper.go // 本模块的子存储空间的读写功能
│   ├── keeper_test.go
│   ├── keys.go
│   ├── msg_server.go // 消息处理
│   ├── msg_server_test.go
│   ├── nft.go
│   ├── nft_batch.go
│   └── nft_batch_test.go
├── keys.go
├── module
│   ├── autocli.go
│   └── module.go  // 实现AppModule接口
├── msgs.go
├── nft.pb.go
├── query.pb.go
├── query.pb.gw.go
├── simulation // 模糊测试相关代码
│   ├── decoder.go
│   ├── decoder_test.go
│   ├── genesis.go
│   ├── genesis_test.go
│   ├── operations.go
│   └── operations_test.go
├── sonar-project.properties
├── testutil
│   ├── app_config.go
│   └── expected_keepers_mocks.go
└── tx.pb.go
```

### keeper

Keepers指的是Cosmos SDK抽象，其作用是**管理对各种模块定义的状态子集的访问(对存储看门的)**。keeper是特定于模块的，即模块定义的状态子集只能由所述模块中定义的keeper访问。如果一个模块需要访问由另一个模块定义的状态子集，则需要将对第二个模块的内部保持器的引用传递给第一个模块。

类型定义如下：

```go
type Keeper struct {
    // External keepers, if any （引用的外部keeper）
	
    // Store key(s) // storeKeys授予对模块管理的多存储的存储的访问权限。它们应始终不暴露于外部模块

    // codec 编解码器，默认protobuf

    // authority  
}
```

### AppModule

baseApp通过AppModule接口来统一管理所有模块，其又拆分为AppModuleBasic、AppModuleGenesis、AppModule。

```go
type AppModule interface {
	AppModuleGenesis
  
	RegisterInvariants(sdk.InvariantRegistry)

  // 注册消息服务以及查询服务以响应客户端特定请求🚩
	RegisterServices(Configurator)
  
	ConsensusVersion() uint64
}

```

AppModuleBasic接口定义了应用程序的基础功能，编码注册，Grpc网关路由注册🚩， 交易以及查询命令获取

```go
type AppModuleBasic interface {
	Name() string
	RegisterLegacyAminoCodec(*codec.LegacyAmino)
	RegisterInterfaces(codectypes.InterfaceRegistry)

	DefaultGenesis(codec.JSONCodec) json.RawMessage
	ValidateGenesis(codec.JSONCodec, client.TxEncodingConfig, json.RawMessage) error

	// 将RESTful API 快速转换为 gRPC 服务
	RegisterGRPCGatewayRoutes(client.Context, *runtime.ServeMux)
	GetTxCmd() *cobra.Command
	GetQueryCmd() *cobra.Command
}
```

AppModuleBasic和keeper以及AppModule 三者关系如下：

```markdown
+------------------------------------------------+
|     						 AppModule    								 |
+------------------------------------------------+
        |      															 |
        |     															 |
        v     														   v
+---------------------+     +---------------------+
|    AppModuleGenesis |     |        keeper     	|
+---------------------+     +---------------------+
 				|      															 
        |     															
        v 
+---------------------+     
|    AppModuleBasic |    
+---------------------+    


```

### moduleManager

Cosmos-SDK的每个功能模块都以AppModule接口的形式暴露给应用。为了方便应用对各个模块的管理，Cosmos-SDK提供了模块管理器，有两种类型的模块管理器，一个用来管理模块实现的AppModuleBasic接口功能；一个用来管理模块在AppModuleBasic之外的AppModule接口功能。应用所需的模块都需要注册到这两个模块管理器中，具体定义如下：

```go
type App struct {
	*baseapp.BaseApp

	ModuleManager     *module.Manager
	...
	basicManager      module.BasicManager
}
type Manager struct {
	Modules                  map[string]interface{} // interface{} is used now to support the legacy AppModule as well as new core appmodule.AppModule.
	OrderInitGenesis         []string
	OrderExportGenesis       []string
	OrderBeginBlockers       []string
	OrderEndBlockers         []string
	OrderPrepareCheckStaters []string
	OrderPrecommiters        []string
	OrderMigrations          []string
}
type BasicManager map[string]AppModuleBasic
```

模块管理器的引入简化了应用对于模块的管理，应用只需要在初始化时设置好模块管理器这个成员变量，随后与模块的交互都通过该模块管理器来进行即可。例如，当应用接收到ABCI对RequestBeginBlock的请求时，只需要调用模块管理器的BeginBlock()方法，该方法会按照指定的顺序依次调用各个模块的BeginBlock()方法。模块管理器架起了顶层应用与底层模块之间沟通的桥梁.

### baseApp

baseapp是Cosmos SDK应用程序的样板实现。它附带了ABCI的实现，以处理与底层共识引擎的连接，同时，BaseApp负责对各模块进行管理，包括响应ABCI请求、转发，消息和查询请求给各模块、管理各模块的初始状态和状态转移。

```go
type BaseApp struct {
	// initialized on creation
	logger            log.Logger
	name              string                      // application name from abci.BlockInfo
	db                dbm.DB                      // common DB backend
	cms               storetypes.CommitMultiStore // Main (uncached) state
	qms               storetypes.MultiStore       // Optional alternative multistore for querying only.
	storeLoader       StoreLoader                 // function to handle store loading, may be overridden with SetStoreLoader()
	grpcQueryRouter   *GRPCQueryRouter            // router for redirecting gRPC query calls
	msgServiceRouter  *MsgServiceRouter           // router for redirecting Msg service messages
	interfaceRegistry codectypes.InterfaceRegistry
	txDecoder         sdk.TxDecoder // unmarshal []byte into sdk.Tx
	txEncoder         sdk.TxEncoder // marshal sdk.Tx into []byte

	mempool            mempool.Mempool            // application side mempool
	anteHandler        sdk.AnteHandler            // ante handler for fee and auth
	postHandler        sdk.PostHandler            // post handler, optional, e.g. for tips
	initChainer        sdk.InitChainer            // initialize state with validators and state blob
	beginBlocker       sdk.BeginBlocker           // logic to run before any txs
	processProposal    sdk.ProcessProposalHandler // the handler which runs on ABCI ProcessProposal
	prepareProposal    sdk.PrepareProposalHandler // the handler which runs on ABCI PrepareProposal
	endBlocker         sdk.EndBlocker             // logic to run after all txs, and to determine valset changes
	prepareCheckStater sdk.PrepareCheckStater     // logic to run during commit using the checkState
	precommiter        sdk.Precommiter            // logic to run during commit using the deliverState
	addrPeerFilter     sdk.PeerFilter             // filter peers by address and port
	idPeerFilter       sdk.PeerFilter             // filter peers by node ID
	fauxMerkleMode     bool                       // if true, IAVL MountStores uses MountStoresDB for simulation speed.

	// manages snapshots, i.e. dumps of app state at certain intervals
	snapshotManager *snapshots.Manager

	// volatile states:
	//
	// checkState is set on InitChain and reset on Commit
	// deliverState is set on InitChain and BeginBlock and set to nil on Commit
	checkState           *state // for CheckTx
	deliverState         *state // for DeliverTx
	processProposalState *state // for ProcessProposal
	prepareProposalState *state // for PrepareProposal

	// an inter-block write-through cache provided to the context during deliverState
	interBlockCache storetypes.MultiStorePersistentCache

	// paramStore is used to query for ABCI consensus parameters from an
	// application parameter store.
	paramStore ParamStore

	// The minimum gas prices a validator is willing to accept for processing a
	// transaction. This is mainly used for DoS and spam prevention.
	minGasPrices sdk.DecCoins

	// initialHeight is the initial height at which we start the baseapp
	initialHeight int64

	// flag for sealing options and parameters to a BaseApp
	sealed bool

	// block height at which to halt the chain and gracefully shutdown
	haltHeight uint64

	// minimum block time (in Unix seconds) at which to halt the chain and gracefully shutdown
	haltTime uint64

	// minRetainBlocks defines the minimum block height offset from the current
	// block being committed, such that all blocks past this offset are pruned
	// from CometBFT. It is used as part of the process of determining the
	// ResponseCommit.RetainHeight value during ABCI Commit. A value of 0 indicates
	// that no blocks should be pruned.
	//
	// Note: CometBFT block pruning is dependant on this parameter in conjunction
	// with the unbonding (safety threshold) period, state pruning and state sync
	// snapshot parameters to determine the correct minimum value of
	// ResponseCommit.RetainHeight.
	minRetainBlocks uint64

	// application's version string
	version string

	// application's protocol version that increments on every upgrade
	// if BaseApp is passed to the upgrade keeper's NewKeeper method.
	appVersion uint64

	// recovery handler for app.runTx method
	runTxRecoveryMiddleware recoveryMiddleware

	// trace set will return full stack traces for errors in ABCI Log field
	trace bool

	// indexEvents defines the set of events in the form {eventType}.{attributeKey},
	// which informs CometBFT what to index. If empty, all events will be indexed.
	indexEvents map[string]struct{}

	// streamingManager for managing instances and configuration of ABCIListener services
	streamingManager storetypes.StreamingManager

	chainID string
}
```

baseApp主要有以下几类成员：

- 基本的日志和存储功能
- ABCI相关
  - txDecoder解码 checktx 以及 diliverTx 接收的交易
  - initChainer、beginBlocker、endBlocker用来响应特定的ABCI请求
  - checkState和deliverState分别对应交易池连接和共识连接方法所依赖的临时应用状态和上下文环境。
- anteHandler: 交易预处理操作，签名验证，gas检查等

## App 和 cometBFT 如何通信

CometBFT通过一个名为ABCI的接口将事务传递给App，app必须实现ABCI。

```markdown
              +---------------------+
              |                     |
              |     Application     |
              |                     |
              +--------+---+--------+
                       ^   |
                       |   | ABCI
                       |   v
              +--------+---+--------+
              |                     |
              |                     |
              |       CometBFT      |
              |                     |
              |                     |
              +---------------------+
```

CometBFT 只处理事务字节。它不知道这些字节的含义。CometBFT 所做的只是确定性地对这些事务字节进行排序。CometBFT 通过 ABCI 将字节传递给应用程序，并期望返回代码通知它事务中包含的消息是否被成功处理。

ABCI有以下几个关键消息：

- CheckTx：当CometBft收到交易时，通过checkTx将传递给应用程序，以检查是否满足了一些基本要求
- DeliverTx：当CometBFT接收到有效块时，块中的每个事务都会通过DeliverTx传递给应用程序以便进行处理。正是在这个阶段，状态转换才会发生。
- BeginBlock/EndBlock:这些消息在每个块的开始和结束时执行，无论块是否包含事务。将会触发自动执行逻辑。

更多细节查看 cometBFT 文档 [ABCI](https://docs.cometbft.com/v0.37/spec/abci/) 

### ABCI

Tendermint core 根据分层设计理念将区块链应用分成3层：p2p通信层、共识协议层以及上层应用层。Tendermint core 提供p2p通信以及共识协议实现，并定义了通用的ABCI与上层应用交互，跟上层应用交互时，Tendermint core 作为客户端发起ABCI请求, 上层应用作为服务端进行响应。交互方式支持GRPC、Socket以及进程内交互(基于go开发的sdk).

「**tendermint客户端和上层APP交互**」

tendermint core 设计了 ABCI client, ABCI Server和Application，跟上层应用交互时，Tendermint core 作为客户端向ABCI Server发起ABCI请求, ABCI Server将请求转发给实现了Application接口的上层应用进行处理。上层应用实现了Application接口和ABCI Server， ABCI提供3种ABCI客户端，localClient和grpcClient以及socketClient， localClient可以直接调用上层应用的ABCI方法，不需要ABCI Server.

**TODO：再具体的需要研究下tendermint底层** 🚩

## 一笔交易如何完成生命周期

### 创建交易

用户可以通过命令行接口来创建交易，如下：

```shell
[appname] tx [command] [args] [flags]
```

### 添加到mempool 

每个接收Tx的完整节点（运行CometBFT）都会向应用层发送一条ABCI消息CheckTx，以检查其有效性，并接收一条ABCI.ResponseCheckTx。如果Tx通过检查，它将保存在节点的Mempool中，等待打包， 如果发现Tx无效，则诚实节点会丢弃它。

### 解码交易

当应用程序从底层共识引擎（例如CometBFT）接收到Tx时，它仍然是其编码的[]字节形式，需要解码才能进行处理。

### 交易基本检查

通过anteHandler 进行签名验证，gas 等检查

### 将交易包含在一个区块中

proposer打包交易进区块，此时的交易是有序的

### 状态改变

达成共识的下一步是执行事务以完全验证它们。所有从正确的提议者那里接收到块提议的完整节点都通过调用ABCI函数BeginBlock、每个事务的DeliverTx和EndBlock来执行事务。虽然每个完整节点都在本地运行所有内容，但由于消息的状态转换是确定性的，并且事务在块建议中是明确排序的，因此该过程会产生一个单一的、明确的结果。

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

DiliverTx完成了大部分的状态转换，顺序的执行块中的每个事务，其大概做了以下事情

- 解码事务

- 通过validateBasic和anteHandler执行相关的校验

- MsgServicerouter会在app中注册相应的handler，所以通过了校验之后会根据runmsgs去路由到相关的handler去处理消息

  ```go
  handler := app.MsgServiceRouter().Handler(msg)
  msgResult, err := handler(app.ctx, msg)
  ```

- msg service: Protobuf-Msg server负责执行Tx中的每个消息，并改变diliver state

### commit 

Validator节点执行交易进行状态改变，然后对块进行签名以进行确认。最后一步是节点提交块和状态更改。

当他们收到足够的验证器投票（2/3+按投票权加权的预提交）时，完整的节点提交到要添加到区块链的新块，并在应用层中完成状态转换。生成一个新的状态根，作为状态转换的merkle证明。应用程序使用从Baseapp继承的Commit ABCI方法；它通过将deliveryState写入应用程序的内部状态来同步所有状态转换。一旦提交了状态更改，checkState就会从最近提交的状态重新开始，deliveryState会重置为nil，以保持一致并反映更改。

此时，Tx的事务生命周期已经结束：节点已经验证了它的有效性，通过执行状态更改来传递它，并提交这些更改。

```GO
tendermint :
ExecCommitBlock -> CommitSync -> app.Application.Commit()

cosmos sdk:
func (app *BaseApp) Commit() abci.ResponseCommit {
  ...
  app.setState(runTxModeCheck, header)
}
```



## 基于cosmos sdk 构建的知名项目

### Defi

- Osmosis: Osmosis 允许用户在不同的区块链之间交换代币和流动性。它支持多种流动性池，例如 AMM (自动做市商)、定向流动性和资本池等。用户可以通过提供流动性和交易费用来赚取收益，并参与 Osmosis 生态系统的治理和发展。

- Binance chain ：Binance Chain 是由 Binance 推出的一个专为去中心化交易所 (DEX) 设计的区块链
- terra ：Terra 是一个稳定币生态系统，它由多个与法定货币挂钩的稳定币组成，例如 TerraUSD (UST) 稳定币和 TerraKRW (KRT) 稳定币。Terra 使用 Cosmos SDK 和 Tendermint 共识机制构建。

### 智能合约平台

- Evmos： Evmos 支持以太坊的智能合约和 ERC20 代币，同时也支持 Cosmos SDK 的模块。

### 跨链

- Gravity Bridge： Gravity Bridge 基于 IBC (Inter-Blockchain Communication) 协议，它允许不同的区块链之间进行安全、可靠、无信任的跨链通信

Layer2

- Celestia: 模块化区块链，cosmos 的二层网络，作为基于cosmos sdk开发的应用链的DA层

### 预言机

- Band Protocol: 是一个去中心化预言机协议，它通过提供可靠、高效、安全的数据源，使得区块链应用可以获取和验证现实世界的数据。它具有去中心化、可扩展性、多链支持和稳定性等特性，为区块链生态系统提供了重要的数据服务

## 与cosmos sdk类似的设计

1. Substrate：Substrate 是一个基于 Rust 语言的区块链框架，由 Parity Technologies 开发。它采用了模块化的设计理念，使得开发者可以根据自己的需求选择和组合不同的模块，从而快速构建自己的区块链应用。Substrate 支持多种共识机制、智能合约语言和网络协议，具有高度的可扩展性和安全性。
2. NEAR Protocol：NEAR Protocol 是一个基于 Rust 语言的公共区块链平台，它采用了类似于 Cosmos SDK 的模块化设计理念。NEAR Protocol 支持多种共识机制和智能合约语言，具有高度的可扩展性和安全性。
