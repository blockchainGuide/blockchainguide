> 以太坊clique算法
>
> 代码分支：https://github.com/ethereum/go-ethereum/tree/v1.9.9

![bca914bd89fa6439facc5a61b6e6d0a1](https://tva1.sinaimg.cn/large/008eGmZEgy1gomsueveelj31ao0t6wor.jpg)

## clique

以太坊的官方共识算法是ethash算法，这在前文已经有了详细的分析：

> 它是基于POW的共识机制的，矿工需要通过计算nonce值，会消耗大量算力来匹配target值。

如果在联盟链或者私链的方案里，继续使用ethash就会浪费算力，POW也没有存在的意义。所以以太坊有了另一种共识方案：基于POA的clique。

> POA, Proof of Authority。权力证明，不同于POW的工作量证明，POA是能够直接确定几个节点具备出块的权力，这几个节点出的块会被全网其他节点验证为有效块。

[建立私链](https://github.com/evsward/blog/blob/master/posts/以太坊-私有链搭建初步实践.md)

通过这篇文章的操作可以建立一个私有链，观察这个流程可以看到，通过puppeth工具建立创世块时，会提示你选择哪种共识方式，有ethash和clique两个选项，说到这里我们就明白了为什么文章中默认要选择clique。

## 源码分析

讲过了基本概念，下面我们深入以太坊源码来仔细分析clique算法的具体实现。

入口仍然选择[seal方法](http://www.cnblogs.com/Evsward/p/miner.html#uncles叔块的概念)，这里与前文分析ethash算法的入口是保持一致的，因为他们是Seal的不同实现。

```
// 我们的注释可以对比着来看，clique的seal函数的目的是：尝试通过本地签名认证（权力签名与认证，找到有权力的结点）来创建一个已密封的区块。
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	header := block.Header()

	number := header.Number.Uint64()
	if number == 0 {// 不允许密封创世块
		return nil, errUnknownBlock
	}
	// 跳转到下方Clique对象的分析。不支持0-period的链，同时拒绝密封空块，没有奖励但是能够旋转密封
	if c.config.Period == 0 && len(block.Transactions()) == 0 {
		return nil, errWaitTransactions
	}
	// 在整个密封区块的过程中，不要持有signer签名者字段。
	c.lock.RLock() // 上锁获取config中的签名者和签名方法。
	signer, signFn := c.signer, c.signFn
	c.lock.RUnlock()

	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)// snapshot函数见下方分析
    // 校验处理：如果我们未经授权去签名了一个区块
	if err != nil {
		return nil, err
	}
	if _, authorized := snap.Signers[signer]; !authorized {
		return nil, errUnauthorized
	}
	// 如果我们是【最近签名者】的一员，则等待下一个区块，// 见下方[底层机制三](http://www.cnblogs.com/Evsward/p/clique.html#%E4%B8%89%E8%AE%A4%E8%AF%81%E7%BB%93%E7%82%B9%E7%9A%84%E5%87%BA%E5%9D%97%E6%9C%BA%E4%BC%9A%E5%9D%87%E7%AD%89)
	for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer当前签名者在【最近签名者】中，如果当前区块没有剔除他的话只能继续等待。
			if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
				log.Info("Signed recently, must wait for others")
				<-stop
				return nil, nil
			}
		}
	}
	// 通过以上校验，到了这里说明协议已经允许我们来签名这个区块，等待此工作完成
	delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now())
	if header.Difficulty.Cmp(diffNoTurn) == 0 {
		// 这不是我们的轮次来签名，delay
		wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime // wiggleTime = 500 * time.Millisecond // 随机推延，从而允许并发签名（针对每个签名者）
		delay += time.Duration(rand.Int63n(int64(wiggle)))

		log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
	}
	log.Trace("Waiting for slot to sign and propagate", "delay", common.PrettyDuration(delay))

	select {
	case <-stop:
		return nil, nil
	case <-time.After(delay):
	}
	// 核心工作：开始签名
	sighash, err := signFn(accounts.Account{Address: signer}, sigHash(header).Bytes())// signFn函数见下方
	if err != nil {
		return nil, err
	}
	copy(header.Extra[len(header.Extra)-extraSeal:], sighash)//将签名结果替换区块头的Extra字段（专门支持记录额外信息的）

	return block.WithSeal(header), nil //通过区块头重新组装一个区块
}
```

Clique对象的分析

```
// Clique是POA共识引擎，计划在Ropsten攻击以后，用来支持以太坊私测试链testnet（也可以自己搭建联盟链或者私有链）
type Clique struct {
	config *params.CliqueConfig // 共识引擎配置参数，见下方CliqueConfig源码介绍
	db     ethdb.Database       // 数据库，用来存储以及获取快照检查点

	recents    *lru.ARCCache // 最近区块的快照，用来加速快照重组
	signatures *lru.ARCCache // 最近区块的签名，用来加速挖矿

	proposals map[common.Address]bool // 目前我们正在推动的提案清单，存的是地址和布尔值的键值对映射

	signer common.Address // 签名者的以太坊地址
	signFn SignerFn       // 签名方法，用来授权哈希
	lock   sync.RWMutex   // 锁，保护签名字段
}
```

CliqueConfig源码分析

```
// CliqueConfig是POA挖矿的共识引擎的配置字段。
type CliqueConfig struct {
	Period uint64 `json:"period"` // 在区块之间执行的秒数(可以理解为距离上一块出块后的流逝时间秒数)
	Epoch  uint64 `json:"epoch"`  // Epoch['iːpɒk]长度，重置投票和检查点
}
```

snapshot函数分析

```
// snapshot函数可通过给定点获取认证快照
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
	// 在内存或磁盘上搜索一个快照以检查检查点。
	var (
		headers []*types.Header// 区块头
		snap    *Snapshot// 快照对象，见下方
	)
	for snap == nil {
		// 如果找到一个内存里的快照，使用以下方案：
		if s, ok := c.recents.Get(hash); ok {
			snap = s.(*Snapshot)
			break
		}
		// 如果一个在磁盘检查点的快照被找到，使用以下方案：
		if number%checkpointInterval == 0 {// checkpointInterval = 1024 // 区块号，在数据库中保存投票快照的区块。
			if s, err := loadSnapshot(c.config, c.signatures, c.db, hash); err == nil {// loadSnapshot函数见下方
				log.Trace("Loaded voting snapshot form disk", "number", number, "hash", hash)
				snap = s
				break
			}
		}
		// 如果我们在创世块，则做一个快照
		if number == 0 {
			genesis := chain.GetHeaderByNumber(0)
			if err := c.VerifyHeader(chain, genesis, false); err != nil {
				return nil, err
			}
			signers := make([]common.Address, (len(genesis.Extra)-extraVanity-extraSeal)/common.AddressLength)
			for i := 0; i < len(signers); i++ {
				copy(signers[i][:], genesis.Extra[extraVanity+i*common.AddressLength:])
			}
			snap = newSnapshot(c.config, c.signatures, 0, genesis.Hash(), signers)// 创建一个新的快照的函数，见下方
			if err := snap.store(c.db); err != nil {
				return nil, err
			}
			log.Trace("Stored genesis voting snapshot to disk")
			break
		}
		// 没有针对这个区块头的快照，则收集区块头并向后移动
		var header *types.Header
		if len(parents) > 0 {
			// 如果我们有明确的父类，从这里强制挑拣出来。
			header = parents[len(parents)-1]
			if header.Hash() != hash || header.Number.Uint64() != number {
				return nil, consensus.ErrUnknownAncestor
			}
			parents = parents[:len(parents)-1]
		} else {
			// 如果没有明确父类（或者没有更多的），则转到数据库
			header = chain.GetHeader(hash, number)
			if header == nil {
				return nil, consensus.ErrUnknownAncestor
			}
		}
		headers = append(headers, header)
		number, hash = number-1, header.ParentHash
	}
	// 找到了先前的快照，那么将所有pending的区块头都放在它的上面。
	for i := 0; i < len(headers)/2; i++ {
		headers[i], headers[len(headers)-1-i] = headers[len(headers)-1-i], headers[i]
	}
	snap, err := snap.apply(headers)//通过区块头生成一个新的snapshot对象
	if err != nil {
		return nil, err
	}
	c.recents.Add(snap.Hash, snap)//将当前快照区块的hash存到recents中。

	// 如果我们生成了一个新的检查点快照，保存到磁盘上。
	if snap.Number%checkpointInterval == 0 && len(headers) > 0 {
		if err = snap.store(c.db); err != nil {
			return nil, err
		}
		log.Trace("Stored voting snapshot to disk", "number", snap.Number, "hash", snap.Hash)
	}
	return snap, err
}
```

Snapshot对象源码分析：

```
// Snapshot对象是在给定点的一个认证投票的状态
type Snapshot struct {
	config   *params.CliqueConfig // 配置参数
	sigcache *lru.ARCCache        // 签名缓存，最近的区块签名加速恢复。

	Number  uint64                      `json:"number"`  // 快照建立的区块号
	Hash    common.Hash                 `json:"hash"`    // 快照建立的区块哈希
	Signers map[common.Address]struct{} `json:"signers"` // 当下认证签名者的集合
	Recents map[uint64]common.Address   `json:"recents"` // 最近签名区块地址的集合
	Votes   []*Vote                     `json:"votes"`   // 按时间顺序排列的投票名单。
	Tally   map[common.Address]Tally    `json:"tally"`   // 当前的投票结果，避免重新计算。
}
```

loadSnapshot函数源码分析：

```
// loadSnapshot函数用来从数据库中加载一个现存的快照，参数列表中很多都是Snapshot对象的关键字段属性。
func loadSnapshot(config *params.CliqueConfig, sigcache *lru.ARCCache, db ethdb.Database, hash common.Hash) (*Snapshot, error) {
	blob, err := db.Get(append([]byte("clique-"), hash[:]...))// ethdb使用的是leveldb，对外开放接口Dababase见下方
	if err != nil {
		return nil, err
	}
	snap := new(Snapshot)
	if err := json.Unmarshal(blob, snap); err != nil {
		return nil, err
	}
	snap.config = config
	snap.sigcache = sigcache

	return snap, nil
}
```

ethdb数据库对外开放接口：

```
// Database接口包裹了所有的数据库相关操作，所有的方法都是线程安全的。
type Database interface {
	Putter
	Get(key []byte) ([]byte, error)//通过某key获取值
	Has(key []byte) (bool, error)//某key是否包含有效值
	Delete(key []byte) error
	Close()
	NewBatch() Batch
}
```

newSnapshot函数源码：

```
// newSnapshot函数创建了一个新的快照，通过给出的特定的启动参数。这个方法没有初始化最近签名者的集合，所以只有使用创世块。
func newSnapshot(config *params.CliqueConfig, sigcache *lru.ARCCache, number uint64, hash common.Hash, signers []common.Address) *Snapshot {
	snap := &Snapshot{// 就是组装一个Snapshot对象，安装相应参数
		config:   config,
		sigcache: sigcache,
		Number:   number,
		Hash:     hash,
		Signers:  make(map[common.Address]struct{}),
		Recents:  make(map[uint64]common.Address),
		Tally:    make(map[common.Address]Tally),
	}
	for _, signer := range signers {
		snap.Signers[signer] = struct{}{}
	}
	return snap
}
```

signFn函数：

```
// SignerFn是一个签名者的回调函数，用来请求一个能够被后台账户签名生成的哈希
type SignerFn func(accounts.Account, []byte) ([]byte, error)
```

clique常量配置：

```
blockPeriod = uint64(15)    // clique规定，两个区块的生成时间至少间隔15秒，timestamp类型。
```

## Clique底层机制

在进入共识引擎之前，当前结点已经生成了一个完整的区块，包括区块头和密封的交易列表，然后进入seal函数，通过ethash或者clique算法引擎来操作出块确权。本文重点讲述了针对clique算法的源码分析，clique算法基于POA共识，是在结点中找出有权力的几个“超级结点”，只有这些结点可以生成合法区块，其他结点的出块都会直接丢弃。

### 一：clique是如何确定签名者以及签名方法的？

我在clique文件中搜索，发现有一个方法做了这个工作：

```
// Authorize函数注入共识引擎clique一个私钥地址（签名者）以及签名方法signFn，用来挖矿新块
func (c *Clique) Authorize(signer common.Address, signFn SignerFn) {
	c.lock.Lock()
	defer c.lock.Unlock()

	c.signer = signer
	c.signFn = signFn
}
```

那么继续搜索，该函数是在何时被调用的，找到了位于/eth/backend.go中的函数StartMining：

```
func (s *Ethereum) StartMining(local bool) error {
	eb, err := s.Etherbase()// 用户地址
	if err != nil {
		log.Error("Cannot start mining without etherbase", "err", err)//未找到以太账户地址，报错
		return fmt.Errorf("etherbase missing: %v", err)
	}
	// 如果是clique共识算法，则走if分支，如果是ethash则跳过if。
	if clique, ok := s.engine.(*clique.Clique); ok {// Comma-ok断言语法见下方分析。
		wallet, err := s.accountManager.Find(accounts.Account{Address: eb})// 通过用户地址获得wallet对象
		if wallet == nil || err != nil {
			log.Error("Etherbase account unavailable locally", "err", err)
			return fmt.Errorf("signer missing: %v", err)
		}
		clique.Authorize(eb, wallet.SignHash)//在这里！注入了签名者以及通过wallet对象获取到签名方法
	}
	if local {
		// 如果本地CPU挖矿已启动，我们可以禁止注入机制以加速同步时间。
		// CPU挖矿在主网是荒诞的，所以没有人能碰到这个路径，然而一旦CPU挖矿同步标志完成以后，将保证私网工作也在一个独立矿工结点。
		atomic.StoreUint32(&s.protocolManager.acceptTxs, 1)
	}
	go s.miner.Start(eb)//并发启动挖矿工作
	return nil
}
```

最终，通过miner.Start(eb)，调用到work -> agent -> CPUAgent -> update -> seal，回到最上方我们的入口。

> 这里要补充一点，[挖矿机制](http://www.cnblogs.com/Evsward/p/miner.html)是从miner.start()作为入口开始分析的，而上面的StartMining函数是在miner.start()之前的。这样就把整个这一条线串起来了。

### Go语法补充：Comma-ok断言

> if clique, ok := s.engine.(*clique.Clique); ok {

这段语句很令人迷惑，经过搜查，以上语法被称作Comma-ok断言。

```
value, ok = element.(T)
```

value是element变量的值，ok是布尔类型用来表达断言结果，element是接口变量，T是断言类型。

套入以上代码段，翻译过来即：

> 如果s.engine是Clique类型，则ok为true，同时clique就等于s.engine。

### 二：Snapshot起到的作用是什么？

Snapshot对象在Seal方法中是通过调用snapshot构造函数来获取到的。而snapshot构造函数内部有较长的函数体，包括newSnapshot方法以及loadSnapshot方法的处理。从这个分析来看，我们也可以知道Snapshot是快照，也是缓存的一种机制，同时它也不仅仅是缓存，因为它存储了最近签名者的map集合。

> Snapshot可以从内存（即程序中的变量）或是磁盘上（即通过数据库leveldb）获取或者存储，实际上这就是二级缓存的概念了。

### 三：认证结点的出块机会均等

首先将上文Seal方法的源码遗留代码段展示如下。

```
for seen, recent := range snap.Recents {
	if recent == signer {
		if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
			log.Info("Signed recently, must wait for others")
			<-stop
			return nil, nil
		}
	}
}
```

其中

```
if recent == signer {
```

如果当前结点最近签名过，则跳过，为保证**机会均等**，避免某个认证结点可以连续出块，从而作恶。

```
if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
```

实际上到了这里就已经在决定出块权了。我们依次来看，

- snap.Signers是所有的认证结点。
- limit的值是所有认证结点的数量的一半加1，也就是说可以保证limit>50%好的认证结点个数（安全性考虑：掌握大于50%的控制权）。**结合上面的机会均等，clique要求认证结点在每轮limit个区块中只能生成一个区块。**
- number是当前区块号
- seen是 “for seen, recent := range snap.Recents {” 中Recents的index，从0开始，最大值为Recents的总数-1。

接着，我们来分析控制程序中止的条件表达式：

```
number < limit || seen > number-limit
```

- number < limit, 如果区块高度小于limit
- seen > number - limit，缓存中最近签发者序号已经超过了区块高度与limit之差。number-limit是最多的坏节点，索引seen大于坏节点也要中断（**TODO: number区块高度与认证结点的关系**）

在这两种情况下，会中断程序，停止签名以及出块操作。

### 四：出块难度

```
// inturn函数通过给定的区块高度和签发者返回该签发者是否在轮次内
func (s *Snapshot) inturn(number uint64, signer common.Address) bool {
    // 方法体的内容就是区块高度与认证签发者集合长度的余数是否等于该签发者的下标值
	signers, offset := s.signers(), 0
	for offset < len(signers) && signers[offset] != signer {
		offset++
	}
	return (number % uint64(len(signers))) == uint64(offset)
}
```

> 一句话，clique要求签发者必须按照其在snapshot中的认证签发者集合按照字典排序的顺序出块。

符合以上条件的话，难度为2，否则为1。

```
diffInTurn = big.NewInt(2) // 签名在轮次内的区块难度为2。
diffNoTurn = big.NewInt(1) // 签名未在轮次内的区块难度为1。
```

clique的出块难度比较容易理解，这是在POW中大书特书的部分但在clique中却十分简单，当inturn的结点离线时，其他结点会来竞争，难度值降为1。然而正常出块时，limit中的所有认证结点包括一个inturn和其他noturn的结点，clique是采用了给noturn加延迟时间的方式来支持inturn首先出块，避免noturn的结点无谓生成区块。这部分代码在下面再贴一次。

```
wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime // wiggleTime = 500 * time.Millisecond // 随机推延，从而允许并发签名（针对每个签名者）
delay += time.Duration(rand.Int63n(int64(wiggle)))
```

> clique认可难度值最高的链为主链，所以完全inturn结点出的块组成的链会是最理想的主链。

### 五：区块校验

```
// 同样位于clique文件中的verifySeal函数，顾名思义是结点用来校验别的结点广播过来的区块信息的。
func (c *Clique) verifySeal(chain consensus.ChainReader, header *types.Header, parents []*types.Header) error {
	// 创世块的话不校验
	number := header.Number.Uint64()
	if number == 0 {
		return errUnknownBlock
	}
	// 取到所需snapshot对象，用来校验区块头并且将其缓存。
	snap, err := c.snapshot(chain, number-1, header.ParentHash, parents)
	if err != nil {
		return err
	}

	// 处理授权秘钥，检查是否违背认证签名者集合
	signer, err := ecrecover(header, c.signatures)// 从区块头中解密出Extra字段，找到签名字符串，获得签名者地址信息。可以跳转到下面ecrecover函数的源码分析。
	if err != nil {
		return err
	}
	if _, ok := snap.Signers[signer]; !ok {
		return errUnauthorized
	}
    // 与Seal相同的处理，机会均等
	for seen, recent := range snap.Recents {
		if recent == signer {
			if limit := uint64(len(snap.Signers)/2 + 1); seen > number-limit {
				return errUnauthorized
			}
		}
	}
	// 区分是否inturn，设置区块困难度，上面也介绍过了。
	inturn := snap.inturn(header.Number.Uint64(), signer)
	if inturn && header.Difficulty.Cmp(diffInTurn) != 0 {
		return errInvalidDifficulty
	}
	if !inturn && header.Difficulty.Cmp(diffNoTurn) != 0 {
		return errInvalidDifficulty
	}
	return nil
}
```

ecrecover函数的源码分析：

```
// ecrecover函数从一个签名的区块头中解压出以太坊账户地址
func ecrecover(header *types.Header, sigcache *lru.ARCCache) (common.Address, error) {
	// 如果签名已经被缓存，返回它。
	hash := header.Hash()
	if address, known := sigcache.Get(hash); known {
		return address.(common.Address), nil
	}
	// 从区块头的Extra字段取得签名内容。
	if len(header.Extra) < extraSeal {
		return common.Address{}, errMissingSignature
	}
	signature := header.Extra[len(header.Extra)-extraSeal:]

	// 通过密码学技术从签名内容中解密出公钥和以太坊地址。
	pubkey, err := crypto.Ecrecover(sigHash(header).Bytes(), signature)// 具体源码见下方
	if err != nil {
		return common.Address{}, err
	}
	var signer common.Address
	copy(signer[:], crypto.Keccak256(pubkey[1:])[12:])//将公钥利用keccak256解密赋值给signer。

	sigcache.Add(hash, signer)//加入缓存
	return signer, nil
}
```

crypto包的Ecrecover函数：

```
func Ecrecover(hash, sig []byte) ([]byte, error) {
	return secp256k1.RecoverPubkey(hash, sig)
}
```

Ecrecover函数是使用secp256k1来解密公钥。

下面我们从VerifySeal函数反推，找出调用该函数的位置在miner/remote_agent.go，

```
// SubmitWork函数尝试注入一个pow解决方案（共识引擎）到远程代理，返回这个解决方案是否被接受。（不能同时是一个坏的pow也不能有其他任何错误，例如没有工作被pending
func (a *RemoteAgent) SubmitWork(nonce types.BlockNonce, mixDigest, hash common.Hash) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	// 保证被提交的工作不是空
	work := a.work[hash]
	if work == nil {
		log.Info("Work submitted but none pending", "hash", hash)
		return false
	}
	// 保证引擎是真实有效的。
	result := work.Block.Header()
	result.Nonce = nonce
	result.MixDigest = mixDigest

	if err := a.engine.VerifySeal(a.chain, result); err != nil {//在这里，VerifySeal方法被调用。
		log.Warn("Invalid proof-of-work submitted", "hash", hash, "err", err)
		return false
	}
	block := work.Block.WithSeal(result)

	// 解决方案看上去是有效的，返回到矿工并且通知接受结果。
	a.returnCh <- &Result{work, block}
	delete(a.work, hash)

	return true
}
```

这个SubmitWork位于挖矿的pkg中，主要工作是对work的校验，包括work本身是否为空，work中的区块头以及区块头中包含的字段的有效性，然后是对区块头的VerifySeal（该函数的功能在上面已经介绍到了，主要是对区块签名者的认证，区块难度值的确认）

继续反推找到SubmitWork函数被调用的位置：

```
// SubmitWork函数能够被外部矿工用来提交他们的POW。
func (api *PublicMinerAPI) SubmitWork(nonce types.BlockNonce, solution, digest common.Hash) bool {
	return api.agent.SubmitWork(nonce, digest, solution)
}
```

**总结**

区块的校验是外部结点自动执行PublicMinerAPI的SubmitWork方法，从而层层调用，通过检查区块头内的签名内容，通过secp256k1方法恢复公钥，然后利用Keccak256将公钥加密为一个以太地址作为签名地址，获得签名地址以后，去本地认证结点缓存中检查，看该签名地址是否符合要求。最终只要通过层层校验，就不会报出errUnauthorized的错误。

> 注意：签名者地址common.Address在Seal时被签名signature存在区块头的Extra字段中，然后在VerifySeal中被从区块头中取出签名signature。该签名的解密方式比较复杂：要先通过secp256k1恢复一个公钥，然后利用这个公钥和Keccak256加密出签名者地址common.Address。

common.Address本身就是结点公钥的Keccak256加密结果。请参照common/types.go：

```
// Hex函数返回了一个十六禁止的字符串，代表了以太坊地址。
func (a Address) Hex() string {
	unchecksummed := hex.EncodeToString(a[:])
	sha := sha3.NewKeccak256()//这里就不展开了，可以看出是通过Keccak256方法将未检查的明文Address加密为一个标准以太坊地址
	sha.Write([]byte(unchecksummed))
	hash := sha.Sum(nil)

	result := []byte(unchecksummed)
	for i := 0; i < len(result); i++ {
		hashByte := hash[i/2]
		if i%2 == 0 {
			hashByte = hashByte >> 4
		} else {
			hashByte &= 0xf
		}
		if result[i] > '9' && hashByte > 7 {
			result[i] -= 32
		}
	}
	return "0x" + string(result)
}
```

### 六： 基于投票的认证结点的运行机制

上面我们分析了clique的认证结点的出块，校验等细节，那么这里引出终极问题：如何确认一个普通结点是否是认证结点呢？

> 答：clique是基于投票机制来确认认证结点的。

先来看投票实体类，存在于snapshot源码中。

```
// Vote代表了一个独立的投票，这个投票可以授权一个签名者，更改授权列表。
type Vote struct {
	Signer    common.Address `json:"signer"`    // 已授权的签名者（通过投票）
	Block     uint64         `json:"block"`     // 投票区块号
	Address   common.Address `json:"address"`   // 被投票的账户，修改它的授权
	Authorize bool           `json:"authorize"` // 对一个被投票账户是否授权或解授权
}
```

这个Vote是存在于Snapshot的属性字段中，所以投票机制离不开Snapshot，我们在这里再次将Snapshot实体源码重新分析一遍，上面注释过的内容我不再复述，而是直接关注在投票机制相关字段内容上。

```
type Snapshot struct {
	config   *params.CliqueConfig
	sigcache *lru.ARCCache       

	Number  uint64                      `json:"number"`  
	Hash    common.Hash                 `json:"hash"`    
	Signers map[common.Address]struct{} `json:"signers"` // 认证节点集合
	Recents map[uint64]common.Address   `json:"recents"` 
	Votes   []*Vote                     `json:"votes"`   // 上面的Vote对象数组
	Tally   map[common.Address]Tally    `json:"tally"`   // 也是一个自定义类型，见下方
}
```

Tally结构体：

```
// Tally是一个简单的用来保存当前投票分数的计分器
type Tally struct {
	Authorize bool `json:"authorize"` // 授权true或移除false
	Votes     int  `json:"votes"`     // 该提案已获票数
}
```

另外Clique实体中还有个有争议的字段proposals，当时并没有分析清楚，何谓提案？

> proposal是可以通过rpc申请加入或移除一个认证节点，结构为待操作地址（节点地址）和状态（加入或移除）

#### 投票中某些概念的确定

- 投票的范围是在委员会，委员会的意思就是所有矿工。
- 概念介绍：checkpoint，checkpointInterval = 1024 ，每过1024个区块，则保存snapshot到数据库
- 概念介绍：Epoch，与ethash一样，一个Epoch是三万个区块

#### 投票流程

- 首先委员会某个成员（即节点矿工）通过rpc调用consensus/clique/api.go中的propose方法

```
// Propose注入一个新的授权提案，可以授权一个签名者或者移除一个。
func (api *API) Propose(address common.Address, auth bool) {
	api.clique.lock.Lock()
	defer api.clique.lock.Unlock()

	api.clique.proposals[address] = auth// true:授权，false:移除
}
```

- 上面rpc提交过来的propose会写入Clique.proposals集合中。
- 在挖矿开始以后，会在miner.start()中提交一个commitNewWork，其中涉及到准备区块头Prepare的方法，我们进入到clique的实现，其中涉及到对上面的Clique.proposals的处理：

```
// 如果存在pending的proposals，则投票
if len(addresses) > 0 {
	header.Coinbase = addresses[rand.Intn(len(addresses))]//将投票节点的地址赋值给区块头的Coinbase字段。
	// 下面是通过提案内容来组装区块头的随机数字段。
	if c.proposals[header.Coinbase] {
		copy(header.Nonce[:], nonceAuthVote)
	} else {
		copy(header.Nonce[:], nonceDropVote)
	}
}

// nonceAuthVote和nonceDropVote常量的声明与初始化
nonceAuthVote = hexutil.MustDecode("0xffffffffffffffff") // 授权签名者的必要随机数
nonceDropVote = hexutil.MustDecode("0x0000000000000000") // 移除签名者的必要随机数
```

- 整个区块组装好以后（其他的内容不再复述），会被广播到外部结点校验，如果没有问题该块被成功出了，则区块头中的这个提案也会被记录在主链上。
- 区块在生成时，会创建Snapshot，在snapshot构造函数中，会涉及到对proposal的处理apply方法。

```
// apply通过接受一个给定区块头创建了一个新的授权
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
	if len(headers) == 0 {
		return s, nil
	}
	for i := 0; i < len(headers)-1; i++ {
		if headers[i+1].Number.Uint64() != headers[i].Number.Uint64()+1 {
			return nil, errInvalidVotingChain
		}
	}
	if headers[0].Number.Uint64() != s.Number+1 {
		return nil, errInvalidVotingChain
	}
	snap := s.copy()
    // 投票的处理核心代码
	for _, header := range headers {
		// Remove any votes on checkpoint blocks
		number := header.Number.Uint64()
		// 如果区块高度正好在Epoch结束，则清空投票和计分器
		if number%s.config.Epoch == 0 {
			snap.Votes = nil
			snap.Tally = make(map[common.Address]Tally)
		}
		if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
			delete(snap.Recents, number-limit)
		}
		// 从区块头中解密出来签名者地址
		signer, err := ecrecover(header, s.sigcache)
		if err != nil {
			return nil, err
		}
		if _, ok := snap.Signers[signer]; !ok {
			return nil, errUnauthorized
		}
		for _, recent := range snap.Recents {
			if recent == signer {
				return nil, errUnauthorized
			}
		}
		snap.Recents[number] = signer

		// 区块头认证，不管该签名者之前的任何投票
		for i, vote := range snap.Votes {
			if vote.Signer == signer && vote.Address == header.Coinbase {
				// 从缓存计数器中移除该投票
				snap.uncast(vote.Address, vote.Authorize)

				// 从按时间排序的列表中移除投票
				snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
				break // only one vote allowed
			}
		}
		// 从签名者中计数新的投票
		var authorize bool
		switch {
		case bytes.Equal(header.Nonce[:], nonceAuthVote):
			authorize = true
		case bytes.Equal(header.Nonce[:], nonceDropVote):
			authorize = false
		default:
			return nil, errInvalidVote
		}
		if snap.cast(header.Coinbase, authorize) {
			snap.Votes = append(snap.Votes, &Vote{
				Signer:    signer,
				Block:     number,
				Address:   header.Coinbase,
				Authorize: authorize,
			})
		}
		// 判断票数是否超过一半的投票者，如果投票通过，更新签名者列表
		if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
			if tally.Authorize {
				snap.Signers[header.Coinbase] = struct{}{}
			} else {
				delete(snap.Signers, header.Coinbase)

				if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
					delete(snap.Recents, number-limit)
				}
				for i := 0; i < len(snap.Votes); i++ {
					if snap.Votes[i].Signer == header.Coinbase {
						snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)

						snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)

						i--
					}
				}
			}
			// 不管之前的任何投票，直接改变账户
			for i := 0; i < len(snap.Votes); i++ {
				if snap.Votes[i].Address == header.Coinbase {
					snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
					i--
				}
			}
			delete(snap.Tally, header.Coinbase)
		}
	}
	snap.Number += uint64(len(headers))
	snap.Hash = headers[len(headers)-1].Hash()

	return snap, nil
}
```

关键控制的代码是tally.Votes > len(snap.Signers)/2，意思是计分器中的票数大于一半的签名者，就表示该投票通过，下面就是要更改snapshot中的认证签名者列表缓存，同时要同步给其他节点，并删除该投票相关信息。

## 总结

本以为clique比较简单，不必调查这么长，然而POA的共识算法还是比较有难度的，它和POW是基于完全不同的两种场景的实现方式，出块方式也完全不同。下面我尝试用简短的语言来总结Clique的共识机制。

> clique共识是基于委员会选举认证节点来确认出块权力的方式实现的。投票方式通过rpc请求propose，snapshot二级缓存机制，唱票，执行投票结果。认证节点出块机会均等，困难度通过轮次（是否按照缓存认证顺序出块）确定，区块头Extra存储签名，keccak256加密以太地址，secp256k1解密签名为公钥，通过认证结点出块的逻辑可以反推区块校验。

到目前为止，我们对POA共识机制，以及以太坊clique的实现有了深刻的理解与认识，相信如果让我们去实现一套POA，也是完全有能力的。大家在阅读本文时有任何疑问均可留言给我，我一定会及时回复。

----

## 总结&参考

> https://github.com/blockchainGuide 
>
> 公众号：区块链技术栈  （推荐哦）
>
> https://eth.wiki/concepts/ethash/design-rationale
>
> https://eth.wiki/concepts/ethash/dag
>
> https://www.vijaypradeep.com/blog/2017-04-28-ethereums-memory-hardness-explained/