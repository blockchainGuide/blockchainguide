> 死磕以太坊源码分析之txpool
>
> 请结合以下代码阅读:https://github.com/blockchainGuide/
>
> 写文章不易，也希望大家多多指出问题


## 交易池概念原理

交易池工作概况：

![image-20201225104748102](https://tva1.sinaimg.cn/large/0081Kckwgy1glzwre4v4ej31120tcgpa.jpg)

1. 交易池的数据来源主要来自：
   - 本地提交，也就是第三方应用通过调用本地以太坊节点的`RPC`服务所提交的交易；
   - 远程同步，是指通过广播同步的形式，将其他以太坊节点的交易数据同步至本地节点;
2. 交易池中交易去向：被Miner模块获取并验证，用于挖矿；挖矿成功后写进区块并被广播
3. `Miner`取走交易是复制，交易池中的交易并不减少。直到交易被写进规范链后才从交易池删除；
4. 交易如果被写进分叉，交易池中的交易也不减少，等待重新打包。

## 关键数据结构

### TxPoolConfig

```go
type TxPoolConfig struct {
	Locals    []common.Address // 本地账户地址存放
	NoLocals  bool             // 是否开启本地交易机制
	Journal   string           // 本地交易存放路径
	Rejournal time.Duration    // 持久化本地交易的间隔
	PriceLimit uint64         // 价格超出比例，若想覆盖一笔交易的时候，若价格上涨比例达不到要求，那么不能覆盖
	PriceBump  uint64 // 替换现有交易的最低价格涨幅百分比（一次）
	AccountSlots uint64 // 每个账户的可执行交易限制
	GlobalSlots  uint64 // 全部账户最大可执行交易
	AccountQueue uint64 // 单个账户不可执行的交易限制
	GlobalQueue  uint64 // 全部账户最大非执行交易限制
	Lifetime time.Duration // 一个账户在queue中的交易可以存活的时间
}
```

默认配置：

> ```go
> Journal:   "transactions.rlp",
> Rejournal: time.Hour,
> 
> PriceLimit: 1,
> PriceBump:  10,
> 
> AccountSlots: 16,
> GlobalSlots:  4096,
> AccountQueue: 64,
> GlobalQueue:  1024,
> 
> Lifetime: 3 * time.Hour
> ```

### TxPool

```go
type TxPool struct {
	config      TxPoolConfig // 交易池配置
	chainconfig *params.ChainConfig // 区块链配置
	chain       blockChain // 定义blockchain接口
	gasPrice    *big.Int
	txFeed      event.Feed //时间流
	scope       event.SubscriptionScope // 订阅范围
	signer      types.Signer //签名
	mu          sync.RWMutex

	istanbul bool // Fork indicator whether we are in the istanbul stage.

	currentState  *state.StateDB // 当前头区块对应的状态
	pendingNonces *txNoncer      // Pending state tracking virtual nonces
	currentMaxGas uint64         // Current gas limit for transaction caps

	locals  *accountSet // Set of local transaction to exempt from eviction rules
	journal *txJournal  // Journal of local transaction to back up to disk

	pending map[common.Address]*txList   // All currently processable transactions
	queue   map[common.Address]*txList   // Queued but non-processable transactions
	beats   map[common.Address]time.Time // Last heartbeat from each known account
	all     *txLookup                    // All transactions to allow lookups
	priced  *txPricedList                // All transactions sorted by price

	chainHeadCh     chan ChainHeadEvent
	chainHeadSub    event.Subscription
	reqResetCh      chan *txpoolResetRequest
	reqPromoteCh    chan *accountSet
	queueTxEventCh  chan *types.Transaction
	reorgDoneCh     chan chan struct{}
	reorgShutdownCh chan struct{}  // requests shutdown of scheduleReorgLoop
	wg              sync.WaitGroup // tracks loop, scheduleReorgLoop
}

```



## txpool初始化

`Txpool`初始化主要做了以下几件事：

①：检查配置  配置有问题则用默认值填充

```go
   config = (&config).sanitize()
```

   对于这部分的检查查看`TxPoolConfig`的字段。

②：初始化本地账户

```go
   pool.locals = newAccountSet(pool.signer)
```

③：将配置的本地账户地址加到交易池

```go
   pool.locals.add(addr)
```

   我们在安装以太坊客户端可以指定一个数据存储目录，此目录便会存储着所有我们导入的或者通过本地客户端创建的帐户`keystore`文件。而这个加载过程便是从该目录加载帐户数据

④：更新交易池

```go
   pool.reset(nil, chain.CurrentBlock().Header())
```

⑤：创建所有交易存储的列表，所有交易的价格用最小堆存放

```go
   pool.priced = newTxPricedList(pool.all)
```

   通过排序，优先处理`gasprice`越高的交易。

⑥：如果本地交易开启 那么从本地磁盘加载本地交易

```go
   if !config.NoLocals && config.Journal != "" {
   		pool.journal = newTxJournal(config.Journal)
   
   		if err := pool.journal.load(pool.AddLocals); err != nil {
   			log.Warn("Failed to load transaction journal", "err", err)
   		}
   		if err := pool.journal.rotate(pool.local()); err != nil {
   			log.Warn("Failed to rotate transaction journal", "err", err)
   		}
   	}
```

⑦：订阅链上事件消息

```go
   pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)
```

⑧：开启主循环

```go
   go pool.loop()
```

> 注意：local交易比remote交易具有更高的权限，一是不轻易被替换；二是持久化，即通过一个本地的journal文件保存尚未打包的local交易。所以在节点启动的时候，优先从本地加载local交易。
>
> 本地地址会被加入白名单，凡由此地址发送的交易均被认为是local交易，不论是从本地递交还是从远端发送来的。

到此为止交易池加载过程结束。



## 添加交易到txpool

之前我们说过交易池中交易的来源一方面是其他节点广播过来的，一方面是本地提交的，追根到源代码一个是`AddLocal`，一个是`AddRemote`,不管哪个都会调用`addTxs`。我们对添加交易的讨论就会从这个函数开始，它主要做了以下几件事,先用一张简图说明一下：

![image-20201225104721173](https://tva1.sinaimg.cn/large/0081Kckwgy1glzxhi23euj31ak0u0h34.jpg)

1. 过滤池中已经存在的交易

   ```go
   if pool.all.Get(tx.Hash()) != nil {
     errs[i] = fmt.Errorf("known transaction: %x", tx.Hash())
   			knownTxMeter.Mark(1)
   			continue
   		}
   ```

2. 将交易添加到队列中

   ```go
   newErrs, dirtyAddrs := pool.addTxsLocked(news, local)
   ```

   ```go
   进入到addTxsLocked函数中：
   replaced, err := pool.add(tx, local)
   ```

   进入到 `pool.add`函数中，这个`add`函数相当重要，它是将交易添加到`queue`中，等待后面的promote，到`pending`中去。如果在`queue`或者`pending`中已经存在，并且它的gas price更高时，将覆盖之前的交易。下面来拆开的分析一下add 这个函数。

   ①：看交易是否收到过，如果已经收到过就丢弃

   ```GO
   if pool.all.Get(hash) != nil {
   		log.Trace("Discarding already known transaction", "hash", hash)
   		knownTxMeter.Mark(1)
   		return false, fmt.Errorf("known transaction: %x", hash)
   	}
   ```

   ②：如果交易没通过验证也要丢弃，这里的重点是验证函数：

   ```go
   validateTx: 主要做了以下几件事
   - 交易大小不能超过32kb
   - 交易金额不能为负
   - 交易gas值不能超出当前交易池设定的gaslimit
   - 交易签名必须正确
   - 如果交易为远程交易，则需验证其gasprice是否小于交易池gasprice最小值，如果是本地，优先打包，不管gasprice
   - 判断当前交易nonce值是否过低
   - 交易所需花费的转帐手续费是否大于帐户余额  cost == V + GP * GL
   - 判断交易花费gas是否小于其预估花费gas
   ```

   ③：如果交易池已满，丢弃价格过低的交易

   ```go
   if uint64(pool.all.Count()) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
   		if !local && pool.priced.Underpriced(tx, pool.locals) {
   			...
   		}
   		drop := pool.priced.Discard(pool.all.Count()-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
   		for _, tx := range drop {
   			...
   			pool.removeTx(tx.Hash(), false)
   		}
   	}
   ```
   
   注意这边的`GlobalSlots`和`GlobalQueue` ，就是我们说的`pending`和`queue`的最大容量，如果交易池的交易数超过两者之和，就要丢弃价格过低的交易。
   
   
   

④：判断当前交易在pending队列中是否存在`nonce`值相同的交易。存在则判断当前交易所设置的`gasprice`是否超过设置的`PriceBump`百分比，超过则替换覆盖已存在的交易，否则报错返回`替换交易gasprice过低`，并且把它扔到`queue`队列中`(enqueueTx)`。

```go
   if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
		// Nonce already pending, check if required price bump is met
   		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted {
   			pendingDiscardMeter.Mark(1)
   			return false, ErrReplaceUnderpriced
   		}
   		// New transaction is better, replace old one
   		if old != nil {
   			pool.all.Remove(old.Hash())
   			pool.priced.Removed(1)
   			pendingReplaceMeter.Mark(1)
   		}
   		pool.all.Add(tx)
   		pool.priced.Put(tx)
   		pool.journalTx(from, tx)
   		pool.queueTxEvent(tx)
   		log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())
   		return old != nil, nil
   	}
   	// New transaction isn't replacing a pending one, push into queue
   	replaced, err = pool.enqueueTx(hash, tx)
```

   添加交易的流程就到此为止了。接下来就是如何把`queue`（暂时不可执行）中添加的交易扔到`pending`（可执行交易）中，速成`promote`。

3. 提升交易

   提升交易主要把交易从`queue`扔到`pending`中，我们在接下来的里面重点讲

   ```go
   done := pool.requestPromoteExecutables(dirtyAddrs)
   ```

## 交易升级

`promoteExecutables`将`future queue`中的交易移动到`pending`中，同时也会删除很多无效交易比如`nonce`低或者余额低等等，主要分以下步骤：先看张图：

![image-20201225104612253](https://tva1.sinaimg.cn/large/0081Kckwgy1glzxix54vaj313m0si4d2.jpg)

①：将所有`queue`中`nonce`低于账户当前`nonce`的交易从`all`里面删除

```go
forwards := list.Forward(pool.currentState.GetNonce(addr))
		for _, tx := range forwards {
			hash := tx.Hash()
			pool.all.Remove(hash)
			log.Trace("Removed old queued transaction", "hash", hash)
		}
```

②：将所有`queue`中花费大于账户余额 或者`gas`大于限制的交易从all里面删除

```go
drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			pool.all.Remove(hash)
			log.Trace("Removed unpayable queued transaction", "hash", hash)
		}
```

③：将所有可执行的交易从`queue`里面移到`pending`里面（`proteTx`）

注：可执行交易：将`pending`里面`nonce`值大于等于账户当前状态`nonce`的，且`nonce`连续的几笔交易作为准备好的交易

```go
readies := list.Ready(pool.pendingNonces.get(addr))
		for _, tx := range readies {
			hash := tx.Hash()
			if pool.promoteTx(addr, hash, tx) {
				log.Trace("Promoting queued transaction", "hash", hash)
				promoted = append(promoted, tx)
			}
		}
```

重点就是 **promoteTx**的处理，这个方法与add的不同之处在于，`addTx`是获得到的**新交易插入pending**，而`promoteTx`是将**queue列表中的Txs放入pending**接下来我们先看看里面是如何来处理的：

```go
inserted, old := list.Add(tx, pool.config.PriceBump)
	if !inserted {
		// An older transaction was better, discard this
		// 老的交易更好，删除这个交易
		pool.all.Remove(hash)
		pool.priced.Removed(1)

		pendingDiscardMeter.Mark(1)
		return false
	}
	// Otherwise discard any previous transaction and mark this
	// 现在这个交易更好，删除旧的交易
	if old != nil {
		pool.all.Remove(old.Hash())
		pool.priced.Removed(1)

		pendingReplaceMeter.Mark(1)
	} else {
	...
	}
```

主要就做了这几件事：

1. 将交易插入`pending`中，如果待插入的交易`nonce`在`pending`列表中存在，那么待插入的交易`gas price`大于或等于原交易价值的`110%（`跟`pricebump`设定有关）时，替换原交易
2. 如果新交易替换了某个交易，从`all`列表中删除老交易
3. 最后更新一下`all`列表

经过`proteTx`之后，要扔到`pending`的交易都放在了`promoted []*types.Transaction`中，再回到`promoteExecutables`中，继续下面步骤：

④：如果非本地账户`queue`大于限制（`AccountQueue`），从最后取出`nonce`较大的交易进行`remove`

```GO
if !pool.locals.contains(addr) {
			caps = list.Cap(int(pool.config.AccountQueue))
			for _, tx := range caps {
				hash := tx.Hash()
				pool.all.Remove(hash)
			...
		}
```

⑤：最后如果队列中此账户的交易为空则删除此账户

```go
if list.Empty() {
			delete(pool.queue, addr)
		}
```

到此我们的升级交易要做的事情就完毕了。

------

## 交易降级

交易降级的几个场景：

1. 出现了新的区块，将会从`pending`中移除出现在区块中的交易到`queue`中
2. 或者是另外一笔交易（`gas price` 更高）,则会从`pending`中移除到`queue`中

关键函数：demoteUnexecutables，主要做的事情如下：

①：遍历`pending`中所有地址对应的交易列表

```go
for addr, list := range pool.pending {
  ...}
```

②：删除所有认为过旧的交易（`low nonce`）

```go
olds := list.Forward(nonce)
		for _, tx := range olds {
			hash := tx.Hash()
			pool.all.Remove(hash)
			log.Trace("Removed old pending transaction", "hash", hash)
		}
```

③：删除所有费用过高的交易（余额低或用尽），并将所有无效者送到`queue`中以备后用

```go
drops, invalids := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			log.Trace("Removed unpayable pending transaction", "hash", hash)
			pool.all.Remove(hash)
		}
		pool.priced.Removed(len(olds) + len(drops))
		pendingNofundsMeter.Mark(int64(len(drops)))

		for _, tx := range invalids {
			hash := tx.Hash()
			log.Trace("Demoting pending transaction", "hash", hash)
			pool.enqueueTx(hash, tx)
		}
```

④：如果交易前面有间隙，将后面的交易移到`queue`中

```go
if list.Len() > 0 && list.txs.Get(nonce) == nil {
			gapped := list.Cap(0)
			for _, tx := range gapped {
				hash := tx.Hash()
				log.Error("Demoting invalidated transaction", "hash", hash)
				pool.enqueueTx(hash, tx)
			}
			pendingGauge.Dec(int64(len(gapped)))
		}
```

注：间隙的出现通常是因为交易余额问题导致的。假如原规范链 A 上交易m花费10，分叉后该账户又在分叉链B发出一个交易m花费20，这就导致该账户余额本来可以支付A链上的某笔交易，但在B链上可能就不够了。这个余额不足的交易在B如果是n+3，那么在A链上n+2，n+4号交易之间就出现了空隙，这就导致从n+3开始往后所有的交易都要降级；

到此为止交易降级结束。

-----

## 重置交易池

-------

**重置交易池**将检索区块链的当前状态（主要由于更新导致链状态变化），并确保交易池的内容对于链状态而言是有效的。

`reset`的调用时机如下：

1. `TxPool`初始化的过程：`NewTxPool`；
2. `TxPool`事件监听`go`程收到规范链更新事件

流程图如下：

![image-20201015185551752](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjq7vc6bz8j31260sodlq.jpg)

根据上面流程图，主要功能是由于规范链的更新，重新整理交易池：

①：*如果老区块头不为空 且老区块头不是新区块的父区块，说明新老区块不在一条链上*

```go
if oldHead != nil && oldHead.Hash() != newHead.ParentHash {}
```

②：*如果新头区块和旧头区块相差大于64，则所有交易不必回退到交易池*

```go
if depth := uint64(math.Abs(float64(oldNum) - float64(newNum))); depth > 64 {
  log.Debug("Skipping deep transaction reorg", "depth", depth)
}
```

③：*如果旧链的头区块大于新链的头区块高度，旧链向后退并回收所有回退的交易*

```go
for rem.NumberU64() > add.NumberU64() {
				discarded = append(discarded, rem.Transactions()...)
				if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
					log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
					return
				}
			}
```

④：*如果新链的头区块大于旧链的头区块，新链后退并回收交易*

```go
for add.NumberU64() > rem.NumberU64() {
				included = append(included, add.Transactions()...)
				if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
					log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
					return
				}
			}
```

⑤：*当新旧链到达同一高度的时候同时回退，知道找到共同的父节点*

```go
for rem.Hash() != add.Hash() {
				discarded = append(discarded, rem.Transactions()...)
				if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
					log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
					return
				}
				included = append(included, add.Transactions()...)
				if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
					log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
					return
				}
			} 
```

⑥：*给交易池设置最新的世界状态*

```go
statedb, err := pool.chain.StateAt(newHead.Root)
	if err != nil {
		log.Error("Failed to reset txpool state", "err", err)
		return
	}
	pool.currentState = statedb
	pool.pendingNonces = newTxNoncer(statedb)
	pool.currentMaxGas = newHead.GasLimit
```

⑦：*把旧链回退的交易放入交易池*

```go
senderCacher.recover(pool.signer, reinject)
pool.addTxsLocked(reinject, false)
```

到此整个`reset`的流程就结束了。

-----

> 参考：
>
> https://mindcarver.cn/
>
> https://github.com/mindcarver/blockchain_guide 
>
> https://learnblockchain.cn/2019/06/03/eth-txpool/#%E6%B8%85%E7%90%86%E4%BA%A4%E6%98%93%E6%B1%A0
>
> https://blog.csdn.net/lj900911/article/details/84825739

