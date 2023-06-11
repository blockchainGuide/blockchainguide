## app组成

- 引用baseapp,它实现了ABCI接口和路由逻辑
- store keys列表：持久化其状态数据
- keeper列表:处理模块存储的读写
- 对appCodec的引用:应用程序的appCodec用于序列化和反序列化数据结构以存储它们，因为存储只能持久化[]字节,默认protobuf
- 对模块管理器和基本模块管理器的引用：模块管理器是一个包含应用程序模块列表的对象。它方便了与这些模块相关的操作，例如注册它们的Msg服务和gRPC查询服务，或者为InitChainer、BeginBlocker和EndBlocker等各种函数设置模块之间的执行顺序

### initChainer

InitChainer是一个从genesis文件初始化应用程序状态的函数（即genesis帐户的令牌余额）。当应用程序从CometBFT引擎接收到InitChain消息时会调用它，当节点在appBlockHeight==0（即在genesis上）启动时会发生这种情况。应用程序必须通过SetInitChainer方法在其构造函数中设置InitChainer。

通常，InitChainer主要**由每个应用程序模块的InitGenesis**函数组成。这是通过调用模块管理器的InitGenesis函数来完成的，该函数反过来调用它所包含的每个模块的InitGenesis函数。请注意，必须使用模块管理器的SetOrderInitGenesis方法在模块管理器中设置调用模块InitGeness函数的顺序。这是在应用程序的构造函数中完成的，并且必须在SetInitChainer之前调用SetOrderInitGenesis。

## BeginBlocker and EndBlocker

Cosmos SDK为开发人员提供了将代码自动执行作为应用程序一部分的可能性。这是通过两个称为BeginBlocker和EndBlocker的函数实现的。当应用程序从CometBFT引擎接收到BeginBlock和EndBlock消息时，会调用它们，这分别发生在每个块的开头和结尾。应用程序必须通过SetBeginBlocker和SetEndBlocker方法在其构造函数中设置BeginBlockers和EndBlocker。

通常，BeginBlocker和EndBlocker函数主要由应用程序的每个模块的BeginBlock和EndBlock函数组成。这是通过调用模块管理器的BeginBlock和EndBlock函数来完成的，模块管理器依次调用它所包含的每个模块的BeginBlock和EndBlock功能。请注意，必须分别使用SetOrderBeginBlockers和SetOrderEndBlockers方法在模块管理器中设置模块的BeginBlock和EndBlock函数的调用顺序。这是通过应用程序构造函数中的模块管理器完成的，并且必须在SetBeginBlocker和SetEndBlocker函数之前调用SetOrderBeginBlockers和SetOrderEndBlockers方法。

顺便说一句，重要的是要记住，特定于应用程序的区块链是确定性的。开发者必须小心不要在BeginBlocker或EndBlocker中引入非决定论，也**必须小心不要使它们在计算上过于昂贵**，因为天然气**不会限制BeginBlock和EndBlocker的成本**

## Register Codec

EncodingConfig结构是app.go文件的最后一个重要部分。这个结构的目标是定义将在整个应用程序中使用的编解码器。

```GO
type EncodingConfig struct {
	InterfaceRegistry types.InterfaceRegistry
	Codec             codec.Codec
	TxConfig          client.TxConfig
	Amino             *codec.LegacyAmino
}
```

## modules

模块是Cosmos SDK应用程序的核心和灵魂。它们可以被视为嵌套在状态机中的状态机。当事务通过ABCI从底层CometBFT引擎中继到应用程序时，它会由baseapp路由到适当的模块以便进行处理。这种范式使开发人员能够轻松地构建复杂的状态机，因为他们需要的大多数模块通常已经存在。对于开发人员来说，构建Cosmos SDK应用程序所涉及的大部分工作都围绕着构建应用程序所需的自定义模块展开，这些模块还不存在，并将它们与已经存在的模块集成到一个连贯的应用程序中。在应用程序目录中，标准做法是将模块存储在x/文件夹中（不要与Cosmos SDK的x/文件夹混淆，后者包含已构建的模块）

### Application Module Interface

模块必须实现Cosmos SDK、AppModuleBasic和AppModule中定义的接口。前者实现模块的基本非依赖元素，如编解码器，而后者处理大部分模块方法（包括需要引用其他模块的保持器的方法）。按照惯例，AppModule和AppModuleBasic类型都是在一个名为module.go的文件中定义的。

AppModule在模块上公开了一组有用的方法，这些方法有助于将模块组合成一个连贯的应用程序。这些方法是从模块管理器调用的，模块管理器管理应用程序的模块集合。

## Msg service

每个应用程序模块定义两个Protobuf服务：**一个Msg服务用于处理消息**，另一个**gRPC查询服务用于处理查询**。如果我们将模块视为一个状态机，那么Msg服务就是一组状态转换RPC方法。每个Protobuf-Msg服务方法都与Protobuf请求类型1:1相关，该请求类型必须实现sdk.Msg接口。请注意，sdk.Msgs捆绑在事务中，每个事务包含一个或多个消息。

当完整节点接收到有效的事务块时，CometBFT通过DeliverTx将每个事务块中继到应用程序。然后，应用程序处理事务：

在接收到事务后，应用程序首先从[]字节对其进行解组。

然后，在提取交易中包含的Msg之前，它会验证交易的一些内容，如费用支付和签名。

消息使用Protobuf-Anys进行编码。通过分析每个Any的type_url，baseapp的msgServiceRouter将sdk.Msg路由到相应模块的Msg服务。

如果消息处理成功，则状态会更新。

模块开发人员在构建自己的模块时会创建自定义的Msg服务。一般做法是在tx.proto文件中定义Msg-Protobuf服务。例如，x/bank模块定义了一个服务，该服务具有两种传输令牌的方法

```go
// Msg defines the bank Msg service.
service Msg {
  option (cosmos.msg.v1.service) = true;

  // Send defines a method for sending coins from one account to another account.
  rpc Send(MsgSend) returns (MsgSendResponse);

  // MultiSend defines a method for sending coins from some accounts to other accounts.
  rpc MultiSend(MsgMultiSend) returns (MsgMultiSendResponse)
}
```

## keeper

Keeper是模块存储的看门人。要在模块的存储中读取或写入，必须通过其keeper的方法之一。这是通过Cosmos SDK的对象能力模型来确保的。只有持有存储区密钥的对象才能访问它，并且只有模块的管理员才能持有模块存储区的密钥。

keeper通常在一个名为keeper.go的文件中定义。它包含keeper的类型定义和方法。