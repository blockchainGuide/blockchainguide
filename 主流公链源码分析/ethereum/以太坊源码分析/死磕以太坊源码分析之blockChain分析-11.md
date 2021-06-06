> 死磕以太坊源码分析之blockChain分析
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/
>
> 写文不易，给个小关注，有什么问题可以指出，便于大家交流学习。

## blockchain关键元素

- `db`：持久化到底层数据储存，即`leveldb`；
- `genesisBlock`：创始区块
- `currentBlock`：当前区块，`blockchain`中并不是储存链所有的`block`，而是通过`currentBlock`向前回溯直到`genesisBlock`，这样就构成了区块链
- `bodyCache`、`bodyRLPCache`、`blockCache`、`futureBlocks`：区块链中的缓存结构，用于加快区块链的读取和构建；
- `hc`：`headerchain`区块头链，由`blockchain`额外维护的另一条链，由于`Header`和`Block`的储存空间是有很大差别的，但同时`Block`的`Hash`值就是`Header`（RLP）的`Hash`值，所以维护一个`headerchain`可以用于快速延长链，验证通过后再下载`blockchain`，或者可以与`blockchain`进行相互验证；
- `processor`：执行区块链交易的接口，收到一个新的区块时，要对区块中的所有交易执行一遍，一方面是验证，一方面是更新世界状态；
- `validator`：验证数据有效性的接口
- `futureBlocks`：收到的区块时间大于当前头区块时间15s而小于30s的区块，可作为当前节点待处理的区块。

----

## 函数介绍

```go
// BadBlocks 处理客户端从网络上获取的最近的bad block列表
func (bc *BlockChain) BadBlocks() []*types.Block {}

// addBadBlock 把bad block放入缓存
func (bc *BlockChain) addBadBlock(block *types.Block) {}
```

```go
// CurrentBlock取回主链的当前头区块，这个区块是从blockchian的内部缓存中取得
func (bc *BlockChain) CurrentBlock() *types.Block {}
 
// CurrentHeader检索规范链的当前头区块header。从HeaderChain的内部缓存中检索标头。
func (bc *BlockChain) CurrentHeader() *types.Header{}
 
// CurrentFastBlock取回主链的当前fast-sync头区块，这个区块是从blockchian的内部缓存中取得
func (bc *BlockChain) CurrentFastBlock() *types.Block {}
```

```go
// 将活动链或其子集写入给定的编写器.
func (bc *BlockChain) Export(w io.Writer) error {}
func (bc *BlockChain) ExportN(w io.Writer, first uint64, last uint64) error {}
```

```go
// FastSyncCommitHead快速同步，将当前头块设置为特定hash的区块。
func (bc *BlockChain) FastSyncCommitHead(hash common.Hash) error {}
```

```go
// GasLimit返回当前头区块的gas limit
func (bc *BlockChain) GasLimit() uint64 {}
```

```go
// Genesis 取回genesis区块
func (bc *BlockChain) Genesis() *types.Block {}
```

```go
// 通过hash从数据库或缓存中取到一个区块体(transactions and uncles)或RLP数据
func (bc *BlockChain) GetBody(hash common.Hash) *types.Body {}
func (bc *BlockChain) GetBodyRLP(hash common.Hash) rlp.RawValue {}
```

```go
// GetBlock 通过hash和number取到区块
func (bc *BlockChain) GetBlock(hash common.Hash, number uint64) *types.Block {}
// GetBlockByHash 通过hash取到区块
func (bc *BlockChain) GetBlockByHash(hash common.Hash) *types.Block {}
// GetBlockByNumber 通过number取到区块
func (bc *BlockChain) GetBlockByNumber(number uint64) *types.Block {}
```

```go
// 获取给定hash和number区块的header
func (bc *BlockChain) GetHeader(hash common.Hash, number uint64) *types.Header{}
 
// 获取给定hash的区块header
func (bc *BlockChain) GetHeaderByHash(hash common.Hash) *types.Header{}
 
// 获取给定number的区块header
func (bc *BlockChain) GetHeaderByNumber(number uint64) *types.Header{}
```

```go
// HasBlock检验hash对应的区块是否完全存在数据库中
func (bc *BlockChain) HasBlock(hash common.Hash, number uint64) bool {}
 
// 检查给定hash和number的区块的区块头是否存在数据库
func (bc *BlockChain) HasHeader(hash common.Hash, number uint64) bool{}
 
// HasState检验state trie是否完全存在数据库中
func (bc *BlockChain) HasState(hash common.Hash) bool {}
 
// HasBlockAndState检验hash对应的block和state trie是否完全存在数据库中
func (bc *BlockChain) HasBlockAndState(hash common.Hash, number uint64) bool {}
```

```go
// 获取给定hash的区块的总难度
func (bc *BlockChain) GetTd(hash common.Hash, number uint64) *big.Int{}
```

```go
// 获取从给定hash的区块到genesis区块的所有hash
func (bc *BlockChain) GetBlockHashesFromHash(hash common.Hash, max uint64) []common.Hash{}
 
// GetReceiptsByHash 在特定的区块中取到所有交易的收据
func (bc *BlockChain) GetReceiptsByHash(hash common.Hash) types.Receipts {}
 
// GetBlocksFromHash 取到特定hash的区块及其n-1个父区块
func (bc *BlockChain) GetBlocksFromHash(hash common.Hash, n int) (blocks []*types.Block) {}
 
// GetUnclesInChain 取回从给定区块到向前回溯特定距离到区块上的所有叔区块
func (bc *BlockChain) GetUnclesInChain(block *types.Block, length int) []*types.Header {}
```

```go
// insert 将新的头块注入当前块链。 该方法假设该块确实是真正的头。
// 如果它们较旧或者它们位于不同的侧链上，它还会将头部标题和头部快速同步块重置为同一个块。
func (bc *BlockChain) insert(block *types.Block) {}
 
// InsertChain尝试将给定批量的block插入到规范链中，否则，创建一个分叉。 如果返回错误，它将返回失败块的索引号以及描述错误的错误。
//插入完成后，将触发所有累积的事件。
func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error){}
 
// insertChain将执行实际的链插入和事件聚合。 
// 此方法作为单独方法存在的唯一原因是使用延迟语句使锁定更清晰。
func (bc *BlockChain) insertChain(chain types.Blocks) (int, []interface{}, []*types.Log, error){}
 
// InsertHeaderChain尝试将给定的headerchain插入到本地链中，可能会创建一个重组
func (bc *BlockChain) InsertHeaderChain(chain []*types.Header, checkFreq int) (int, error){}
 
// InsertReceiptChain 使用交易和收据数据来完成已经存在的headerchain
func (bc *BlockChain) InsertReceiptChain(blockChain types.Blocks, receiptChain []types.Receipts) (int, error) {}
```

```go
//loadLastState从数据库加载最后一个已知的链状态。
func (bc *BlockChain) loadLastState() error {}
```

```go
// Processor 返回当前current processor.
func (bc *BlockChain) Processor() Processor {}
```

```go
// Reset重置清除整个区块链，将其恢复到genesis state.
func (bc *BlockChain) Reset() error {}
 
// ResetWithGenesisBlock 清除整个区块链, 用特定的genesis state重塑，被Reset所引用
func (bc *BlockChain) ResetWithGenesisBlock(genesis *types.Block) error {}
 
// repair尝试通过回滚当前块来修复当前的区块链，直到找到具有关联状态的块。
// 用于修复由崩溃/断电或简单的非提交尝试导致的不完整的数据库写入。
//此方法仅回滚当前块。 当前标头和当前快速块保持不变。
func (bc *BlockChain) repair(head **types.Block) error {}
 
// reorgs需要两个块、一个旧链以及一个新链，并将重新构建块并将它们插入到新的规范链中，并累积潜在的缺失事务并发布有关它们的事件
func (bc *BlockChain) reorg(oldBlock, newBlock *types.Block) error{}
 
// Rollback 旨在从数据库中删除不确定有效的链片段
func (bc *BlockChain) Rollback(chain []common.Hash) {}
```

```go

// SetReceiptsData 计算收据的所有非共识字段
func SetReceiptsData(config *params.ChainConfig, block *types.Block, receipts types.Receipts) error {}
 
// SetHead将本地链回滚到指定的头部。
// 通常可用于处理分叉时重选主链。对于Header，新Header上方的所有内容都将被删除，新的头部将被设置。
// 但如果块体丢失，则会进一步回退（快速同步后的非归档节点）。
func (bc *BlockChain) SetHead(head uint64) error {}
 
// SetProcessor设置状态修改所需要的processor
func (bc *BlockChain) SetProcessor(processor Processor) {}
 
// SetValidator 设置用于验证未来区块的validator
func (bc *BlockChain) SetValidator(validator Validator) {}
 
// State 根据当前头区块返回一个可修改的状态
func (bc *BlockChain) State() (*state.StateDB, error) {}
 
// StateAt 根据特定时间点返回新的可变状态
func (bc *BlockChain) StateAt(root common.Hash) (*state.StateDB, error) {}
 
// Stop 停止区块链服务，如果有正在import的进程，它会使用procInterrupt来取消。
// it will abort them using the procInterrupt.
func (bc *BlockChain) Stop() {}
 
// TrieNode从memory缓存或storage中检索与trie节点hash相关联的数据。
func (bc *BlockChain) TrieNode(hash common.Hash) ([]byte, error) {}
 
// Validator返回当前validator.
func (bc *BlockChain) Validator() Validator {}
 
```

```go
// WriteBlockWithoutState仅将块及其元数据写入数据库，但不写入任何状态。 这用于构建竞争方叉，直到超过规范总难度。
func (bc *BlockChain) WriteBlockWithoutState(block *types.Block, td *big.Int) (err error){}
 
// WriteBlockWithState将块和所有关联状态写入数据库。
func (bc *BlockChain) WriteBlockWithState(block *types.Block, receipts []*types.Receipt, state *state.StateDB) {}
 
// writeHeader将标头写入本地链，因为它的父节点已知。 如果新插入的报头的总难度变得大于当前已知的TD，则重新路由规范链
func (bc *BlockChain) writeHeader(header *types.Header) error{}
 
// 处理未来区块链
func (bc *BlockChain) update() {}
```

## blockchain初始化

主要步骤：

①：创建一个新的headerChain结构

```go
bc.hc, err = NewHeaderChain(db, chainConfig, engine, bc.getProcInterrupt)
```

1. 根据number（0）获取genesisHeader
2. 从rawdb中读取HeadBlock并存储在currentHeader中

②：获取genesisBlock

```go
bc.genesisBlock = bc.GetBlockByNumber(0)
```

③：如果链不为空，则用老的链数据初始化链

```go
if bc.empty() {
		rawdb.InitDatabaseFromFreezer(bc.db)
	}
```

④：加载最新的状态数据

```go
if err := bc.loadLastState(); err != nil {
		return nil, err
	}
```

⑤：检查区块哈希的当前状态，并确保链中没有任何坏块

```go
for hash := range BadHashes {
		if header := bc.GetHeaderByHash(hash); header != nil {
			headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
			if headerByNumber != nil && headerByNumber.Hash() == header.Hash() {
				log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
				bc.SetHead(header.Number.Uint64() - 1)
				log.Error("Chain rewind was successful, resuming normal operation")
			}
		}
	}
```

⑥：定时处理future block

```go
go bc.update()
	->procFutureBlocks
		->InsertChain
```

总的来说做了以下几件事：

1. 配置`cacheConfig`，创建各种lru缓存
2. 初始化`triegc`
3. 初始化`stateDb`：**state.NewDatabase(db)**
4. 初始化区块和状态验证：**NewBlockValidator()**
5. 初始化状态处理器：**NewStateProcessor()**
6. 初始化区块头部链：**NewHeaderChain()**
7. 查找创世区块：**bc.genesisBlock = bc.GetBlockByNumber(0)**
8. 加载最新的状态数据：**bc.loadLastState()**
9. 检查区块哈希的当前状态，并确保链中没有任何坏块
10. `go bc.update()` 定时处理`future block`

## 加载区块链状态

①：从数据库中恢复headblock，如果空的话，触发reset chain

```go
head := rawdb.ReadHeadBlockHash(bc.db)
	if head == (common.Hash{}) {
		log.Warn("Empty database, resetting chain")
		return bc.Reset()
	}
```

②：确保整个head block是可以获取的，若为空，则触发reset chain

```go
currentBlock := bc.GetBlockByHash(head)
	if currentBlock == nil {
		// Corrupt or empty database, init from scratch
		log.Warn("Head block missing, resetting chain", "hash", head)
		return bc.Reset()
	}
```

③：从stateDb中打开最新区块的状态trie，如果打开失败调用bc.repair(&currentBlock)方法进行修复。修复方法就是从当前区块一个个的往前面找，直到找到好的区块，然后赋值给currentBlock。

```go
if _, err := state.New(currentBlock.Root(), bc.stateCache); err != nil {
		// Dangling block without a state associated, init from scratch
		log.Warn("Head state missing, repairing chain", "number", currentBlock.Number(), "hash", currentBlock.Hash())
		if err := bc.repair(&currentBlock); err != nil {
			return err
		}
		rawdb.WriteHeadBlockHash(bc.db, currentBlock.Hash())
	}
```

④：存储当前的headblock和设置当前的headHeader以及头部快速块

```go
bc.currentBlock.Store(currentBlock)
....
bc.hc.SetCurrentHeader(currentHeader)
...
bc.currentFastBlock.Store(currentBlock)
```



----

## 参考

> https://mindcarver.cn
>
> https://github.com/blockchainGuide