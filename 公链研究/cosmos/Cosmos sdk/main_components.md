## 通过 dilivertx 传送给其他节点的APP

APP处理事务如下：

- Decode `transactions` received from the CometBFT consensus engine
- Extract `messages` from `transactions` and do basic sanity checks. 
- Route each message to the appropriate module 
- Commit state changes.

## baseapp

baseapp是Cosmos SDK应用程序的样板实现。它**附带了ABCI**的实现，以处理与底层共识引擎的连接。通常，Cosmos SDK应用程序通过将baseapp嵌入app.go来扩展它。

以下是Cosmos SDK演示应用程序simapp的一个示例：

```go
type SimApp struct {
	*baseapp.BaseApp
	legacyAmino       *codec.LegacyAmino
	appCodec          codec.Codec
	txConfig          client.TxConfig
	interfaceRegistry types.InterfaceRegistry

	// keys to access the substores
	keys    map[string]*storetypes.KVStoreKey
	tkeys   map[string]*storetypes.TransientStoreKey
	memKeys map[string]*storetypes.MemoryStoreKey

	// keepers
	AccountKeeper         authkeeper.AccountKeeper
	BankKeeper            bankkeeper.Keeper
	...

	// the module manager
	ModuleManager *module.Manager

	// simulation manager
	sm *module.SimulationManager

	// module configurator
	configurator module.Configurator
}
```

baseapp的目标是在存储和可扩展状态机之间提供一个安全的接口，同时尽可能少地定义状态机（忠于ABCI）



## Multistore

Cosmos SDK为持久化状态提供了一个Multistore。Multistore允许开发人员声明任意数量的KVStores。这些KVStores只接受[]字节类型作为值，因此任何自定义结构在存储之前都需要使用编解码器进行编组。

Multistore抽象用于将状态划分为不同的分区，每个分区由自己的模块管理。

## Modules

Cosmos SDK应用程序是通过聚合一组**可互操作的模块**来构建的。每个模块都定义了**状态的一个子集**，并包含自己的消息/事务处理器，而Cosmos SDK负责将每个消息路由到各自的模块。

以下是当事务在有效块中接收时，每个完整节点的应用程序如何处理事务的简化视图：

```markdown
                                      +
                                      |
                                      |  Transaction relayed from the full-node's
                                      |  CometBFT engine to the node's application
                                      |  via DeliverTx
                                      |
                                      |
                +---------------------v--------------------------+
                |                 APPLICATION                    |
                |                                                |
                |     Using baseapp's methods: Decode the Tx,    |
                |     extract and route the message(s)           |
                |                                                |
                +---------------------+--------------------------+
                                      |
                                      |
                                      |
                                      +---------------------------+
                                                                  |
                                                                  |
                                                                  |  Message routed to
                                                                  |  the correct module
                                                                  |  to be processed
                                                                  |
                                                                  |
+----------------+  +---------------+  +----------------+  +------v----------+
|                |  |               |  |                |  |                 |
|  AUTH MODULE   |  |  BANK MODULE  |  | STAKING MODULE |  |   GOV MODULE    |
|                |  |               |  |                |  |                 |
|                |  |               |  |                |  | Handles message,|
|                |  |               |  |                |  | Updates state   |
|                |  |               |  |                |  |                 |
+----------------+  +---------------+  +----------------+  +------+----------+
                                                                  |
                                                                  |
                                                                  |
                                                                  |
                                       +--------------------------+
                                       |
                                       | Return result to CometBFT
                                       | (0=Ok, 1=Err)
                                       v
```

每个模块都可以看作是一个小型状态机。开发人员需要定义模块处理的状态的子集，以及修改状态的自定义消息类型（注意：消息是由baseapp从事务中提取的）。通常，每个模块在多存储中声明自己的KVStore，以持久化其定义的状态的子集。大多数开发人员在构建自己的模块时都需要访问其他第三方模块。鉴于Cosmos SDK是一个开放的框架，一些模块可能是恶意的，这意味着需要安全原则来解释模块间的交互。这些原则是基于对象能力的。在实践中，这意味着不是让每个模块为其他模块保留访问控制列表，而是每个模块实现称为keeper的特殊对象，这些对象可以传递给其他模块以授予预定义的一组功能。