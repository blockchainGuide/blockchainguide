> 死磕以太坊源码分析之blockChain分析
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/

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

## 插入数据到blockchain中

①：如果链正在中断，直接返回

```GO
if atomic.LoadInt32(&bc.procInterrupt) == 1 {
		return 0, nil
	}
```

②：开启并行的签名恢复

```GO
	senderCacher.recoverFromBlocks(types.MakeSigner(bc.chainConfig, chain[0].Number()), chain)
```

③：校验header

```go
abort, results := bc.engine.VerifyHeaders(bc, headers, seals)
```

④：循环校验body

```go
block, err := it.next()
	-> ValidateBody
		-> VerifyUncles
```

包括以下错误：

- block已知
- uncle太多
- 重复的uncle
- uncle是祖先块
- uncle哈希不匹配
- 交易哈希不匹配
- 未知祖先
- 祖先块的状态无法获取

如果block存在，且是已知块，则写入已知块。 

```go
bc.writeKnownBlock(block)
```

如果是祖先块的状态无法获取的错误，则作为侧链插入：

```go
bc.insertSideChain(block, it)
```

如果是未来块或者未知祖先，则添加未来块：

```go
bc.addFutureBlock(block);
```

如果是其他错误，直接中断，并且报告坏块。

```go
bc.futureBlocks.Remove(block.Hash())
...
bc.reportBlock(block, nil, err)
```

⑤：没有校验错误

如果是坏块，则报告；

```go
if BadHashes[block.Hash()] {
			bc.reportBlock(block, nil, ErrBlacklistedHash)
			return it.index, ErrBlacklistedHash
		}
```

如果是未知块，则写入未知块；

```go
if err == ErrKnownBlock {
			logger := log.Debug
			if bc.chainConfig.Clique == nil {
				logger = log.Warn
			}
			logger("Inserted known block", "number", block.Number(), "hash", block.Hash(),
				"uncles", len(block.Uncles()), "txs", len(block.Transactions()), "gas", block.GasUsed(),
				"root", block.Root())

			if err := bc.writeKnownBlock(block); err != nil {
				return it.index, err
			}
			stats.processed++
			lastCanon = block
			continue
		}
```

根据给定trie，创建state；

```go
parent := it.previous()
		if parent == nil {
			parent = bc.GetHeader(block.ParentHash(), block.NumberU64()-1)
		}
		statedb, err := state.New(parent.Root, bc.stateCache)
```

执行块中的交易：

```go
receipts, logs, usedGas, err := bc.processor.Process(block, statedb, bc.vmConfig)
```

使用默认的validator校验状态：

```go
bc.validator.ValidateState(block, statedb, receipts, usedGas);
```

将块写入到区块链中并获取状态：

```go
status, err := bc.writeBlockWithState(block, receipts, logs, statedb, false)
```

⑥：校验写入区块的状态

- CanonStatTy ： 插入成功新的block
- SideStatTy：插入成功新的分叉区块
- Default：插入未知状态的block

⑦：如果还有块，并且是未来块的话，那么将块添加到未来块的缓存中去

```go
bc.addFutureBlock(block)
```

至此insertChain 大概介绍清楚。

-----

### 执行块中交易

在我们将区块上链，有一个关键步骤就是执行区块交易：

```go
receipts, logs, usedGas, err := bc.processor.Process(block, statedb, bc.vmConfig)
```

进入函数，具体分析：

①：准备要用的字段，循环执行交易

关键函数：`ApplyTransaction`,根据此函数返回收据。

1.1 将交易结构转成`Message`结构

```go
msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
```

1.2 创建要在EVM环境中使用的新上下文

```go
context := NewEVMContext(msg, header, bc, author)
```

1.3 创建一个新环境，其中包含有关事务和调用机制的所有相关信息。

```GO
vmenv := vm.NewEVM(context, statedb, config, cfg)
```

1.4 将交易应用到当前状态(包含在env中)

```go
_, gas, failed, err := ApplyMessage(vmenv, msg, gp)
```

这部分代码继续跟进：

```go
func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, uint64, bool, error) {
	return NewStateTransition(evm, msg, gp).TransitionDb()
}
```

`NewStateTransition` 是一个状态转换对象，`TransitionDb()` 负责转换交易状态，继续跟进：
先进行`preCheck`，用来校验`nonce`是否正确

```go
st.preCheck()

if st.msg.CheckNonce() {
		nonce := st.state.GetNonce(st.msg.From())
		if nonce < st.msg.Nonce() {
			return ErrNonceTooHigh
		} else if nonce > st.msg.Nonce() {
			return ErrNonceTooLow
		}
	}
```

计算所需`gas`：

```go
gas, err := IntrinsicGas(st.data, contractCreation, homestead, istanbul)
```

扣除`gas`：

```go
if err = st.useGas(gas); err != nil {
		return nil, 0, false, err
	}
```

```go
func (st *StateTransition) useGas(amount uint64) error {
	if st.gas < amount {
		return vm.ErrOutOfGas
	}
	st.gas -= amount
	return nil
}
```

如果是合约交易,则新建一个合约

```go
ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
```

如果不是合约交易，则增加`nonce`

```go
st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
```

重点关注`evm.call`方法：

*检查账户是否有足够的气体进行转账*

```go
if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, gas, ErrInsufficientBalance
	}
```

*如果stateDb不存在此账户，则新建账户*

```go
if !evm.StateDB.Exist(addr) {
  evm.StateDB.CreateAccount(addr)
}
```

*执行转账操作*

```go
evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)
```

*创建合约*

```go
contract := NewContract(caller, to, value, gas)
```

*执行合约*

```go
ret, err = run(evm, contract, input, false)
```

添加余额

```go
	st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))
```

回到`ApplyTransaction`

1.5 调用`IntermediateRoot`计算状态`trie`的当前根哈希值。

最终确定所有肮脏的存储状态，并把它们写进`trie`

```go
s.Finalise(deleteEmptyObjects)
```

将trie根设置为当前的根哈希并将给定的`object`写入到`trie`中

```go
obj.updateRoot(s.db)
s.updateStateObject(obj)
```

1.6 创建收据

```go
receipt := types.NewReceipt(root, failed, *usedGas)
	receipt.TxHash = tx.Hash()
	receipt.GasUsed = gas
	// if the transaction created a contract, store the creation address in the receipt.
	if msg.To() == nil {
		receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
	}
	// Set the receipt logs and create a bloom for filtering
	receipt.Logs = statedb.GetLogs(tx.Hash())
	receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
	receipt.BlockHash = statedb.BlockHash()
	receipt.BlockNumber = header.Number
	receipt.TransactionIndex = uint(statedb.TxIndex())
```

②：最后完成区块，应用任何共识引擎特定的额外功能(例如区块奖励)

```go
p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles())
```

```go
func (ethash *Ethash) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header) {
	// Accumulate any block and uncle rewards and commit the final state root
	//累积任何块和叔叔的奖励并提交最终状态树根
	accumulateRewards(chain.Config(), state, header, uncles)
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))
}
```

到此为止`bc.processor.Process`执行完毕，返回`receipts`.

------

### 校验状态

大致包括4部分的校验：

①：校验使用的`gas`是否相等

```go
if block.GasUsed() != usedGas {
		return fmt.Errorf("invalid gas used (remote: %d local: %d)", block.GasUsed(), usedGas)
	}
```

②：校验bloom是否相等

```go
rbloom := types.CreateBloom(receipts)
	if rbloom != header.Bloom {
		return fmt.Errorf("invalid bloom (remote: %x  local: %x)", header.Bloom, rbloom)
	}
```

③：校验收据哈希是否相等

```go
receiptSha := types.DeriveSha(receipts)
	if receiptSha != header.ReceiptHash {
		return fmt.Errorf("invalid receipt root hash (remote: %x local: %x)", header.ReceiptHash, receiptSha)
	}
```

④：校验merkleroot 是否相等

```go
if root := statedb.IntermediateRoot(v.config.IsEIP158(header.Number)); header.Root != root {
		return fmt.Errorf("invalid merkle root (remote: %x local: %x)", header.Root, root)
	}
```

-----

### 将块和关联状态写入到数据库

函数：**WriteBlockWithState**

①：计算块的`total td` 

```go
ptd := bc.GetTd(block.ParentHash(), block.NumberU64()-1)
```

②：添加待插入块本身的`td` ,并将此时最新的`total td` 存储到数据库中。

```GO
bc.hc.WriteTd(block.Hash(), block.NumberU64(), externTd)
```

③：将块的`header`和`body`分别序列化到数据库

```go
rawdb.WriteBlock(bc.db, block)
	->WriteBody(db, block.Hash(), block.NumberU64(), block.Body())
	->WriteHeader(db, block.Header())
```

④：将状态写入底层内存`Trie`数据库

```go
state.Commit(bc.chainConfig.IsEIP158(block.Number()))
```

⑤：遍历节点数据写入到磁盘

```go
triedb.Commit(header.Root, true)
```

⑥：存储一个块的所有交易数据

```go
rawdb.WriteReceipts(batch, block.Hash(), block.NumberU64(), receipts)
```

⑦：将新的head块注入到当前链中

```go
if status == CanonStatTy {
		bc.insert(block)
	}
```

- 存储分配给规范块的哈希
- 存储头块的哈希
- 存储最新的快
- 更新currentFastBlock

⑧：发送`chainEvent`事件或者`ChainSideEvent`事件或者`ChainHeadEvent`事件

```go
if status == CanonStatTy {
		bc.chainFeed.Send(ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
		if len(logs) > 0 {
			bc.logsFeed.Send(logs)
    }
		if emitHeadEvent {
			bc.chainHeadFeed.Send(ChainHeadEvent{Block: block})
		}
	} else {
		bc.chainSideFeed.Send(ChainSideEvent{Block: block})
	}
```

到此writeBlockWithState 结束，从上面可以知道，insertChain的最终还是调用了writeBlockWithState的insert方法完成了最终的插入动作。

最后整个insertChain *函数，如果已经完成了插入，就发送chain head事件*

```go
	defer func() {
		if lastCanon != nil && bc.CurrentBlock().Hash() == lastCanon.Hash() {
			bc.chainHeadFeed.Send(ChainHeadEvent{lastCanon})
		}
	}()
```



----

## 参考

> https://mindcarver.cn