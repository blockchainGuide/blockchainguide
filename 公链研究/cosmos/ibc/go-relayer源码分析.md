## 配置跨链

参考：https://github.com/cosmos/relayer

要通过IBC协议进行跨链，中间组件Relayer必不可少，他充当了消息的转发器，将指定消息路由到对应的链上，要正确使用Relayer需要以下配置：

- 配置想要Relay的两条链
- 导入Relayer在两条链上的Key
- 配置好跨链路径
- 启动跨链

配置跨链路径主要使用已配置的路径在两个已配置的链之间创建客户端、连接和通道，只有当两条链直接可以通信，Relayer才能正确的工作，基于事件查询未传递的IBC消息，并路由到指定链上进行处理，以下主要分析关键的功能

### 创建客户端

对于跨链两端，都必须能够验证对手链的证明和状态，所以对于src链和dst链，src链必须存有dst链轻客户端状态，dst链必须存有src链轻客户状态。

Relayer会通过以下函数创建客户端，分别为对手链创建客户端:

```go
// Create client on src for dst if the client id is unspecified
		clientSrc, err = CreateClient(egCtx, c, dst, srcUpdateHeader, dstUpdateHeader, allowUpdateAfterExpiry, allowUpdateAfterMisbehaviour, override, customClientTrustingPeriod, memo)

	// Create client on dst for src if the client id is unspecified
		clientDst, err = CreateClient(egCtx, dst, c, dstUpdateHeader, srcUpdateHeader, allowUpdateAfterExpiry, allowUpdateAfterMisbehaviour, override, customClientTrustingPeriod, memo)
```

Relayer会先创建createClienetMsg,然后发送到对手链上:

```go
createMsg, err := src.ChainProvider.MsgCreateClient(clientState, dstUpdateHeader.ConsensusState())
res, success, err = src.ChainProvider.SendMessages(ctx, msgs, memo)
```

之后，tendermint会将tx打包，并路由到ibc模块进行相应的消息处理：

```go
func (k Keeper) CreateClient(
	ctx sdk.Context, clientState exported.ClientState, consensusState exported.ConsensusState,
) (string, error) {
	...
	clientID := k.GenerateClientIdentifier(ctx, clientState.ClientType())
	clientStore := k.ClientStore(ctx, clientID)

	if err := clientState.Initialize(ctx, k.cdc, clientStore, consensusState); err != nil {
		return "", err
	}

	if status := k.GetClientStatus(ctx, clientState, clientID); status != exported.Active {
		return "", errorsmod.Wrapf(types.ErrClientNotActive, "cannot create client (%s) with status %s", clientID, status)
	}
	...
	emitCreateClientEvent(ctx, clientID, clientState)

	return clientID, nil
}
```

源连会根据relayer提供的对手链clientState和consensus state 初始化一个客户端，并将其存储在自己链上。 

- Client state是用于描述其他链的状态和特性的数据结构。它包含了其他链的基本信息，如链的ID、状态根、验证人集合等。该状态还包含了一些元数据，如链的名称、版本号等。

- Consensus state是用于描述其他链的共识状态的数据结构。它包含了其他链的最新共识状态，例如其他链的最新区块头和验证人集合。该状态还包含了一些元数据，如最新高度、最新时间戳等

经过这一步，两端都会有各自对手链的轻客户端，从而验证证明和状态。

### 创建连接 

当创建了客户端之后，需要创建两个链之间的连接，连接的创建需要通过4个阶段的握手来实现，类似于TCP3阶段握手，过程如下：

- 阶段1：connOpenInit

​	INIT状态的新连接被创建并存储在启动区块链上

- 阶段2：connOpenTry

​	如果已验证INIT状态的连接是在发起链上创建的，relyer则会创建TRYOPEN状态的新连接并将其存储在相反的区块链上。

- 阶段3：connOpenAck

​	如果src链已验证TRYOPEN状态的连接是在dst链上创建的，则src链上的连接状态将从INIT更新为OPEN	

- 阶段4：connOpenConfirm

​	如果验证了发起链上的连接状态从INIT转换为OPEN，则在相反链上，连接状态将从TRYOPEN更新为OPEN	

中间的状态转换所构造的消息都是由Relayer完成并发送到对应链上的。就以connOpenInit为例：

Relayer组装如下的初始化连接消息，并发送到src链上，通过ibc协议进行处理。

```go
msg := &conntypes.MsgConnectionOpenInit{
		ClientId: info.ClientID,
		Counterparty: conntypes.Counterparty{
			ClientId:     info.CounterpartyClientID,
			ConnectionId: "",
			Prefix:       info.CounterpartyCommitmentPrefix,
		},
		Version:     nil,
		DelayPeriod: defaultDelayPeriod,
		Signer:      signer,
	}
```

```go
func (k Keeper) ConnOpenInit(
	ctx sdk.Context,
	clientID string,
	counterparty types.Counterparty, // counterpartyPrefix, counterpartyClientIdentifier
	version *types.Version,
	delayPeriod uint64,
) (string, error) {
	...
	clientState, found := k.clientKeeper.GetClientState(ctx, clientID)
	if !found {
		return "", errorsmod.Wrapf(clienttypes.ErrClientNotFound, "clientID (%s)", clientID)
	}

	if status := k.clientKeeper.GetClientStatus(ctx, clientState, clientID); status != exported.Active {
		return "", errorsmod.Wrapf(clienttypes.ErrClientNotActive, "client (%s) status is %s", clientID, status)
	}

	connectionID := k.GenerateConnectionIdentifier(ctx)
	if err := k.addConnectionToClient(ctx, clientID, connectionID); err != nil {
		return "", err
	}

	// connection defines chain A's ConnectionEnd
	connection := types.NewConnectionEnd(types.INIT, clientID, counterparty, types.ExportedVersionsToProto(versions), delayPeriod)
	k.SetConnection(ctx, connectionID, connection)
...
}
```

首先获取所有兼容的版本，然后检查传入的版本是否被支持。如果不支持，则返回错误。接下来，它获取客户端状态，并检查该客户端是否存在以及是否处于活动状态。如果客户端不存在或不处于活动状态，则返回错误。

然后，该方法生成一个连接ID，并将其添加到客户端中。接着，它创建一个新的连接，并将其设置为INIT状态，表示连接已初始化但尚未确认。这样连接就绑定到具体的客户端上。且连接状态为Init, 之后Relayer会进行几轮的状态转换，使得两端的连接状态都到达open，这样表示连接的握手成功真正可以用了。 这里就会有个问题，4阶段握手能否优化，以及为什么需要4阶段。 

![connection process](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uR79CCsMGfsk7h-CkIkr2g.png)

----------

### 创建channel

在介绍channel之前还需要理解一个概念port，它表示链上的一个模块，那么既然这个模块要跨链，必须创建跨链通道channel，所以port+channel可以用来表示和某个链的跨链信道，channel则会关联另外一个链的port+channel，可以形象的看做一个桥的两半必须合在一起才能通信，所以在创建channel的时候也会进行4阶段握手，将自己的port+channel和对方的port+channel关联起来，这样才能在通道中发送packet。

另外一个概念Capability, 描述模块所具备的能力，IBC中会通过portA+channel1来声明portA具备跨链能力，同时channel绑定了对手链的portB+channelB以及连接信息，也就是说这个能力抽象了A链和B链的跨链通信，假设A链要和C链进行通信，必须声明portA+channel2(绑定portC+channelC的信息)的能力。这种面向对象能力的设计思想使得应用程序或模块只能在指定的通道上进行跨链交互操作，这种方式可以提高跨链交互的安全性和可靠性。

![channel-handshake](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*C5GhKxzi7a7tCXdMVcjs0Q.png)

------------

到此为止IBC/TAO层的准备工作完毕，链之间创建了轻客户端，连接以及通信信道，用户可以基于IBC/APP层进行跨链通信，通过packet的形式发出，Relayer则会启动事件监听来Relay跨链packet。

## relayer消息处理设计

EventProcessor是Relayer的主要处理器，其包含chainProcessors 和 pathProcessors两个关键对象：

 ChainProcessor 接口负责轮询块并将 IBC 消息事件发送到 PathProcessor。它还负责跟踪打开的通道，而不是将消息发送到关闭通道的 PathProcessors。

PathProcessor是一个处理来自一对链的传入 IBC 消息的进程。 它确定需要中继哪些消息并发送它们。一条链对应一个chainprocessor

### chainProcessor工作任务

```go
func (ccp *CosmosChainProcessor) Run(ctx context.Context, initialBlockHistory uint64) error {
	...
	for {
    	// Infinite retry to get initial latest height
		....
	}
	...
	var eg errgroup.Group
	eg.Go(func() error {
		return ccp.initializeConnectionState(ctx)
	})
	eg.Go(func() error {
		return ccp.initializeChannelState(ctx)
	})
	if err := eg.Wait(); err != nil {
		return err
	}

	ccp.log.Debug("Entering main query loop")

	ticker := time.NewTicker(persistence.minQueryLoopDuration)
	defer ticker.Stop()

	for {
		if err := ccp.queryCycle(ctx, &persistence); err != nil {
			return err
		}
		select {
		case <-ctx.Done():
			return nil
		case <-ticker.C:
			ticker.Reset(persistence.minQueryLoopDuration)
		}
	}
}

```

Chainpropossor 启动后会启动任务让不同的链各自不断尝试获取最新的高度，并从上次查询和目前最新高度的差值中获取所有的IBC相关的事件，并将其转换成对应的消息，分门别类的放到缓存中（IBCMessagesCache），根据高度排序的

```GO
type IBCMessagesCache struct {
	PacketFlow          ChannelPacketMessagesCache
	ConnectionHandshake ConnectionMessagesCache
	ChannelHandshake    ChannelMessagesCache
	ClientICQ           ClientICQMessagesCache
}
```

接着会把链相关的一包数据丢给pathprocessor相关的pathend去处理。

### pathProcessor

pathprocessor一直等待来自chanproposor的消息，并进行处理，其主流程如下：

```go
func (pp *PathProcessor) Run(ctx context.Context, cancel func()) {
	var retryTimer *time.Timer

	pp.flushTicker = time.NewTicker(pp.flushInterval)
	defer pp.flushTicker.Stop()

	for {
		// block until we have any signals to process
		if pp.processAvailableSignals(ctx, cancel) {
			return
		}

		for len(pp.pathEnd1.incomingCacheData) > 0 || len(pp.pathEnd2.incomingCacheData) > 0 || len(pp.retryProcess) > 0 {
			// signals are available, so this will not need to block.
			if pp.processAvailableSignals(ctx, cancel) {
				return
			}
		}

		if !pp.pathEnd1.inSync || !pp.pathEnd2.inSync {
			continue
		}

	....

		// process latest message cache state from both pathEnds
		if err := pp.processLatestMessages(ctx, cancel); err != nil {
			// in case of IBC message send errors, schedule retry after durationErrorRetry
			if retryTimer != nil {
				retryTimer.Stop()
			}
			if ctx.Err() == nil {
				retryTimer = time.AfterFunc(durationErrorRetry, pp.ProcessBacklogIfReady)
			}
		}
	}
}
```

processAvailableSignals会一直接收消息， 直到消息全部接收完毕（A链-B链消息），之后会将两个链上的4类消息进行一类一类的合并，之后将执行4类消息，过程如下：

```GO
func (mp *messageProcessor) processMessages(
	ctx context.Context,
	messages pathEndMessages,
	src, dst *pathEndRuntime,
) error {
	needsClientUpdate, err := mp.shouldUpdateClientNow(ctx, src, dst)
	if err != nil {
		return err
	}

	if err := mp.assembleMsgUpdateClient(ctx, src, dst); err != nil {
		return err
	}

	mp.assembleMessages(ctx, messages, src, dst)

	return mp.trackAndSendMessages(ctx, src, dst, needsClientUpdate)
}

```

首先升级相关客户端，并组装4类消息为cosmos msg,然后发送消息到mempool中，完成消息的传递。

------

Relayer 根据不同的链先通过chainprocessor获取指定区块区间的所有IBC事件，再将其转换成不同类别消息分发给相关的pathprocessor,pathprocessor再将不同类别的消息，分别开4个go协程通过组装函数，发送消息函数来进行一致化处理。

## relay packect

- relayer对 packet 进行基础校验	

- Relayer准备packect commitment(packet proof) 以及对应高度 ，来源src

- 然后准备一个msgrecvPackect 在 dst 上带调用 (附带了src packet 的包的存在性证明)

  ```go
  msg := &chantypes.MsgRecvPacket{
  		Packet:          msgTransfer.Packet(),
  		ProofCommitment: proof.Proof,
  		ProofHeight:     proof.ProofHeight,
  		Signer:          signer,
  	}
  ```

- 准备升级客户端的消息

- 将消息发到对应链的池子中 

## relay ack

- 查询dst 上PacketAcknowledgement ，代表Relayer发送的msgrecvpacket消息已经在dst链上被执行了

- Relayer 准备 MsgAcknowledgement

  ```go
  msg := &chantypes.MsgAcknowledgement{
  		Packet:          msgRecvPacket.Packet(),
  		Acknowledgement: msgRecvPacket.Ack,
  		ProofAcked:      proof.Proof,
  		ProofHeight:     proof.ProofHeight,
  		Signer:          signer,
  	}
  
  ```

- Relayer会升级在src链上的dst轻客户端

- Relayer将这两类消息发送到src

## relayer付费

1. 当监控到来源于src的packet时候，需要查询packect commitment ,这部分需要付费
2. 升级在dst链上的src轻客户端需要付费
3. 然后需要构造msgrecvpacket 在dst链上发送也需要付费
4. 升级在src链上的dst轻客户端需要付费
5. 构造MsgAcknowledgement，发送src链，也需要付费

## relayer的激励机制

Reference: 

- https://github.com/cosmos/ibc/issues/578
- https://medium.com/imperator-guide/ics29-how-do-relayer-fees-work-with-ibc-in-the-cosmos-ecosystem-e89ae6cf4530

## 参考

- [ics protocol](https://github.com/cosmos/ibc)
- [ibc-go](https://github.com/cosmos/ibc-go)
- [relayer-go](https://github.com/cosmos/relayer)
- [how ibc works between blockchains](https://medium.com/@datachain/how-cosmoss-ibc-works-to-achieve-interoperability-between-blockchains-d3ee052fc8c3)
- [hermes guide](https://hermes.informal.systems/)
- 面向对象能力设计思想
- [good-articles](https://polymerlabs.medium.com/blockchain-bridges-101-a-guide-to-inter-blockchain-communication-ibc-3efc092770b1)