> 死磕hyperledger fabric源码|交易广播
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![40fe3d4a84cc46e22caf5e06071c3aa7](https://tva1.sinaimg.cn/large/008eGmZEgy1gmz2lvzbprj31ao0t6n61.jpg)

## 前言

`Hyperledger Fabric`提供了`Broadcast(srv ab.AtomicBroadcast_BroadcastServer)`交易广播服务接口，接收客户端提交的签名交易消息请求，交由共识组件链对象对交易进行排序与执行通道管理，按照交易出块规则切割打包，构造新区块并提交账本。同时，通过`Deliver()`区块分发服务接口，将区块数据发送给通道组织内发起请求的`Leader`主节点，再基于`Gossip`消息协议广播到组织内的其他节点上，从而实现广播交易消息的目的。 

## Broadcast服务消息处理

`Orderer`节点启动时已经在本地的`gRPC`服务器上注册了`Orderer`排序服务器，并创建了`Broadcast`服务处理句柄。当客户端调用`Broadcast()`服务接口发起服务请求时，`Orderer`排序服务器会调用`Broadcast()→s.bh.Handle()`方法处理请求，流程如下：

```go
func (s *server) Broadcast(srv ab.AtomicBroadcast_BroadcastServer) error {
...
	return s.bh.Handle(&broadcastMsgTracer{
	...
	})
}
```

```go
func (bh *handlerImpl) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {
  ...
}
```

主要就是这个`Handle`的处理，分析如下：

①：等待接收处理消息

```go
msg, err := srv.Recv()
```

②：解析获取通道头部chdr、配置交易消息标志位isConfig、通道链支持对象（通道消息处理器）

```go
chdr, isConfig, processor, err := bh.sm.BroadcastChannelSupport(msg)
```

③：检查共识组件链对象是否准备好接收新的交易消息

```go
if err = processor.WaitReady(); err != nil {}
```

④：分类处理消息

**处理普通消息**

4.1 解析获取通道的最新配置序号

```go
configSeq, err := processor.ProcessNormalMsg(msg)
```

```go
/orderer/common/msgprocessor/standardchannel.go
func (s *StandardChannel) ProcessNormalMsg(env *cb.Envelope) (configSeq uint64, err error) {
	configSeq = s.support.Sequence()
	err = s.filters.Apply(env)
	return
}
```

configSeq是最新配置序号，默认初始值为0，新建应用通道后该配置序号自增为1，通过比较该序号就能判断当前通道配置版本是否发生了更新，从而确定当前交易消息是否需要重新过滤与重新排序。

接着就是使用自带的默认通道消息过滤器过滤消息，有以下过滤条件：

- 验证不能为空
- 拒绝过期的签名者身份证书
- 消息最大字节数过滤器（98MB）
- 消息签名验证过滤器

4.2 构造新的普通交易消息并发送到共识组件链对象请求处理

```GO
err = processor.Order(msg, configSeq) 
```

这里我们只关注`kafka`的共识组件处理。

首先序列化消息，然后将该消息发送到`Kafka`集群的指定分区上请求排序，再转发给`Kafka`共识组件链对象请求打包出块。

```go
/orderer/consensus/kafka/chain.go
func (chain *chainImpl) order(env *cb.Envelope, configSeq uint64, originalOffset int64) error {
	marshaledEnv, err := utils.Marshal(env)
	if err != nil {
		return fmt.Errorf("cannot enqueue, unable to marshal envelope because = %s", err)
	}
	if !chain.enqueue(newNormalMessage(marshaledEnv, configSeq, originalOffset)) {
		return fmt.Errorf("cannot enqueue")
	}
	return nil
}
```

我们来看看enqueue方法是如何做的：

```go
func (chain *chainImpl) enqueue(kafkaMsg *ab.KafkaMessage) bool {
	logger.Debugf("[channel: %s] Enqueueing envelope...", chain.ChainID())
	select {
	case <-chain.startChan: // // 共识组件在启动阶段启动完成
		select {
		case <-chain.haltChan: //  已经关闭chain.startChan通道
		...
			}
			//// 创建Kafka生产者消息
			message := newProducerMessage(chain.channel, payload)
			//// 发送消息到Kafka集群请求排序
			if _, _, err = chain.producer.SendMessage(message); err != nil {
			...
	}
}
```

**处理通道配置交易消息**

4.3  获取配置交易消息与通道的最新配置序号

```go
config, configSeq, err := processor.ProcessConfigUpdateMsg(msg)
```

代码位置：/orderer/common/msgprocessor/systemchannel.go/ProcessConfigUpdateMsg,大概做了以下事情：

- *获取消息中的通道ID*
- *检查消息中的通道ID与当前通道ID是否一致,一致的话交由标准通道处理器处理*
- *创建新应用通道的通道配置实体Bundle结构对象*
- *构造新的通道配置更新交易消息（ConfigEnvelope类型），注意将该消息的通道配置序号更新为1*
- *创建内层的通道配置交易消息（CONFIG类型）*
- *创建外层的配置交易消息（ORDERER_TRANSACTION类型）*
- *应用系统通道的消息过滤器*
- *返回新的通道配置交易消息与当前系统通道的配置序号*

```go
func (s *SystemChannel) ProcessConfigUpdateMsg(envConfigUpdate *cb.Envelope) (config *cb.Envelope, configSeq uint64, err error) {
	channelID, err := utils.ChannelID(envConfigUpdate) // 获取消息中的通道ID
	...
	//检查消息中的通道ID与当前通道ID是否一致
	if channelID == s.support.ChainID() {
		//// 交由标准通道处理器处理
		return s.StandardChannel.ProcessConfigUpdateMsg(envConfigUpdate)
	}
	...
	// 创建新的应用通道，其通道配置序号默认初始化为0
	// 创建新应用通道的通道配置实体Bundle结构对象
	bundle, err := s.templator.NewChannelConfig(envConfigUpdate)
	...
	//构造新的通道配置更新交易消息（ConfigEnvelope类型），注意将该消息的通道配置序号更新为1
	newChannelConfigEnv, err := bundle.ConfigtxValidator().ProposeConfigUpdate(envConfigUpdate)
	...
	//创建内层的通道配置交易消息（CONFIG类型）
	newChannelEnvConfig, err := utils.CreateSignedEnvelope(cb.HeaderType_CONFIG, channelID, s.support.Signer(), newChannelConfigEnv, msgVersion, epoch)
	...
	//创建外层的配置交易消息（ORDERER_TRANSACTION类型）
	wrappedOrdererTransaction, err := utils.CreateSignedEnvelope(cb.HeaderType_ORDERER_TRANSACTION, s.support.ChainID(), s.support.Signer(), newChannelEnvConfig, msgVersion, epoch)
	...
	// 应用系统通道的消息过滤器
	err = s.StandardChannel.filters.Apply(wrappedOrdererTransaction)
	...
	//返回新的通道配置交易消息与当前系统通道的配置序号
	return wrappedOrdererTransaction, s.support.Sequence(), nil
```

4.4 构造新的配置交易消息发送到共识组件链对象请求排序

```go
err = processor.Configure(config, configSeq)
```

这里我们依旧只是考虑`kafka`共识组件，`processor.Configure()`方法实际上是调用`chainImpl.configure()`方法，同样构造`Kafka`常规消息（`KafkaMessageRegular`类型）。其中，`Class`消息类别属于`KafkaMessageRegular_CONFIG`类型，包含了通道配置交易消息、 通道配置序号`configSeq`与初始消息偏移量`originalOffset（0）`。接着，调用`chain.enqueue()`方法，将其发送到`Kafka`集群上指定主题（`chainID`）和分区号（0）的分区上，同时，由`Kafka`共识组件链对象分区消费者`channelConsumer`获取该消息，再交由给`Kafka`共识组件链对象请求打包出块。

⑤：发送成功处理状态响应消息

```go
err = srv.Send(&ab.BroadcastResponse{Status: cb.Status_SUCCESS})
```

整个流程图如下：

![image-20210125112957202](https://tva1.sinaimg.cn/large/008eGmZEgy1gmzs6ruy53j31860u0doq.jpg)

------

## 参考 

> https://github.com/blockchainGuide/ (文章图片代码资料)
>
> 微信公众号：区块链技术栈

