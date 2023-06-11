

// start event bus

// get State

// start the machine

- enterNewRound
- startRoutines

// wait for proposal

// wait for prevote

// wait for precommit

// wait to finish commit, propose in next height

ensureNewBlock(newBlockCh, height)

// finalizeCommit

// 

1. state 负责什么工作
2. 监听了哪些事件
   - voteCh := subscribeUnBuffered(cs.eventBus, types.EventQueryVote)
   -  propCh := subscribe(cs.eventBus, types.EventQueryCompleteProposal)
   -  newRoundCh := subscribe(cs.eventBus, types.EventQueryNewRound)



> https://liukay.com/posts/4334
>
> https://arxiv.org/pdf/1807.04938.pdf



## 服务启动，进入第一轮共识

```go
func (cs *State) OnStart() error{
  cs.loadWalFile()
  ....
  cs.timeoutTicker.Start()
  ...
  loop
  for{
    ??
  }
  cs.evsw.Start()
  ...
  cs.checkDoubleSigningRisk(cs.Height)
  ...
  go cs.receiveRoutine(0)
  ...
  cs.scheduleRound0(cs.GetRoundState()) // 他这个思路是往timeoutTicker发送消息，由receiveRoutine来处理，在新高度开启新round
}
```



## 共识消息处理

> receiveRoutine

### 处理交易



```GO
实际这一连串的代码都是从tendermint 共识启动发出的，也就是说他的共识启动是往timeouttiker中发送超时信息，从而重置timeoutTicker计时器，然后再次触发timer.C来往cs.timeoutTicker.Chan()写入数据，从而触发handleTimeout的真正处理逻辑
```



Handlemsg 和handleTimeOut 

handleMsg:

- peerMsgQueue
- cs.internalMsgQueue:

handleTimeout:从tockchan获取的各种超时信息：

```SHELL
RoundStepNewHeight     = RoundStepType(0x01) // Wait til CommitTime + timeoutCommit
	RoundStepNewRound      = RoundStepType(0x02) // Setup new round and go to RoundStepPropose
	RoundStepPropose       = RoundStepType(0x03) // Did propose, gossip proposal
	RoundStepPrevote       = RoundStepType(0x04) // Did prevote, gossip prevotes
	RoundStepPrevoteWait   = RoundStepType(0x05) // Did receive any +2/3 prevotes, start timeout
	RoundStepPrecommit     = RoundStepType(0x06) // Did precommit, gossip precommits
	RoundStepPrecommitWait = RoundStepType(0x07) // Did receive any +2/3 precommits, start timeout
	RoundStepCommit        = RoundStepType(0x08) // Entered commit state machine
```



## 流程

Func (cs *State) enterNewRound(height int64, round int32) 

​	cs.eventBus.PublishEventNewRound(cs.NewRoundEvent()) ??? 发布出去的newround别人怎么处理的

​	cs.enterPropose(height, round)	



enterPropose

 - cs.scheduleTimeout(cs.config.Propose(round), height, round, cstypes.RoundStepPropose) 这里会开始一个超时计时器，如果在 Propose 阶段无法及时收到区块提案和所有区块部件，则节点会进入到 Prevote 阶段，以保证共识算法的正常运行

 -  cs.decideProposal(height, round)



decideProposal -> defaultDecideProposal

- cs.createProposalBlock(context.TODO()): 从mempool的state/txs创建一个新的提案块。 （具体如何create 看下面的）

- blockParts, err = block.MakePartSet(types.BlockPartSizeBytes)：分解区块为一个个的64Kb的数据 (这个重点理解，运用到了merkle proof)

- 构造proposal: 

  - NewProposal

  - cs.privValidator.SignProposal

  - 发送proposal和区块部件（他实际是切割了区块，降低网络延迟，提高网络传输效率）

    ```go
    func (cs *State) sendInternalMessage(mi msgInfo) {
    	select {
    	case cs.internalMsgQueue <- mi:
    	default:
    		// NOTE: using the go-routine means our votes can
    		// be processed out of order.
    		// TODO: use CList here for strict determinism and
    		// attempt push to internalMsgQueue in receiveRoutine
    		cs.Logger.Debug("internal msg queue is full; using a go-routine")
    		go func() { cs.internalMsgQueue <- mi }()
    	}
    }
    这段代码如果不对internalMsgQueue进行限制，则可以进行DDOS攻击，会导致下面default开启大量协程，
    tendermint只是进行了1000条消息的设置
    Tendermint 会对节点的行为进行监控和限制，如限制每个节点发送 mi 消息的速率，防止节点恶意发送大量消息。
    Tendermint 还具有抗拒绝服务（DoS）攻击的机制，在检测到某个节点发送大量无效消息时，会将该节点列入黑名单，不再接收其消息
    ```

  - 接着就直接handlemsg了

    - 保存proposal和组装block，主要看下组装block(BlockPartMessage)：
      - cs.addProposalBlockPart (这一部分可能enterPrevote 或者	tryFinalizeCommit)
        - cs.eventBus.PublishEventCompleteProposal(cs.CompleteProposalEvent()) 发布事件
      - handleCompleteProposal （这里面需要细看，我只讲直接进入到enterPrevote阶段）
        - 

​	

cs.createProposalBlock(context.TODO())

 - cs.blockExec.CreateProposalBlock(ctx, cs.Height, cs.state, lastExtCommit, proposerAddr)
	- 


##疑问

1. 为什么有自定义的函数作为参数传入





http://yangzhe.me/2022/08/13/tendermint-bft/



## 关键结构

### reactor

```go
type Reactor struct {
	p2p.BaseReactor // BaseService + p2p.Switch

	conS *State   // 见下state

	mtx      cmtsync.RWMutex
	waitSync bool
	eventBus *types.EventBus
	rs       *cstypes.RoundState

	Metrics *Metrics
}
```



### state

```go
type State struct {
	service.BaseService

	// config details
	config        *cfg.ConsensusConfig
	privValidator types.PrivValidator // for signing votes

	// store blocks and commits
	blockStore sm.BlockStore

	// create and execute blocks
	blockExec *sm.BlockExecutor

	// notify us if txs are available
	txNotifier txNotifier

	// add evidence to the pool
	// when it's detected
	evpool evidencePool

	// internal state
	mtx cmtsync.RWMutex
	cstypes.RoundState
	state sm.State // State until height-1.
	// privValidator pubkey, memoized for the duration of one block
	// to avoid extra requests to HSM
	privValidatorPubKey crypto.PubKey

	// state changes may be triggered by: msgs from peers,
	// msgs from ourself, or by timeouts
	peerMsgQueue     chan msgInfo
	internalMsgQueue chan msgInfo
	timeoutTicker    TimeoutTicker  

	// information about about added votes and block parts are written on this channel
	// so statistics can be computed by reactor
	statsMsgQueue chan msgInfo

	// we use eventBus to trigger msg broadcasts in the reactor,
	// and to notify external subscribers, eg. through a websocket
	eventBus *types.EventBus   // //使用eventBus来触发反应器中的消息广播，并通知外部订阅者，例如通过websocket

	// a Write-Ahead Log ensures we can recover from any kind of crash
	// and helps us avoid signing conflicting votes
	wal          WAL
	replayMode   bool // so we don't log signing errors during replay
	doWALCatchup bool // determines if we even try to do the catchup

	// for tests where we want to limit the number of transitions the state makes
	nSteps int

	// some functions can be overwritten for testing
	decideProposal func(height int64, round int32)
	doPrevote      func(height int64, round int32)
	setProposal    func(proposal *types.Proposal) error

	// closed when we finish shutting down
	done chan struct{}

	// synchronous pubsub between consensus state and reactor.
	// state only emits EventNewRoundStep and EventVote
	evsw cmtevents.EventSwitch // 共识状态和反应器之间的同步pubsub。状态仅发出EventNewRoundStep和EventVote
}
```

### EventSwitch

cmtevents.EventSwitch 是一个事件总线，用于在不同的模块之间传递事件。EventSwitch 的设计思想是将事件的发送者和接收者解耦，从而实现模块间的松耦合。

具体来说，每个模块都可以将自己感兴趣的事件类型注册到 EventSwitch 中。当某个模块触发了一个事件时，EventSwitch 会将该事件发送给所有已注册的模块。这样，不同的模块就可以在不了解彼此具体实现的情况下进行协作。

在 Tendermint 中，EventSwitch 被广泛用于不同模块之间的交互。例如，在共识模块和反应器之间，EventSwitch 被用于同步共识状态和反应器。具体来说，共识状态会触发两种类型的事件：EventNewRoundStep 和 EventVote。这些事件会被发送到 EventSwitch 中，并被订阅了这些事件类型的反应器模块接收。反应器模块会根据这些事件执行相应的操作，例如更新状态或发送消息。

## 节点启动流程

从NewRunNodeCmd开始

## eventbus启动流程

eventbus 是cometBFT节点内部的消息总线，其启动最终是启动了pubsub.

```GO
func (s *Server) loop(state state) {
loop:
	for cmd := range s.cmds {
		switch cmd.op {
		case unsub:
			if cmd.query != nil {
				state.remove(cmd.clientID, cmd.query.String(), ErrUnsubscribed)
			} else {
				state.removeClient(cmd.clientID, ErrUnsubscribed)
			}
		case shutdown:
			state.removeAll(nil)
			break loop
		case sub:
			state.add(cmd.clientID, cmd.query, cmd.subscription)
		case pub:
			if err := state.send(cmd.msg, cmd.events); err != nil {
				s.Logger.Error("Error querying for events", "err", err)
			}
		}
	}
}
```

这段代码实现了一个循环，用于处理来自 `cmds` 通道的命令。具体来说，循环会不断地从 `cmds` 通道中接收命令，然后根据命令类型进行相应的处理。

在代码中，`state` 参数代表当前的状态，即已经订阅的客户端和事件查询。`cmds` 通道中的命令包括四种类型：

- `unsub`：取消订阅命令，用于取消指定客户端或查询的订阅。
- `shutdown`：关闭命令，用于关闭消息服务器并清空当前状态。
- `sub`：订阅命令，用于添加新的订阅。
- `pub`：发布命令，用于向订阅者发送消息。

当接收到 `unsub` 命令时，会根据命令中的参数从当前状态中移除相应的订阅。如果命令中指定了查询条件，则移除与该查询条件相匹配的订阅；否则，移除指定客户端的所有订阅。

当接收到 `shutdown` 命令时，会将当前状态中的所有订阅都移除，并退出循环。

当接收到 `sub` 命令时，会根据命令中的参数添加新的订阅。

当接收到 `pub` 命令时，会将消息和相关的事件发送给所有匹配的订阅者。如果发送消息时出现错误，则会记录日志。

## consensusReactor启动流程

从NewRunNodeCmd开始，一直到reactor的onstart启动：

```go
func (conR *Reactor) OnStart() error {
	// start routine that computes peer statistics for evaluating peer quality
	go conR.peerStatsRoutine()

	conR.subscribeToBroadcastEvents()
	go conR.updateRoundStateRoutine()

	if !conR.WaitSync() {
		err := conR.conS.Start()
	}
	return nil
}
```

- 启动计算peer统计信息以评估peer质量的协程
- 订阅新round step ，并使用state上定义的内部pubsub进行投票，以便在接收时将其广播给peer。
- 定期更新共识状态的RoundState
- 如若没有完成状态同步，则要启动共识状态



订阅的细节：

```go
func (conR *Reactor) subscribeToBroadcastEvents() {
	const subscriber = "consensus-reactor"
	if err := conR.conS.evsw.AddListenerForEvent(subscriber, types.EventNewRoundStep,
		func(data cmtevents.EventData) {
			conR.broadcastNewRoundStepMessage(data.(*cstypes.RoundState))
		}); err != nil {
		conR.Logger.Error("Error adding listener for events", "err", err)
	}

	if err := conR.conS.evsw.AddListenerForEvent(subscriber, types.EventValidBlock,
		func(data cmtevents.EventData) {
			conR.broadcastNewValidBlockMessage(data.(*cstypes.RoundState))
		}); err != nil {
		conR.Logger.Error("Error adding listener for events", "err", err)
	}

	if err := conR.conS.evsw.AddListenerForEvent(subscriber, types.EventVote,
		func(data cmtevents.EventData) {
			conR.broadcastHasVoteMessage(data.(*types.Vote))
		}); err != nil {
		conR.Logger.Error("Error adding listener for events", "err", err)
	}
}
```

可以看到consensus reactor订阅了三种类型的事件，EventNewRoundStep、EventValidBlock、EventVote， 当共识状态发布这些事件时候，eventswitch会将事件广播给订阅了这些事件的对象

## timeoutTiker的启动

```go
func (t *timeoutTicker) OnStart() error {

	go t.timeoutRoutine()

	return nil
}
func (t *timeoutTicker) timeoutRoutine() {
	t.Logger.Debug("Starting timeout routine")
	var ti timeoutInfo
	for {
		select {
		case newti := <-t.tickChan:
			t.Logger.Debug("Received tick", "old_ti", ti, "new_ti", newti)

			// ignore tickers for old height/round/step
			if newti.Height < ti.Height {
				continue
			} else if newti.Height == ti.Height {
				if newti.Round < ti.Round {
					continue
				} else if newti.Round == ti.Round {
					if ti.Step > 0 && newti.Step <= ti.Step {
						continue
					}
				}
			}

			// stop the last timer
			t.stopTimer()

			// update timeoutInfo and reset timer
			// NOTE time.Timer allows duration to be non-positive
			ti = newti
			t.timer.Reset(ti.Duration)
			t.Logger.Debug("Scheduled timeout", "dur", ti.Duration, "height", ti.Height, "round", ti.Round, "step", ti.Step)
		case <-t.timer.C:
			t.Logger.Info("Timed out", "dur", ti.Duration, "height", ti.Height, "round", ti.Round, "step", ti.Step)
			// go routine here guarantees timeoutRoutine doesn't block.
			// Determinism comes from playback in the receiveRoutine.
			// We can eliminate it by merging the timeoutRoutine into receiveRoutine
			//  and managing the timeouts ourselves with a millisecond ticker
			go func(toi timeoutInfo) { t.tockChan <- toi }(ti)
		case <-t.Quit():
			return
		}
	}
}
```

- tickChan ：接收各种超时信息，然后重置定时器
- timer.C: 根据tickChan ，将tickChan传过来的超时信息送到tockChan 进行处理
- 在receiveRoutine里，会有case一直监听tockChan，来处理超时消息



## 完整的共识流程阶段

### 共识过程的启动

也是从节点启动命令开始，一直调用到state的onstart方法：

```go
func (cs *State) OnStart() error{
  cs.loadWalFile()
  ....
  cs.timeoutTicker.Start()
  ...
  loop
  for{
    ????
  }
  cs.evsw.Start()
  ...
  cs.checkDoubleSigningRisk(cs.Height)
  ...
  go cs.receiveRoutine(0)
  ...
  cs.scheduleRound0(cs.GetRoundState()) // 他这个思路是往timeoutTicker发送消息，由receiveRoutine来处理，在新高度开启新round
}
```

- 处理状态转换消息 ： cs.receiveRoutine(0)
- 开启第一轮共识： cs.scheduleRound0(cs.GetRoundState())

首先还是从开启第一轮共识开始：

### 开启第一轮共识

①：首先发送timeoutInfo{duration, height, round, step}到tickchan , 然后重置定时器，将超时信息发往tockChan 

②：tockchan会通过handleTImeout来处理消息，进入到enterNewRound

③：进入到enterprose， 如果是proposer， cs.decideProposal(height, round)进入组块流程

 - block, err = cs.createProposalBlock(context.TODO()) 

 - block.MakePartSet(types.BlockPartSizeBytes) 切割区块

 - cs.sendInternalMessage(msgInfo{&ProposalMessage{proposal}, ""})  将proposal和blockparts发送到内部消息队列里

 - cs.sendInternalMessage(msgInfo{&BlockPartMessage{cs.Height, cs.Round, part}, ""})

 - 到此会把这两部分消息发给内部共识状态的 receiveroutine来处理

 - 外部的话需要再次回到enterPropose， 在defer中通过eventBus 通知外部调用者，比如通过websocket订阅了事件的，然后通过eventswitch来通知内部订阅了NewRoundStep事件的组件（这里指consensus reactor）， 通过FireEvent 来触发监听器回调他注册的处理函数

   ```go
   conR.broadcastNewRoundStepMessage(data.(*cstypes.RoundState))
   const subscriber = "consensus-reactor"
   	if err := conR.conS.evsw.AddListenerForEvent(subscriber, types.EventNewRoundStep,
   		func(data cmtevents.EventData) {
   			conR.broadcastNewRoundStepMessage(data.(*cstypes.RoundState))
   		}); err != nil {
   		conR.Logger.Error("Error adding listener for events", "err", err)
   	}
   
   func (conR *Reactor) broadcastNewRoundStepMessage(rs *cstypes.RoundState) {
   	nrsMsg := makeRoundStepMessage(rs)
   	conR.Switch.Broadcast(p2p.Envelope{
   		ChannelID: StateChannel,
   		Message:   nrsMsg,
   	})
   }
   ```

④：上面提到的组块流程如下，回到cs.createProposalBlock

- 获取可验证的恶意行为

- 从mempool中获取最大maxBytes 的交易（都是字节切片）

- 他会请求APP执行PrepareProposal （app层可以修改删除事务，这是cometBFT 后加的），理由：

- // Ref: https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-060-abci-1.0.md

  // Ref: https://github.com/cometbft/cometbft/blob/main/spec/abci/abci%2B%2B_basic_concepts.md

- 构造区块，基本就完成了

⑤：通过P2P广播出去的消息，在哪里接收，这部分得看p2p如何处理消息

⑥：由我们从p2p发出去的共识消息，会经过其他节点的p2p模块转发到consensus reactor 的func (conR *Reactor) Receive(e p2p.Envelope) 来进行处理， 作为回调函数处理p2p消息，应该有一个peer的循环，收到相关消息，会调用注册的reactor的回调函数

- 上面的proposal消息 和blockpart消息会进入到DataChannel， 关于状态转换的消息会进入到StateChannel，先看下datachannel中的消息
- datachannel中的消息会进入到peerMsgQueue 进行处理，会设置propoal,和添加完整块，会根据当前状态，决定进入enterPrevote 还是enterPrecommit还是tryFinalizeCommit
- statechannel中的roundstep会更新peerstate 

⑦：接着正常步骤，进入到Prevote

- 步骤跟上面一致, 进入到addnote处理，会执行enterprecommit处理

⑧：进入entercommit

⑨：进入到finalizecommit

- 持久化区块数据
- 调用 baseapp的 FinalizeBlock （执行begin blcok , txs ,endblock）返回了apphash
- 更新内存池
- cs.scheduleRound0(&cs.RoundState) 继续新round

## 技术细节

1. eventswitch 事件总线的设计
2. block parts 如何通过merkle proof 来证明
3. 





### enterNewRound

- 更新validators （来源）
- 更新Proposer （算法细节待了解）
- 更新RoundStep Height -> RoundStepNewRound
- Eventbus 发布事件
- enterPropose

### enterPropose

-  决定proposal block
  - 从evpool获取作恶行为数据
  - 从mempool 获取交易数据
  - 通过现有的高度，txs, proposeraddr等数据构造block
  - 请求APP执行PrepareProposal,由应用程序选择交易，之前tendermint是没有这一步的，设计原理参考：
    - https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-060-abci-1.0.md
    - https://github.com/cometbft/cometbft/blob/main/spec/abci/abci%2B%2B_basic_concepts.md
  - 最后再重新构造一下block
  - 将构造的块进行切割，并计算merkle 树 （TODO）
- 构造proposal 并签名，这个proposal并不是区块，只是区块的hash,以及高度，当前round，区块分割的信息
- 将proposal和分割的blockparts 发到内部消息队列进行处理 （异步消息处理），直接返回
- 更新RoundStepNewRound -> RoundStepPropose
- FireEvent (EventNewRoundStep),reactor监听此事件并通过p2p 发送broadcastNewRoundStepMessage消息

### enterprevote

- 进入到这个阶段，是获取到了2/3票，cs.isProposalComplete（TODO）
- 执行预投票
  - 从共识角度验证提案块，区块无效则投nil票
  - 调用app 来执行提案块
  - 有效则签署投票，并将投票信息发布到内部消息队列中
- 更新RoundStepPropose -> RoundStepPrevote
- FireEvent(eventVote), broadcastHasVoteMessage

### enterPrecommit

- 更新RoundStepPrevote -> RoundStepPrecommit

## enterCommit

- 更新RoundStepPrecommit -> RoundStepCommit
-  FireEven（EventValidBlock）
- finalizeCommit
  - 校验block
  - 存储块到block store
  - applyblock
    - buildLastCommitInfo
    - blockExec.proxyApp.FinalizeBlock（执行startblock,txs, endblock）, 会返回apphash和更新的validator，以及txresult
    - 之后会调用app 提交更新完的kv state store 
    - 最后block store 会保存app层返回的数据
  - 开启新round scheduleRound0

### 分别介绍上述几个消息

#### proposal 

#### blockparts

#### vote





### 如何验证validator

