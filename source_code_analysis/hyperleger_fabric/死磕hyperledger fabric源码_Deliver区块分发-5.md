> 死磕hyperledger fabric源码|Deliver区块分发
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![00a1ba6f145942b3fdb1a7b63964967e](https://tva1.sinaimg.cn/large/008eGmZEgy1gn0uq99cf2j31hc0u0k1k.jpg)



## 概述

`Orderer`排序服务器提供了区块分发服务接口，接收客户端提交的区块请求消息（`Envelope`类型，通道头部类型是`DELIVER_SEEK_INFO`、`CONFIG_UPDATE`等），根据该消息封装的区块搜索信息对象（`SeekInfo`类型），包括查找最旧区块SeekOldest类型、查找最新区块`SeekNewest`类型、查找指定位置区块`SeekSpecified`类型等，构造对应请求范围的范围查询结果迭代器，读取`Orderer`节点指定通道账本上的区块数据，同时，建立消息处理循环，基于该结果迭代器依次读取请求的区块数据结果，**发送给组织的Leader主节点等请求节点。**

`Orderer`节点启动时在本地`gRPC`服务器上注册了`Orderer`排序服务器，并创建了Deliver服务处理句柄。当客户端发起`Deliver`服务请求时，`Orderer`排序服务器就调用`Deliver()`方法处理消息请求。



## Diliver消息服务处理

入口在`orderer/common/server/server.go/Deliver()`方法中：

```go
func (s *server) Deliver(srv ab.AtomicBroadcast_DeliverServer) error {
	...
	policyChecker := func(env *cb.Envelope, channelID string) error { // 定义策略检查器
		chain, ok := s.GetChain(channelID) // 获取指定通道的链支持对象
		if !ok {
			return errors.Errorf("channel %s not found", channelID)
		}
		// 创建消息过滤器
		sf := msgprocessor.NewSigFilter(policies.ChannelReaders, chain)
		return sf.Apply(env) // 过滤消息
	}
	server := &deliverMsgTracer{
		DeliverSupport: &deliverHandlerSupport{AtomicBroadcast_DeliverServer: srv},
		msgTracer: msgTracer{
			debug:    s.debug,
			function: "Deliver",
		},
	}
	// Deliver服务消息处理
	return s.dh.Handle(deliver.NewDeliverServer(server, policyChecker, s.sendProducer(srv)))
}
```

大概做了以下几件事：

- 定义策略检查器：用于检查接收的区块请求消息必须满足指定通道上的访问控制权限策略的要求
- 获取指定通道的链支持对象
- 创建消息过滤器，过滤消息
- Deliver服务消息处理区块请求

我们来看是如何处理的，进入到`s.dh.Handle`:  

> /common/deliver/deliver.go/Handle

```GO
func (ds *deliverHandler) Handle(srv *DeliverServer) error {
...
	// 等待消息请求并进行处理
	for {
		...
		envelope, err := srv.Recv() // 等待接收客户端发送的区块消息请求
	...
		// 从Orderer节点本地指定通道的区块账本中获取指定区块，并向客户端发送请求
		if err := ds.deliverBlocks(srv, envelope); err != nil {
			return err
		}
...
	}
}
```

不言而喻，直接进入到`deliverBlocks`,这部分的内容是最核心的，逐步分析如下：

①：*解析PayLoad,检查header和ChannelHeader的合法性*

```go
payload, err := utils.UnmarshalPayload(envelope.Payload) // 解析消息负载
...
if payload.Header == nil {}
// 解析通道头部
	chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
err = ds.validateChannelHeader(srv, chdr) // 验证通道头部合法性
```

②：*从chains字典中获取指定通道（chainID）的链支持对象chain，并检查该对象是否存在错误信息* 

```go
chain, ok := ds.sm.GetChain(chdr.ChannelId) // 获取指定通道的链支持对象
```

③：*创建访问控制对象,并*检查消息签名是否符合指定的通道读权限策略**

```go
accessControl, err := newSessionAC(chain, envelope, srv.PolicyChecker, chdr.ChannelId, crypto.ExpiresAt)
...
err := accessControl.evaluate()
```

④：*解析区块搜索信息SeekInfo结构对象*

```GO
seekInfo := &ab.SeekInfo{}
if err = proto.Unmarshal(payload.Data, seekInfo); err != nil {}
```

⑤：*检查起始位置与结束位置的合法性*

```go
if seekInfo.Start == nil || seekInfo.Stop == nil {}
```

⑥：*创建区块账本迭代器并获取起始区块号，同时设置起始位置*

```GO
cursor, number := chain.Reader().Iterator(seekInfo.Start)
```

`Iterator`根据`startPosition.Type`起始位置对象的类型计算起始区块号`startingBlockNumbe`,类型如下：

- SeekPosition_Oldest：搜索最旧的区块，将起始区块号`startingBlockNumber`设置为 0； 

- SeekPosition_Newest：搜索最新的区块，将起始区块号`startingBlockNumber`设置为当前通道账本的最新区块号`info.Height-1`，即账本高度减1； 

- SeekPosition_Specified：搜索指定位置的区块，将起始区块号`startingBlockNumber`设置为指定起始位置的区块号`start.Specified.Number`。

`Iterator` 方法的大致功能如下： `common/ledger/blockledger/file/impl.go/Iterator`

```go
func (fl *FileLedger) Iterator(startPosition *ab.SeekPosition) (blockledger.Iterator, uint64) {
	var startingBlockNumber uint64
	switch start := startPosition.Type.(type) { // 分析起始位置类型
	case *ab.SeekPosition_Oldest: // 搜索最旧区块，区块号为0
		startingBlockNumber = 0
	case *ab.SeekPosition_Newest: // 搜索最新区块
		info, err := fl.blockStore.GetBlockchainInfo() // 获取区块链信息
		if err != nil {
			logger.Panic(err)
		}
		newestBlockNumber := info.Height - 1 // 最新区块号
		startingBlockNumber = newestBlockNumber
	case *ab.SeekPosition_Specified: // 搜索指定位置区块
		startingBlockNumber = start.Specified.Number
		height := fl.Height()
		if startingBlockNumber > height { // 若超过高度，则报错
			return &blockledger.NotFoundErrorIterator{}, 0
		}
	default:
		return &blockledger.NotFoundErrorIterator{}, 0
	}
	// 构造区块迭代器
	iterator, err := fl.blockStore.RetrieveBlocks(startingBlockNumber)
	if err != nil {
		return &blockledger.NotFoundErrorIterator{}, 0
	}
	// 构造账本区块迭代器
	return &fileLedgerIterator{ledger: fl, blockNumber: startingBlockNumber, commonIterator: iterator}, startingBlockNumber
}
```

⑦：循环读取区块数据，从本地区块账本中获取指定区块号范围内的区块数据，并依次顺序发送给请求客户端

7.1 未找到数据返回

```go
if seekInfo.Behavior == ab.SeekInfo_FAIL_IF_NOT_READY {
			if number > chain.Reader().Height()-1 {
				return sendStatusReply(srv, cb.Status_NOT_FOUND)
			}
		}
```

7.2 获取下一个数据

```go
block, status := nextBlock(cursor, erroredChan) // 从本地账本获取下一个区块
if status != cb.Status_SUCCESS {...}
```

7.3  再次检查是否满足访问控制策略要求

```GO
if err := accessControl.evaluate(); err != nil {}
```

7.4  发送区块数据

```go
if err := sendBlockReply(srv, block); err != nil { }
```

7.5 *循环结束，发送成功状态*

```go
if err := sendStatusReply(srv, cb.Status_SUCCESS);
```

## Deliver服务客户端

以`Leader`主节点为例，分析`Deliver`服务客户端从`Orderer`节点请求获取区块的流程。

### 初始化Deliver服务实例

入口：`gossip/service/gossip_service.go/InitializeChannel`

```go
func (g *gossipServiceImpl) InitializeChannel(chainID string, endpoints []string, support Support) {
	...
	g.chains[chainID] = state.NewGossipStateProvider(chainID, servicesAdapter, coordinator)
	if g.deliveryService[chainID] == nil { // 检查是否已经存在Deliver服务实例
		var err error
		g.deliveryService[chainID], err = g.deliveryFactory.Service(g, endpoints, g.mcs) // 检查是否已经存在Deliver服务实例
		...
		// peer.gossip.useLeaderElection与peer.gossip.orgLeader是互斥的两个配置参数，
		// 如果将两个都设置为true且没有被定义，则会引起Peer节点错误
		// 启用Leader主节点动态选举机制
		leaderElection := viper.GetBool("peer.gossip.useLeaderElection")
		// 静态设置为组织Leader主节点
		isStaticOrgLeader := viper.GetBool("peer.gossip.orgLeader")
		...
		if leaderElection { // 启用了动态Leader主节点选举机制
			logger.Debug("Delivery uses dynamic leader election mechanism, channel", chainID)
			g.leaderElection[chainID] = g.newLeaderElectionComponent(chainID, g.onStatusChangeFactory(chainID, support.Committer))
		} else if isStaticOrgLeader {
			// 若静态指定了Leader主节点，则连接 Orderer节点请求区块数据
			// 启动指定通道上的Deliver服务实例请求获取区块数据
			g.deliveryService[chainID].StartDeliverForChannel(chainID, support.Committer, func() {})
		} ....
}
```

首先检查是否已经存在`Deliver`实例，然后根据`Leader`主节点动态选举机制还是静态指定了`Leader`主节点分别进入不同的分支，如果是静态指定了`Leader`主节点，则连接 `Orderer`节点请求区块数据,启动指定通道上的`Deliver`服务实例请求获取区块数据。接下来关注启动`Deliver`服务实例。

### 启动Deliver服务实例

主要做了以下事：

①：*获取绑定指定通道的区块提供者*

```go
if _, exist := d.blockProviders[chainID];
```

②：*不存在区块提供者*

```go
client := d.newClient(chainID, ledgerInfo)
```

```go
func (d *deliverServiceImpl) newClient(chainID string, ledgerInfoProvider blocksprovider.LedgerInfo) *broadcastClient {
	requester := &blocksRequester{ //定义区块请求者blocksRequester结构对象
		tls:     comm.TLSEnabled(),
		chainID: chainID,
	}
	//定义broadcastSetup()方法
	broadcastSetup := func(bd blocksprovider.BlocksDeliverer) error {
		return requester.RequestBlocks(ledgerInfoProvider) // 请求区块数据
	}
	...
	//创建connProducer对象
	connProd := comm.NewConnectionProducer(d.conf.ConnFactory(chainID), d.conf.Endpoints)
	//// 创建broadcastClient客户端
	bClient := NewBroadcastClient(connProd, d.conf.ABCFactory, broadcastSetup, backoffPolicy)
	requester.client = bClient // 设置到区块请求者对象的客户端
	return bClient
}
```

2.1 创建Deliver服务实例上的 broadcastClient客户端

```go
client := d.newClient(chainID, ledgerInfo)
```

2.2 创建指定通道关联的区块提供者

```go
d.blockProviders[chainID] = blocksprovider.NewBlocksProvider(chainID, client, d.conf.Gossip, d.conf.CryptoSvc)
```

2.3 启动goroutine开始从Orderer节点请求获取区块，并发送到组织内其他Peer节点

```go
go func() {
			d.blockProviders[chainID].DeliverBlocks() // 请求获取区块数据
			finalizer()
		}()
```

接下来就是调用区块提供者对象的`DeliverBlocks()`方法，向`Orderer`节点发送消息请求的区块数据。

### 请求获取区块数据

入口在：`core/deliverservice/blocksprovider/blocksprovider.go/DeliverBlocks()`,具体分析如下：

①：接收消息

```go
msg, err := b.client.Recv() 
```

② ：根据消息类型进行处理

大致有以下几种消息类型：

- DeliverResponse_Status:用于描述`Deliver`服务请求执行状态。
- DeliverResponse_Block：包含请求获取的区块数据。

2.1 DeliverResponse_Status分支

如果DeliverBlocks()方法接收到Status_SUCCESS状态，则说明本次区块请求处理成功，表示已经接收完毕区块请求消息指定范围内的区块数据。除此以外的其他状态消息都是非成功的执行状态消息，包括Status_BAD_REQUEST、Status_FORBIDDEN等

```go
if t.Status == common.Status_SUCCESS {}
if t.Status == common.Status_BAD_REQUEST || t.Status == common.Status_FORBIDDEN {}
if t.Status == common.Status_BAD_REQUEST {
				b.client.Disconnect(false)
			} else {
				b.client.Disconnect(true)
			}
```

2.2 DeliverResponse_Block分支

2.2.1 *获取区块号*

```go
seqNum := t.Block.Header.Number
```

2.2.2*获取经过序列化的区块字节数组*

```GO
marshaledBlock, err := proto.Marshal(t.Block)
```

2.2.3*验证区块*

```GO
err := b.mcs.VerifyBlock(gossipcommon.ChainID(b.chainID), seqNum, marshaledBlock);
```

2.2.4*获取通道Peer节点数量*

```go
numberOfPeers := len(b.gossip.PeersOfChannel(gossipcommon.ChainID(b.chainID)))
```

2.2.5*创建消息负载和Gossip消息*

```go
payload := createPayload(seqNum, marshaledBlock) 
gossipMsg := createGossipMsg(b.chainID, payload)
```

2.2.6*添加消息负载到本地消息负载缓冲区，等待提交账本*

```go
err := b.gossip.AddPayload(b.chainID, payload)
```

2.2.7*通过Gossip消息协议发送区块消息到组织内的其他节点*

基于`Gossip`消息协议将`DataMsg`类型数据消息（只含有区块数据）分发到组织内的其他`Peer`节点上，并保存到该节点的消息存储器上。

```GO
b.gossip.Gossip(gossipMsg)
```

## 参考

> https://github.com/blockchainGuide/ (文章图片代码资料)
>
> 微信公众号：区块链技术栈





