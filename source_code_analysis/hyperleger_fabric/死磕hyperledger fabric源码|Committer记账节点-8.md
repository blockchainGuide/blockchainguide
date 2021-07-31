> 死磕hyperledger fabric源码|Committer记账节点
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![facdd49577decf1dc62abc9fc97caf97](https://tva1.sinaimg.cn/large/008eGmZEgy1gn3h9nzkclj31c00u0wt0.jpg)



## 概述

`Committer`记账节点功能模块的设计与实现的源代码主要分布在下表：

| 源码目录 |           |       文件        |       功能阐述       |
| :------: | :-------: | :---------------: | :------------------: |
|   core   | committer |    Txvalidator    |  交易验证器功能模块  |
|          |           |   committer.go    |  账本提交器接口定义  |
|          |           | committer_impl.go |    账本提交器实现    |
|          |  ledger   |      kvledge      | kvLedger账本功能模块 |
|          |           |   ledgerstorage   | 账本数据存储对象模块 |
|          |           |  pvtdatastorage   | 隐私数据存储对象模块 |
|          |           |    ledgermgmt     |     账本管理模块     |
|          |           |     customtx      |  配置交易处理器模块  |
|  common  |  ledger   |   blockstorage    |     区块存储模块     |
|  protos  |  Common   |      ledger       | protobuf消息定义模块 |

接下来将会围绕着这部分的内容进行分析。



## 创建Committer功能模块

Peer节点通过请求调用`CSCC`系统链码加入应用通道，执行`joinChain()→peer.Create- ChainFromBlock()→createChain()`函数，基于应用通道创世区块创建通道的链结构对象，用于管理账本、通道配置等资源，以正常接收通道的账本区块。

接着，创建了交易验证器，并封装了`vsccValidatorImpl`结构对象用于支持调用VSCC链码。

然后，创建账本提交器，并定义回调函数`eventer`，用于提交账本后自动更新链结构上的最新配置区块对象。

现在进入到`createChain`里面分析：

```go
func createChain(cid string, ledger ledger.PeerLedger, cb *common.Block) error {
...
  vcs := struct { // 构造新的验证链码支持对象
		*chainSupport
		*semaphore.Weighted
		Support
	}{cs, validationWorkersSemaphore, GetSupport()}
	validator := txvalidator.NewTxValidator(vcs) // 创建交易验证器
	// 创建账本提交器
	c := committer.NewLedgerCommitterReactive(ledger, func(block *common.Block) error {
		chainID, err := utils.GetChainIDFromBlock(block)
		if err != nil {
			return err
		}
		return SetCurrConfigBlock(block, chainID)
	})
...
	// 创建transient隐私数据存储对象
	store, err := transientStoreFactory.OpenStore(bundle.ConfigtxValidator().ChainID())
	...
	// 初始化指定通道上的Gossip消息模块。
	// 若是主节点，则从Orderer服务节点获取区块数据。否则，从组织内其他节点同步数据
	service.GetGossipService().InitializeChannel(bundle.ConfigtxValidator().ChainID(), ordererAddresses, service.Support{
		Validator: validator,
		Committer: c,
		Store:     store,
		Cs:        simpleCollectionStore,
	})

	chains.Lock()
	defer chains.Unlock()
	// 构造新的链结构并插入Peer节点链结构
	chains.list[cid] = &chain{
		cs:        cs, // 链支持对象
		cb:        cb, // 配置区块
		committer: c,  // 账本提交器
	}

}
```



## 验证交易数据的合法性

验证交易入口：`core/committer/txvalidator/validator.go/validateTx()`,主要做了以下几件事

①：*解析获取交易数据的Envelope结构对象*

```go
if env, err := utils.GetEnvelopeFromBlock(d); err != nil {}
```

②：*检查交易格式是否正确、签名是否合法、交易内容是否被篡改*

```go
if payload, txResult = validation.ValidateTransaction(env, v.support.Capabilities()); txResult != peer.TxValidationCode_VALID {
			logger.Errorf("Invalid transaction with index %d", tIdx)
			results <- &blockValidationResult{
				tIdx:           tIdx,
				validationCode: txResult,
			}
			return
		}
```

③：*解析获取通道头部*

```go
chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
```

④：*检查通道链结构是否存在*

```go
channel := chdr.ChannelId
if !v.chainExists(channel) {}
```



⑤：*根据`Header`的类型来分别处理消息*

5.1 *普通交易消息*

先从账本获取指定交易的ID数据，检查是否存在，然后获取交易读写集，并检查写集的合法性，调用`VSCC`验证交易背书策略，接着获取交易链码实例，并设置调用链码实例

```go
txID = chdr.TxId
// 从账本获取指定交易的ID数据，检查是否存在
if _, err := v.support.Ledger().GetTransactionByID(txID); err == nil {
  ...
}
// 获取交易读写集，并检查写集的合法性，调用VSCC验证交易背书策略
err, cde := v.vscc.VSCCValidateTx(payload, d, env)
i..
// 获取交易链码实例
invokeCC, upgradeCC, err := v.getTxCCInstance(payload)
...
// 设置调用链码实例
txsChaincodeName = invokeCC
```

5.2 *通道配置交易消息*

先解析获取配置交易对象，然后更新通道配置。

```go
// 通道配置交易消息，解析获取配置交易对象
configEnvelope, err := configtx.UnmarshalConfigEnvelope(payload.Data)
...
// 更新通道配置
if err := v.support.Apply(configEnvelope); err != nil {
 ...
}
```

5.3 如果是*Peer资源更新消息*，直接构造`blockValidationResult`返回

⑥：*序列化封装交易Envelope结构对象*

```go
if _, err := proto.Marshal(env); err != nil {
			logger.Warningf("Cannot marshal transaction: %s", err)
			results <- &blockValidationResult{
				tIdx:           tIdx,
				validationCode: peer.TxValidationCode_MARSHAL_TX_ERROR,
			}
			return
		}
```

⑦：*最后通过了交易，基于上述参数构造区块验证结果对象*

```go
results <- &blockValidationResult{
			tIdx:                 tIdx,
			txsChaincodeName:     txsChaincodeName,
			txsUpgradedChaincode: txsUpgradedChaincode,
			validationCode:       peer.TxValidationCode_VALID,
			txid:                 txID,
		}
		return
```

## 账本提交器

账本提交器的`LedgerCommitter.CommitWithPvtData()`方法负责执行具体的账本提交工作。该方法首先调用`LedgerCommitter`对象的`lc.preCommit(blockAndPvtData.Block)`方法，预处理待提交的区块数据，对于配置区块执行自定义`lc.eventer(block)`回调函数，即从当前区块中解析出链ID，再调用`SetCurrConfigBlock`()函数，从本地链结构字典中获取关联的链结构`chains.list[cid]`并更新其最新的配置区块。接着，调用`lc.PeerLedger.CommitWithPvtData(blockAndPvtData)→kvLedger.CommitWithPvtData()`方法提交数据到账本中，这是账本提交器的核心工作方法。当成功提交账本后，调用`lc.postCommit(blockAndPvtData.Block)`方法，基于该区块创建区块事件与过滤区块事件，并执行`producer.Send()`方法将两个事件发送到事件服务器，通知订阅客户端有新区块到达。

进入到`CommitWithPvtData`()方法中：

```go
func (l *kvLedger) CommitWithPvtData(pvtdataAndBlock *ledger.BlockAndPvtData) error {
	var err error
	block := pvtdataAndBlock.Block                 // 获取区块对象
	blockNo := pvtdataAndBlock.Block.Header.Number // 获取区块号
	// 验证并准备区块和隐私数据对象
	err = l.txtmgmt.ValidateAndPrepare(pvtdataAndBlock, true)
	...
	//提交区块和隐私数据到账本中
	if err = l.blockStore.CommitWithPvtData(pvtdataAndBlock); err != nil {
		return err
	}
	...	
	if err = l.txtmgmt.Commit(); err != nil { // 更新有效交易数据到状态数据库
		panic(fmt.Errorf(`Error during commit to txmgr:%s`, err))
	}
	if ledgerconfig.IsHistoryDBEnabled() {
		logger.Debugf("Channel [%s]: Committing block [%d] transactions to history database", l.ledgerID, blockNo)
		if err := l.historyDB.Commit(block); err != nil { // 更新区块数据到历史数据库
			panic(fmt.Errorf(`Error during commit to history db:%s`, err))
		}
	}
	return nil
}
```

此函数主要也就做了以下比较关键的事情：

- `ValidateAndPrepare`:验证并准备区块和隐私数据对象
- `CommitWithPvtData`:提交区块和隐私数据到账本中
- `txtmgmt.Commit()`:更新有效交易数据到状态数据库
- `l.historyDB.Commit(block)`:更新区块数据到历史数据库

接下来将分别介绍这些功能的细节。

### 验证并准备区块和隐私数据对象

函数调用：

> err = l.txtmgmt.ValidateAndPrepare(pvtdataAndBlock, true)
>
> -> batch, err := txmgr.validator.ValidateAndPrepareBatch(blockAndPvtdata, doMVCCValidation)

```go
func validateAndPreparePvtBatch(block *valinternal.Block, pvtdata map[uint64]*ledger.TxPvtData) (*privacyenabledstate.PvtUpdateBatch, error) {
	pvtUpdates := privacyenabledstate.NewPvtUpdateBatch()
	for _, tx := range block.Txs {
		if tx.ValidationCode != peer.TxValidationCode_VALID {
			continue
		}
		if !tx.ContainsPvtWrites() {
			continue
		}
		txPvtdata := pvtdata[uint64(tx.IndexInBlock)] // 获取指定交易的隐私数据
		if txPvtdata == nil {                         // 跳过没有隐私数据的交易
			continue
		}
		// 检查是否需要验证隐私数据，默认都返回true
		if requiresPvtdataValidation(txPvtdata) {
			// 验证隐私数据哈希值是否匹配
			if err := validatePvtdata(tx, txPvtdata); err != nil {
				return nil, err
			}
		}
		var pvtRWSet *rwsetutil.TxPvtRwSet
		var err error
		// 解析隐私数据写集合
		if pvtRWSet, err = rwsetutil.TxPvtRwSetFromProtoMsg(txPvtdata.WriteSet); err != nil {
			return nil, err
		}
		// 添加到隐私数据更新批量操作
		addPvtRWSetToPvtUpdateBatch(pvtRWSet, pvtUpdates, version.NewHeight(block.Num, uint64(tx.IndexInBlock)))
	}
	return pvtUpdates, nil
}
```

首先遍历当前内部区块的交易列表`block.Txs`，对于其中的每个交易对象tx，需要过滤掉如下三类交易。

- 交易验证码不为`TxValidationCode_VALID`的无效交易
- 不存在隐私数据写数据哈希值的交易
- 无隐私数据的交易

如果交易通过了上述检查，则对于合法有效的交易tx及其隐私数据txPvtdata（TxPvt-Data类型），调用`validatePvtdata(tx，txPvtdata)`方法，以验证隐私数据哈希值的正确性，因为隐私数据都是由`Endorser`背书节点生成的，需要检查传播后的数据是否被篡改过。大致的过程如下：

```go
func validatePvtdata(tx *valinternal.Transaction, pvtdata *ledger.TxPvtData) error {
	...
	for _, nsPvtdata := range pvtdata.WriteSet.NsPvtRwset {
		for _, collPvtdata := range nsPvtdata.CollectionPvtRwset {
			// 基于原始数据计算隐私数据哈希值
			collPvtdataHash := util.ComputeHash(collPvtdata.Rwset)
			// 获取 交易中的数据哈希值
			hashInPubdata := tx.RetrieveHash(nsPvtdata.Namespace, collPvtdata.CollectionName)
			// 比较隐私数据哈希值
			if !bytes.Equal(collPvtdataHash, hashInPubdata) {
				...
		}
	}
	return nil
}
```

### 提交区块和隐私数据到账本中

函数调用：**l.blockStore.CommitWithPvtData(pvtdataAndBlock)**

> core/ledger/ledgerstorage/store.go/CommitWithPvtData

```go
func (s *Store) CommitWithPvtData(blockAndPvtdata *ledger.BlockAndPvtData) error {
	...
	for _, v := range blockAndPvtdata.BlockPvtData {
		// 添加隐私数据到隐私数据列表pvtdata
		pvtdata = append(pvtdata, v)
	}
	// 准备将隐私数据列表pvtdata提交到账本中，先提交再确认
	if err := s.pvtdataStore.Prepare(blockAndPvtdata.Block.Header.Number, pvtdata); err != nil {
		return err
	}
	// 提交区块到账本中
	if err := s.AddBlock(blockAndPvtdata.Block); err != nil {
		s.pvtdataStore.Rollback()
		return err
	}
	// 确认提交隐私数据
	return s.pvtdataStore.Commit()
}
```

大概就做了以下几件事：

1. 准备提交隐私数据：`Prepare`
2. 提交区块数据：`s.AddBlock`
3. 确认提交隐私数据：`s.pvtdataStore.Commit()`

①：准备提交隐私数据

通过隐私数据存储对象调用s.pvtdataStore.Prepare()→store.Prepare()方法，将pvtdata列表中的每个隐私数据对象重新编码并构成KV键值对，添加到账本上隐私数据库的更新批量操作中，并同步更新到数据库中。最后，等待区块数据提交操作确认后，根据提交结果状态确认提交或回滚恢复隐私数据。

```go
func (s *store) Prepare(blockNum uint64, pvtData []*ledger.TxPvtData) error {
	// 检查合法性，执行Prepare()时应该是false，因为Commit和Rollback操作会重置该标志位
	if s.batchPending {
		return &ErrIllegalCall{`A pending batch exists as as result of last invoke to "Prepare" call.
			 Invoke "Commit" or "Rollback" on the pending batch before invoking "Prepare" function`}
	}
	// 获取下一个区块号
	expectedBlockNum := s.nextBlockNum()
	// 检查区块号的合法性
	if expectedBlockNum != blockNum {
		return &ErrIllegalArgs{fmt.Sprintf("Expected block number=%d, recived block number=%d", expectedBlockNum, blockNum)}
	}
	// 创建数据库更新操作集合batch，记录所有需要删除或增加数据的key键
	batch := leveldbhelper.NewUpdateBatch()
	var key, value []byte
	var err error
	// 遍历隐私数据列表，构造该隐私数据KV键值对
	for _, txPvtData := range pvtData {
		// 遍历隐私数据列表，构造该隐私数据KV键值对
		key = encodePK(blockNum, txPvtData.SeqInBlock)
		if value, err = encodePvtRwSet(txPvtData.WriteSet); err != nil {
			// 构造value值：隐私数据写集合
			return err
		}
		logger.Debugf("Adding private data to LevelDB batch for block [%d], tran [%d]", blockNum, txPvtData.SeqInBlock)
		// 添加隐私数据键值对的操作
		batch.Put(key, value)
	}
	// 添加pendingCommitKey键值对的操作
	batch.Put(pendingCommitKey, emptyValue)
	// 同步执行数据库的更新操作集合
	if err := s.db.WriteBatch(batch, true); err != nil {
		return err
	}
	// 更新状态标志位
	s.batchPending = true
	logger.Debugf("Saved %d private data write sets for block [%d]", len(pvtData), blockNum)
	return nil
}
```

②：提交区块数据

调用`s.AddBlock(blockAndPvtdata.Block)`方法，实际上是通过区块文件管理器，调用`blockfileMgr.addBlock()`方法，提交新区块`blockAndPvt-data.Block`到区块数据文件中，并保存新的区块检查点信息`newCPInfo`。接着，调用`indexBlock()`方法，建立当前区块的索引信息与索引检查点信息（当前区块号等），更新到区块索引数据库中。然后，调用`mgr.updateCheckpoint(newCPInfo)`方法，更新区块文件管理器上的区块检查点信息，再执行mgr.cpInfoCond.Broadcast()方法，广播唤醒所有等待该同步条件变量的程序，通知已有新区块提交到账本中。最后，调用`mgr.updateBlockchain-Info()`方法，更新区块链信息，如最新区块高度、最新区块头哈希值等。

③：确认提交隐私数据

调用 `s.pvtdataStore.Commit()`方法，执行隐私数据的提交确认操作。由于前面的`Prepare()`方法已经更新了所有的隐私数据键值对到数据库中，因此，该方法实际上是在隐私数据库上删除`pendingCommitKey`键值对，并添加`lastCommittedBlkkey`键值对，以保存最近提交成功的区块号`committingBlockNum`。最后，更新隐私数据相关标志位与变量，将等待提交确认标志位batchPending与标志位`isEmpty`设置为`false`，将`lastCommitted-Block`更新为提交账本的区块号`committingBlockNum`。 

如果提交区块数据失败，则`CommitWithPvtData()`将通过隐私数据存储对象调用s.pvt-dataStore.Rollback()方法执行回滚操作，将已提交的隐私数据恢复到提交数据库之前的状态。

### 提交数据到状态数据库

入口：**core/ledger/kvledger/txmgmt/txmgr/lockbasedtxmgr/lockbased_txmgr.go/ Commit()**

```go
func (txmgr *LockBasedTxMgr) Commit() error {
	...
	if err := txmgr.db.ApplyPrivacyAwareUpdates(txmgr.batch,
		version.NewHeight(txmgr.currentBlock.Header.Number, uint64(len(txmgr.currentBlock.Data.Data)-1)))
		...
	}
}
```

```go
func (s *CommonStorageDB) ApplyPrivacyAwareUpdates(updates *UpdateBatch, height *version.Height) error {
	addPvtUpdates(updates.PubUpdates, updates.PvtUpdates)
	addHashedUpdates(updates.PubUpdates, updates.HashUpdates, !s.BytesKeySuppoted())
	return s.VersionedDB.ApplyUpdates(updates.PubUpdates.UpdateBatch, height)
}
```

最终会进入到下面的函数中：

> core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go

```go
func (vdb *versionedDB) ApplyUpdates(batch *statedb.UpdateBatch, height *version.Height) error {
	dbBatch := leveldbhelper.NewUpdateBatch()
	namespaces := batch.GetUpdatedNamespaces()
	for _, ns := range namespaces {
		updates := batch.GetUpdates(ns)
		for k, vv := range updates {
			compositeKey := constructCompositeKey(ns, k)
			logger.Debugf("Channel [%s]: Applying key(string)=[%s] key(bytes)=[%#v]", vdb.dbName, string(compositeKey), compositeKey)

			if vv.Value == nil {
				dbBatch.Delete(compositeKey)
			} else {
				dbBatch.Put(compositeKey, statedb.EncodeValue(vv.Value, vv.Version))
			}
		}
	}
	dbBatch.Put(savePointKey, height.ToBytes())
	// Setting snyc to true as a precaution, false may be an ok optimization after further testing.
	if err := vdb.db.WriteBatch(dbBatch, true); err != nil {
		return err
	}
	return nil
}
```

`ApplyUpdates()`方法主要做了以下几件事：

1. 遍历更新批量操作，对于其包含的键值对（键k与值vv），调用`constructCompositeKey(ns，k)`方法重新构造组合键`compositeKey`
2. 检查该键值对操作的删除标识。如果vv.Value为nil，说明该键值对更新操作为删除操作，则继续调用`dbBatch.Delete(compositeKey)`方法，添加该删除操作到`dbBatch`对象中。否则，将vv.Version中的区块号与交易序号经过编码序列化成字节数组，并与`vv.Value`组合成编码值`encodedValue`，再将其写入操作添加到dbBatch对象中
3. 调用`dbBatch.Put`方法，添加保存点标识的`KV`键值对。其中，键为`[]byte{0x00}`，值为版本`height`经过编码序列化后的字节数组
4. 调用`vdb.db.WriteBatch`方法，以原子操作方式将`dbBatch`更新同步到状态数据库上。注意，在写入数据库时同样会重新构造KV键值对，在原来的键上添加数据库名称（链ID/账本ID）前缀，即`[]byte(dbName)+[]byte{0x00}`，以隔离不同通道上的状态数据。



### 更新历史数据库

调用`l.historyDB.Commit(block)`方法，以更新区块`block`中经过`Endorser`背书的有效交易数据到历史数据库中，代码如下：

> core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go

```go
func (historyDB *historyDB) Commit(block *common.Block) error {

	blockNo := block.Header.Number // 获取区块号
	//Set the starting tranNo to 0
	var tranNo uint64
...
	// Get the invalidation byte array for the block
	// 获取交易验证码列表
	txsFilter := util.TxValidationFlags(block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER])
	// Initialize txsFilter if it does not yet exist (e.g. during testing, for genesis block, etc)
	if len(txsFilter) == 0 {
		txsFilter = util.NewTxValidationFlags(len(block.Data.Data))
		block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsFilter
	}

	// write each tran's write set to history db
	for _, envBytes := range block.Data.Data { // 遍历区块所有交易数据
		if txsFilter.IsInvalid(int(tranNo)) { // 过滤掉无效交易
			logger.Debugf("Channel [%s]: Skipping history write for invalid transaction number %d",
				historyDB.dbName, tranNo)
			tranNo++
			continue
		}
		// 解析获取交易消息Envelope结构对象
		env, err := putils.GetEnvelopeFromBlock(envBytes)
		if err != nil {
			return err
		}

	...
		// 检查类型：经Endorser背书的普通交易消息
		if common.HeaderType(chdr.Type) == common.HeaderType_ENDORSER_TRANSACTION {

			// extract actions from the envelope message
			// 从交易消息中解析提取链码动作
			respPayload, err := putils.GetActionFromEnvelope(envBytes)
			if err != nil {
				return err
			}
...
			// 解析交易读写集到TxReadWriteSet结构对象中
			if err = txRWSet.FromProtoBytes(respPayload.Results); err != nil {
				return err
			}
			// 遍历所有读写集，重新构造KV键值对添加到历史数据库
			for _, nsRWSet := range txRWSet.NsRwSets {
				ns := nsRWSet.NameSpace

				for _, kvWrite := range nsRWSet.KvRwSet.Writes {
					writeKey := kvWrite.Key

					// 构造组合键
					compositeHistoryKey := historydb.ConstructCompositeHistoryKey(ns, writeKey, blockNo, tranNo)

					// 写入空的字节数组[]byte{}
					dbBatch.Put(compositeHistoryKey, emptyValue)
				}
			}

		} else {
			// 跳过交易，因为该消息不是经过Endorser背书的普通交易消息
			logger.Debugf("Skipping transaction [%d] since it is not an endorsement transaction\n", tranNo)
		}
		tranNo++
	}
	height := version.NewHeight(blockNo, tranNo) // 创建版本对象
	dbBatch.Put(savePointKey, height.ToBytes())  // 添加保存点用于恢复

	// 同步更新批量操作dbBatch到历史数据库中
	if err := historyDB.db.WriteBatch(dbBatch, true); err != nil {
		return err
	}
...
}
```

到此为止：账本提交功能分析结束。

## 参考 

> https://github.com/blockchainGuide/

