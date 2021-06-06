> 死磕hyperledger fabric源码|kafka共识排序
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![d1e794177969e09552b173b7d6eaea19](https://tva1.sinaimg.cn/large/008eGmZEgy1gmzuqmak04j30zk0m8weu.jpg)

## 概述

Orderer共识组件提供HandleChain()方法创建通道绑定的共识组件链对象（`consensus.Chain`接口），包括`Solo`（`solo.chain`类型）、`Kafka`（`kafka.chainImpl`类型）等类型，属于通道共识组件的重要实现模块，并设置到链支持对象的`cs.Chain`字段。共识组件链对象提供Orderer共识排序服务，负责关联通道上交易排序、打包出块、提交账本、通道管理等工作，目前采用`Golang`通道或`Kafka`集群作为共识排序后端，**接收来自`Broadcast`服务过滤转发的交易消息**并进行排序。  

## kafka共识排序服务

### orderer服务集群

`Orderer`节点采用`Sarama`开源的`Kafka`第三方库构建`Kafka`共识组件，可以同时接受处理多个客户端发送的交易消息请求，能够有效提高`Orderer`节点处理交易消息的并发能力。同时，可利用`Kafka`集群在**单一分区内**按序收集相同主题消息（**消息序号唯一**）的功能，来保证交易消息具有确定性的顺序（以消息序号排序），从而实现对交易排序达成全局共识的目的。 

`Kafka`生产者按照主题（`Topic`）生产消息并进行发布，`Kafka`服务器集群自动对消息主题进行分类。同一个主题的消息都会被收集到一个或多个分区文件中，按照`FIFO`的顺序追加到文件尾部，并且每个消息在分区中都会有一个`OFFSET`位置偏移量作为该消息的唯一标识ID。目前，`Hyperledger Fabric`基于`Kafka`集群为**每个通道**创建绑定了一个主题（即链ID，`chainID`），并且只设置一个分区（分区号为0）。Kafka消费者管理多个分区消费者并订阅指定分区的主题消息，包括主题（即`chainID`）、分区号（目前只有1个分区号为0的分区）、起始偏移量（开始订阅的消息位置`offset`）等。

Hyperledger Fabric采用`Kafka`集群对单个或多个`Orderer`排序节点提交的交易消息进行排序。此时，`Orderer`排序节点同时充当`Kafka`集群的消息生产者（分区）和消费者，发布消息与订阅消息到Kafka集群上的同一个主题分区，即先将`Peer`节点提交的交易消息转发给Kafka服务端，同时，从指定主题的`Kafka`分区上按顺序获取排序后的交易消息并自动过滤重启的交易消息。这期间可能会存在网络时延造成获取消息时间的差异。如果不考虑丢包造成消息丢失的情况，则所有`Orderer`节点获取消息的顺序与数量应该是确定的和一致的。同时，采用相同的Kafka共识组件链对象与出块规则等，以保证所有Orderer节点都可以创建与更新相同配置的通道，并切割生成相同的批量交易集合出块，再“同步”构造出相同的区块数据，从而基于`Kafka`集群达成全局共识，以保证区块数据的全局一致性。

### 启动共识组件链对象

启动入口：

> orderer/consensus/kafka/chain.go/Start()

```go
func (chain *chainImpl) Start() {
	go startThread(chain)
}
```

```go
func startThread(chain *chainImpl) {
	...
	//创建kafka生产者
	chain.producer, err = setupProducerForChannel(chain.consenter.retryOptions(), chain.haltChan, chain.SharedConfig().KafkaBrokers(), chain.consenter.brokerConfig(), chain.channel)
	...
	// Kafka生产者发送CONNECT消息建立连接
	if err = sendConnectMessage(chain.consenter.retryOptions(), chain.haltChan, chain.producer, chain.channel); err != nil {
		logger.Panicf("[channel: %s] Cannot post CONNECT message = %s", chain.channel.topic(), err)
	}
	...
	//创建Kafka消费者
	chain.parentConsumer, err = setupParentConsumerForChannel(chain.consenter.retryOptions(), chain.haltChan, chain.SharedConfig().KafkaBrokers(), chain.consenter.brokerConfig(), chain.channel)
	...
	//创建Kafka分区消费者
	chain.channelConsumer, err = setupChannelConsumerForChannel(chain.consenter.retryOptions(), chain.haltChan, chain.parentConsumer, chain.channel, chain.lastOffsetPersisted+1)
	...
	close(chain.startChan) // 已经启动共识组件链对象，不阻塞Broadcast
	chain.errorChan = make(chan struct{}) // 创建errorChan通道，不阻塞Deliver服务处理句柄
	...
	chain.processMessagesToBlocks() //创建消息处理循环，循环处理订阅分区上接收到的消息
}
```

`startThread`函数首先创建`kafka`生产者，发布消息到指定主题（即通道ID）和分区号的通道分区（chain.channel）上。

 然后发送`CONNECT`消息建立连接，该消息指定了主题`Topic`字段为链ID、`Key`字段为分区号0、`Value`字段为`CONNECT`类型消息负载等。订阅该主题的`Kafka`（分区）消费者会接收到该消息。

接着创建指定`Kafka`分区和`Broker`服务器配置的`Kafka`消费者对象，并设置从指定主题（链ID）和分区号（0）的`Kafka`分区上获取消息。

最后，调用`processMessagesToBlocks()`方法创建消息处理循环，负责处理从`Kafka`集群中接收到的订阅消息。

### 处理消息

`processMessagesToBlocks`接收到正常的`Kafka`分区消费者消息会根据`kafka`的消息类型进行处理，包括以下几种类型：

- Kafka- Message_Regular
- KafkaMessage_TimeToCut
- KafkaMessage_Connect

```go
func (chain *chainImpl) processMessagesToBlocks() ([]uint64, error) {
	...
	for { // 消息处理循环
		select {
		...
		case in, ok := <-chain.channelConsumer.Messages(): //接收到正常的Kafka分区消费者消息
			...
			select {
			case <-chain.errorChan: // If this channel was closed...  // 如果该通道已经关闭，则重新创建该通道
				...
			switch msg.Type.(type) { //分析Kafka消息类型
			case *ab.KafkaMessage_Connect: //Kafka连接消息  由于错误而重新恢复Kafka消费者分区订阅流程
				_ = chain.processConnect(chain.ChainID()) //处理CONNECT连接消息， 不做任何事情
				counts[indexProcessConnectPass]++         // 成功处理消息计数增1
			case *ab.KafkaMessage_TimeToCut: // Kafka定时切割生成区块消息
				if err := chain.processTimeToCut(msg.GetTimeToCut(), in.Offset); err != nil {
					logger.Warningf("[channel: %s] %s", chain.ChainID(), err)
					logger.Criticalf("[channel: %s] Consenter for channel exiting", chain.ChainID())
					counts[indexProcessTimeToCutError]++
					return counts, err // TODO Revisit whether we should indeed stop processing the chain at this point
				}
				counts[indexProcessTimeToCutPass]++ // 成功处理消息计数增1
			case *ab.KafkaMessage_Regular: // Kafka常规消息
				if err := chain.processRegular(msg.GetRegular(), in.Offset); err != nil { // 处理Kafka常 规消息
					...
					counts[indexProcessRegularError]++
        }...
      }
		case <-chain.timer: // 超时定时器
			if err := sendTimeToCut(chain.producer, chain.channel, chain.lastCutBlockNumber+1, &chain.timer); err != nil { //发送TimeToCut类型消息，请求打包出块
			...
				counts[indexSendTimeToCutError]++
			} ...
		}
	}
}
```

①：KafkaMessage_Connect类型消息

`Kafka`连接消息用于测试连通`Kafka`分区消费者的工作状态，用于验证`Kafka`共识组件的正常工作状态与排除故障，并调用`chain.processConnect(chain.ChainID())`方法处理该消息。

②：KafkaMessage_TimeToCut类型消息

`processMessagesToBlocks`()方法可调用`chain.processTimeToCut()`方法处理`TIMETOCUT`类型消息。如果消息中的区块号`ttcNumber`不是当前`Orderer`节点当前通道账本中下一个打包出块的区块号（最新区块号`lastCutBlockNumber`+1），则直接丢弃不处理。否则，调用`BlockCutter().Cut()`方法，切割当前该通道上待处理的缓存交易消息列表为批量交易集合`batch（[]*cb.Envelope）`，再调用`CreateNextBlock(batch)`方法构造新区块并提交账本。最后，调用`WriteBlock(block，metadata)`方法，更新区块元数据并提交账本，同时更新Kafka共识组件链对象的最新区块号`lastCutBlockNumber`增1。

事实上，`Orderer`服务集群节点独立打包出块的时间点通常不是完全同步的，同时还可能会重复接收其他Orderer节点提交的TIMETOCUT类型消息（重复区块号）。此时，`Orderer`节点以接收到的第一个`TIMETOCUT`类型消息为准，打包出块并提交到账本，再更新当前通道的最新区块号`lastCutBlockNumber`。这样，`processTimeToCut`()方法就能利用最新的`lastCutBlockNumber`过滤掉其他重复的`TIMETOCUT`类型消息，以保证所有`Orderer`节点上账本区块文件的数据同步，实际上是将原先的时间同步机制转换为消息同步机制。

③：KafkaMessage_Regular类型消息

包括通道配置交易消息（KafkaMessageRegular_CONFIG类型）和普通交易消息（KafkaMessageRegular_NORMAL类型）。 详细的分析将会在`processRegular`方法中体现。

## 处理配置交易消息

我们先大概的看一下ProcessRegular中关于处理配置交易消息的代码部分,因为这部分相当的长，必须先看个概览：

```go
func (chain *chainImpl) processRegular(regularMessage *ab.KafkaMessageRegular, receivedOffset int64) error {
  ...
  commitConfigMsg := func(message *cb.Envelope, newOffset int64){...}
  seq := chain.Sequence() // 获取当前通道的最新配置序号
  ...
  switch regularMessage.Class {
	case ab.KafkaMessageRegular_UNKNOWN: // 未知消息类型
	...
	case ab.KafkaMessageRegular_NORMAL: // 普通交易消息类型
		...
	case ab.KafkaMessageRegular_CONFIG: // 通道配置交易消息
	...
		}
	...
}
```

我们直接跳转到`case ab.KafkaMessageRegular_CONFIG`进行分析：

①：如果regularMessage.OriginalOffset 不为 0

说明这是重新过滤验证和排序的通道配置交易消息。

1.1 过滤重复提交的消息

```go
if regularMessage.OriginalOffset <= chain.lastOriginalOffsetProcessed {}
```

1.2 确认是否是最近重新验证且重新排序的配置交易消息，并且通道配置序号是最新的

```go
if regularMessage.OriginalOffset == chain.lastResubmittedConfigOffset &&regularMessage.ConfigSeq == seq {
  // 因此，关闭通道并解除Broadcast服务处理句柄阻塞等待，通知重新接收消息进行处理
  close(chain.doneReprocessingMsgInFlight) 
}
```

1.3 主动更新本通道的最近重新提交排序的配置交易消息初始偏移量lastResubmitted

存在其他`Orderer`节点重新提交了配置消息，但是本地`Orderer`节点没有重新提交该消息。因此这里需要更新本通道的最近重新提交排序的配置交易消息初始偏移量lastResubmitted。

```go
if chain.lastResubmittedConfigOffset < regularMessage.OriginalOffset {
				chain.lastResubmittedConfigOffset = regularMessage.OriginalOffset
			}
```

②：regularMessage.OriginalOffset为 0

说明是第一次提交通道配置交易消息，而不是重新验证和重新排序的。

2.1 如果消息中的配置序号regularMessage.ConfigSeq小于当前通道的最新配置序号seq

则说明已经更新了通道配置（配置序号较高），然后再处理当前配置交易消息（配置序号较低）。将会调用`ProcessConfigMsg`重新过滤和处理该消息。

接着通过`configure`重新提交该配置消息进行排序，重置消息初始偏移量。然后再更新最近重新提交消息的偏移量。

```go
if regularMessage.ConfigSeq < seq {
  ...
	configEnv, configSeq, err := chain.ProcessConfigMsg(env)
  if err := chain.configure(configEnv, configSeq, receivedOffset); err != nil {...}
  
  // 阻塞接收消息处理，更新最近重新提交消息的偏移量
  chain.lastResubmittedConfigOffset = receivedOffset 
  //创建通道阻塞Broadcast服务接收处理消息
  chain.doneReprocessingMsgInFlight = make(chan struct{})
}
```

③：提交配置交易消息执行通道管理操作

经过上面的①和②过滤掉不符合条件的情况，接下来就提交配置交易消息执行通道管理操作，核心函数：`commitConfigMsg(env, offset)`

3.1 将当前缓存交易消息切割成批量交易集合

```go
batch := chain.BlockCutter().Cut()
```

3.2 创建新区块block

```go
block := chain.CreateNextBlock(batch)
```

3.3 构造Kafka元数据

```go
metadata := utils.MarshalOrPanic(&ab.KafkaMetadata{ //构造Kafka元数据
				LastOffsetPersisted:         receivedOffset - 1, // 偏移量减1
				LastOriginalOffsetProcessed: chain.lastOriginalOffsetProcessed,
				LastResubmittedConfigOffset: chain.lastResubmittedConfigOffset,
			})
```

3.4 写入区块

通过区块写组件提交新区块到账本，更新当前通道的最新区块号chain.lastCutBlockNumber增1

```go
chain.WriteBlock(block, metadata)
chain.lastCutBlockNumber++  
```

接着更新本链的lastOriginal- OffsetProcessed为newOffset参数，然后做和上面差不多的事情：

```go
chain.lastOriginalOffsetProcessed = newOffset
		block := chain.CreateNextBlock([]*cb.Envelope{message}) // 构造新区块
		metadata := utils.MarshalOrPanic(&ab.KafkaMetadata{     // 构造Kafka元数据
			LastOffsetPersisted:         receivedOffset,
			LastOriginalOffsetProcessed: chain.lastOriginalOffsetProcessed,
			LastResubmittedConfigOffset: chain.lastResubmittedConfigOffset,
		})
		chain.WriteConfigBlock(block, metadata) // 写入配置区块
		chain.lastCutBlockNumber++              // 最新区块号增1
```

不管是上面的`WriteBlock`还是`WriteConfigBlock`底层都是调用的`commitBlock`，如下：

```go
func (bw *BlockWriter) commitBlock(encodedMetadataValue []byte) {
	... // 添加块签名
	bw.addBlockSignature(bw.lastBlock)
  // 添加最新的配置签名
	bw.addLastConfigSignature(bw.lastBlock)
	// 写入新块
	err := bw.support.Append(bw.lastBlock)
	...
}
```

接下来再讨论kafka共识组件如何处理普通交易消息的。

## 处理普通交易消息

还是先回到 `processRegular`方法，关于处理普通消息的方法大概如下：

```go
func (chain *chainImpl) processRegular(regularMessage *ab.KafkaMessageRegular, receivedOffset int64) error {
  ...
  case ab.KafkaMessageRegular_NORMAL: // 普通交易消息类型
		// 如果OriginalOffset不是0，则说明该消息是重新验证且重新提交排序的
		if regularMessage.OriginalOffset != 0 {
			...
			// 如果消息偏移量不大于lastOriginalOffsetProcessed最近已处理消息的偏移量，
			// 则说明已经处理过该消息，此时应丢弃返回，防止重复处理其他Orderer提交的相同偏移 量的普通交易消息
			if regularMessage.OriginalOffset <= chain.lastOriginalOffsetProcessed {
				...
		}

		// // 检查通道的配置序号是否更新
		if regularMessage.ConfigSeq < seq {
			...
			//// 消息的配置序号低，需要重新验证过滤消息
			configSeq, err := chain.ProcessNormalMsg(env)
			...
			//重新提交普通交易消息
      if err := chain.order(env, configSeq, receivedOffset); err != nil {}
				...
		}
		// advance lastOriginalOffsetProcessed iff message is re-validated and re-ordered
		//当且仅当消息重新验证和重新排序时，才需要修正lastOriginalOffsetProcessed偏移量
		offset := regularMessage.OriginalOffset
		if offset == 0 {
			offset = chain.lastOriginalOffsetProcessed
		}
		// 提交处理普通交易消息，offset为最近处理的普通交易消息偏移量
		commitNormalMsg(env, offset)
}
```

处理普通交易消息的流程与处理配置交易消息的流程基本类似，主要看最后的`commitNormalMsg(env, offset)`，我们来继续分析：

```go
commitNormalMsg := func(message *cb.Envelope, newOffset int64) {
		//// 添加所接收的消息到缓存交易消息列表，并切割成批量交易集合列表batches
		batches, pending := chain.BlockCutter().Ordered(message)
		...
		if len(batches) == 0 {
			// 如果不存在批量交易集合，则启动定时器周期性地发送切割出块消息n
			chain.lastOriginalOffsetProcessed = newOffset
			if chain.timer == nil {
				chain.timer = time.After(chain.SharedConfig().BatchTimeout())
			...
			return
		}
		chain.timer = nil
		offset := receivedOffset // 设置当前消息偏移量
		if pending || len(batches) == 2 {
			offset-- // 计算第1个批量交易消息的偏移量是offset减1
		} else {  // 只有1个批量交易集合构成1个区块
			//// 设置第1个批量交易集合的消息偏移量为newOffset
			chain.lastOriginalOffsetProcessed = newOffset
		}
		//// 构造并提交第1个区块
		block := chain.CreateNextBlock(batches[0])
		metadata := utils.MarshalOrPanic(&ab.KafkaMetadata{
			LastOffsetPersisted:         offset,
			LastOriginalOffsetProcessed: chain.lastOriginalOffsetProcessed,
			LastResubmittedConfigOffset: chain.lastResubmittedConfigOffset,
		})
		chain.WriteBlock(block, metadata) // 更新区块元数据，并提交区块到账本
		chain.lastCutBlockNumber++ // 更新当前通道上最近出块的区块号增1
	...
		// Commit the second block if exists
		//// 检查第2个批量交易集合，构造并提交第2个区块
		if len(batches) == 2 {
			chain.lastOriginalOffsetProcessed = newOffset
			offset++ // 设置第2个批量交易集合的消息偏移量offset加1

			block := chain.CreateNextBlock(batches[1])
			metadata := utils.MarshalOrPanic(&ab.KafkaMetadata{
				LastOffsetPersisted:         offset,
				LastOriginalOffsetProcessed: newOffset,
				LastResubmittedConfigOffset: chain.lastResubmittedConfigOffset,
			})
			chain.WriteBlock(block, metadata)
			chain.lastCutBlockNumber++
			...
		}
	}
```

首先将新的普通交易消息添加到当前的缓存交易列表，并切割成批量交易集合列表batches ,但最多只能包含2个批量交易集合，并且第2个批量交易集合最多包含1个交易。最终也是调用的`WriteBlock`写入到账本。

到此为止整个`processRegular`()方法处理消息结束。

## 总结及参考

kafka共识排序的逻辑其实是比较简单的，大概的流程如下 ：

![image-20210126092717144](https://tva1.sinaimg.cn/large/008eGmZEgy1gn0u9hflmfj318e0u0wj5.jpg)

> https://github.com/blockchainGuide/ (文章图片代码资料在里面)
>
> 微信公众号：区块链技术栈





