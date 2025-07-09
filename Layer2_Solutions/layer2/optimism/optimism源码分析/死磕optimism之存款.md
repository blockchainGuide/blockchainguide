> 本源码分析基于
>
> optimism : https://github.com/ethereum-optimism/optimism/tree/v1.0.9 
>
> Op-geth : https://github.com/ethereum-optimism/op-geth/tree/v1.101105.2 
>
> spec : https://github.com/ethereum-optimism/optimism/tree/v1.0.9/specs

# 存款

存款交易也叫存款，是在 L1 上发起并在 **L2 上执行的交易**。

此存款交易和现有的交易的区别：

- 由L1区块驱动，必须作为协议的一部分
- 不包含签名

### 存款交易的几个字段

- from：OptimismPortal 合约
- to:任何20字节地址，在创建合约的情况下，设置为null
- mint：要在L2上铸造的token
- value：数量
- gaslimit :和发出的值相比没有变化，至少为21000
- isCreation：如果存款交易是合约创建，则设置为true，否则false
- Data: 根据isCreation, 要么为calldata,要么为合约初始代码
- isSystemTx: 由汇总节点为具有未计量执行的某些事务设置。用户存款交易为假  TODO

## L1 存入金额到L2

存入金额调用的是depositETH，这里的calldata🚩, 在存入的时候L1桥合约会记录L1地址映射到L2地址所存入的金额.

QA： 为什么需要地址别名

- https://community.optimism.io/docs/developers/build/differences/#using-eth-in-contracts
- https://community.optimism.io/docs/developers/build/differences/#accessing-the-latest-l1-block-number

QA : l2gas 是如何获取L2上的gas的

QA ： finalizeDeposit传输零地址的意思



### 用户质押

从L1部署的L1StandardBridge.sol 出发，用户通过`depositETH` 将ETH存入到L2的 `msg.sende`r账户

```solidity
// L1StandardBridge.sol
function depositETH(uint32 _l2Gas, bytes calldata _data) external payable onlyEOA {
_initiateETHDeposit(msg.sender, msg.sender, _l2Gas, _data);
}

// L1StandardBridge.sol
function _initiateETHDeposit(
address _from,
address _to,
uint32 _minGasLimit,
bytes memory _extraData
) internal {
_initiateBridgeETH(_from, _to, msg.value, _minGasLimit, _extraData);
}   

// StandardBridge.sol 
function _initiateBridgeETH(
  address _from,
  address _to,
  uint256 _amount,
  uint32 _minGasLimit,
  bytes memory _extraData
) internal {
  ...
  // 只能由L2上的Bridge 合约来执行最终的转账
  MESSENGER.sendMessage{ value: _amount }(
      address(OTHER_BRIDGE),
      abi.encodeWithSelector(
          this.finalizeBridgeETH.selector,
          _from,
          _to,
          _amount,
          _extraData
      ),
      _minGasLimit
  );
}

//  StandardBridge.sol 
function finalizeBridgeETH(
  address _from,
  address _to,
  uint256 _amount,
  bytes calldata _extraData
) public payable onlyOtherBridge {
  require(msg.value == _amount, "StandardBridge: amount sent does not match amount required");
  require(_to != address(this), "StandardBridge: cannot send to self");
  require(_to != address(MESSENGER), "StandardBridge: cannot send to messenger");

  _emitETHBridgeFinalized(_from, _to, _amount, _extraData);

  // 这个是将代币转入到L2的 _to 地址中
  bool success = SafeCall.call(_to, gasleft(), _amount, hex"");
  require(success, "StandardBridge: ETH transfer failed");
}

// CrossDomainMessenger.sol
function sendMessage(
  address _target,  // L2上的Bridge合约
  bytes calldata _message,
  uint32 _minGasLimit
) external payable {
  // 向另一个信使发送消息
  _sendMessage(
      OTHER_MESSENGER,
      baseGas(_message, _minGasLimit),
      msg.value,
      abi.encodeWithSelector(
          this.relayMessage.selector,
          messageNonce(),
          msg.sender,
          _target,
          msg.value,
          _minGasLimit,
          _message
      )
  );

  emit SentMessage(_target, msg.sender, _message, messageNonce(), _minGasLimit);
  emit SentMessageExtension1(msg.sender, msg.value);

  unchecked {
      ++msgNonce;
  }
}

function _sendMessage(
  address _to,
  uint64 _gasLimit,
  uint256 _value,
  bytes memory _data
) internal override {
  PORTAL.depositTransaction{ value: _value }(_to, _value, _gasLimit, false, _data);
}

  function depositTransaction(
    address _to,
    uint256 _value,
    uint64 _gasLimit,
    bool _isCreation,
    bytes memory _data
) public payable metered(_gasLimit) {
    // Just to be safe, make sure that people specify address(0) as the target when doing
    // contract creations.
    if (_isCreation) {
        require(
            _to == address(0),
            "OptimismPortal: must send to address(0) when creating a contract"
        );
    }

    // Prevent depositing transactions that have too small of a gas limit. Users should pay
    // more for more resource usage.
    require(
        _gasLimit >= minimumGasLimit(uint64(_data.length)),
        "OptimismPortal: gas limit too small"
    );

    // Prevent the creation of deposit transactions that have too much calldata. This gives an
    // upper limit on the size of unsafe blocks over the p2p network. 120kb is chosen to ensure
    // that the transaction can fit into the p2p network policy of 128kb even though deposit
    // transactions are not gossipped over the p2p network.
    require(_data.length <= 120_000, "OptimismPortal: data too large");

    // Transform the from-address to its alias if the caller is a contract.
    address from = msg.sender;
    if (msg.sender != tx.origin) {
        from = AddressAliasHelper.applyL1ToL2Alias(msg.sender);
    }

    // Compute the opaque data that will be emitted as part of the TransactionDeposited event.
    // We use opaque data so that we can update the TransactionDeposited event in the future
    // without breaking the current interface.
    bytes memory opaqueData = abi.encodePacked(
        msg.value,
        _value,
        _gasLimit,
        _isCreation,
        _data
    );

    // Emit a TransactionDeposited event so that the rollup node can derive a deposit
    // transaction for this deposit.
    emit TransactionDeposited(from, _to, DEPOSIT_VERSION, opaqueData);
}
```

用户质押交易最终会在L2上被执行，L2上的信使会调用消息，通过Bridge来转账给L2上的reciveer,最终的跨链调用数据都会保存在opaqueData， 通过事件 emit TransactionDeposited(from, _to, DEPOSIT_VERSION, opaqueData); 被op-node 解析出来并调用op-geth去执行，从而实现跨链合约调用。

Op-geth 调用evm 执行了上述 depositTx , 继续要看L2相关的合约。 



```go
```





applyTransaction 最终会将交易发到ch:

```GO
s.txFeed.Send(core.NewTxsEvent{
		Txs:   txs,
		ErrCh: errCh,
	})
```

```go
case ev := <-w.rollupCh:
		...				
		if err := w.commitNewTx(tx); err == nil {
		...
```

commitNewTx（提交单个交易DTL扫的交易）->applyTransaction->applyMessage->evm执行->writeBlockWithState ，sequencer是通过POA共识的。最终挖出一个L2的区块并写入数据库。同时移除了w.chainHeadCh提交挖矿任务，这样只能通过执行同步服务从DTL拉过来的交易和用户发给sequencer的交易(两种交易来源)来执行生成L2 block .同时注意到把TransactionMeta（这是什么@@@@）也记录到state db去了。

🚩这里关于L1到L2的消息如何转换成l2交易的具体过程,是需要详细解释的，这才是比较关键的一步

这里在L2上执行的交易应该是L2上的 finalizeDeposit方法，！！！！！！ ，目前来看貌似是一个交易一个块

```go
txs := block.Transactions()
	if len(txs) != 1 {
		panic(fmt.Sprintf("attempting to create batch element from block %d, "+
			"found %d txs instead of 1", block.Number(), len(txs)))
	}
```

所以他的批处理，实际上是处理一批L2区块的一笔交易（总共1笔）， 带Metadata@@@@ 这是什么玩意

batch-submitter 监听L2区块，会打包txBatch 提交到L1合约,首先会一直判断是否有L2block更新：

```GO
start, end, err := s.cfg.Driver.GetBatchBlockRange(s.ctx)
```

接着会通过CraftBatchTx使用给定的nonce将开始和结束之间的L2块转换为批处理交易。在生成的交易中使用虚拟天然气价格，以用于规模估计。

```GO
tx, err := s.cfg.Driver.CraftBatchTx(
				s.ctx, start, end, nonce,
			)
```

批处理交易转换完成了之后还是会调用，batch-submitter会使用L1的客户端去发送这笔交易（to是CTC合约），这笔交易是CTC合约生成的交易@@@@，这是什么玩法？

```GO
tx, err := d.rawCtcContract.RawTransact(opts, calldata)

func (c *BoundContract) transact(opts *TransactOpts, contract *common.Address, input []byte) (*types.Transaction, error) {
	var err error
。。。。。
}
```

实际就是通过连接的L1客户端去发送交易到绑定的CTC合约（CanonicalTransactionChain.appendSequencerBatch()）这个函数，

```GO
// @@@@ CALLdata用来存储要调用的方法吗，这一整套需要查询 go 发起合约调用交易！！！！！重点了解		
// 数据全部记录在batchElement，序列化了，里面是每个L2区块的交易
appendSequencerBatchID := d.ctcABI.Methods[appendSequencerBatchMethodName].ID
		calldata := append(appendSequencerBatchID, batchArguments...)
```

```GO
    function appendSequencerBatch() external {
        uint40 shouldStartAtElement;
        uint24 totalElementsToAppend;
        uint24 numContexts;
        assembly {
            shouldStartAtElement := shr(216, calldataload(4))
            totalElementsToAppend := shr(232, calldataload(9))
            numContexts := shr(232, calldataload(12))
        }

        require(
            shouldStartAtElement == getTotalElements(),
            "Actual batch start index does not match expected start index."
        );

        require(
            msg.sender == resolve("OVM_Sequencer"),
            "Function can only be called by the Sequencer."
        );

        uint40 nextTransactionPtr = uint40(
            BATCH_CONTEXT_START_POS + BATCH_CONTEXT_SIZE * numContexts
        );

        require(msg.data.length >= nextTransactionPtr, "Not enough BatchContexts provided.");

        // Counter for number of sequencer transactions appended so far.
        uint32 numSequencerTransactions = 0;

        // Cache the _nextQueueIndex storage variable to a temporary stack variable.
        // This is safe as long as nothing reads or writes to the storage variable
        // until it is updated by the temp variable.
        uint40 nextQueueIndex = _nextQueueIndex;

        BatchContext memory curContext;
        for (uint32 i = 0; i < numContexts; i++) {
            BatchContext memory nextContext = _getBatchContext(i);

            // Now we can update our current context.
            curContext = nextContext;

            // Process sequencer transactions first.
            numSequencerTransactions += uint32(curContext.numSequencedTransactions);

            // Now process any subsequent queue transactions.
            nextQueueIndex += uint40(curContext.numSubsequentQueueTransactions);
        }

        require(
            nextQueueIndex <= queueElements.length,
            "Attempted to append more elements than are available in the queue."
        );

        // Generate the required metadata that we need to append this batch
        uint40 numQueuedTransactions = totalElementsToAppend - numSequencerTransactions;
        uint40 blockTimestamp;
        uint40 blockNumber;
        if (curContext.numSubsequentQueueTransactions == 0) {
            // The last element is a sequencer tx, therefore pull timestamp and block number from
            // the last context.
            blockTimestamp = uint40(curContext.timestamp);
            blockNumber = uint40(curContext.blockNumber);
        } else {
            // The last element is a queue tx, therefore pull timestamp and block number from the
            // queue element.
            // curContext.numSubsequentQueueTransactions > 0 which means that we've processed at
            // least one queue element. We increment nextQueueIndex after processing each queue
            // element, so the index of the last element we processed is nextQueueIndex - 1.
            Lib_OVMCodec.QueueElement memory lastElement = queueElements[nextQueueIndex - 1];

            blockTimestamp = lastElement.timestamp;
            blockNumber = lastElement.blockNumber;
        }

        // Cache the previous blockhash to ensure all transaction data can be retrieved efficiently.
        // slither-disable-next-line reentrancy-no-eth, reentrancy-events
        _appendBatch(
            blockhash(block.number - 1),
            totalElementsToAppend,
            numQueuedTransactions,
            blockTimestamp,
            blockNumber
        );

        // slither-disable-next-line reentrancy-events
        emit SequencerBatchAppended(
            nextQueueIndex - numQueuedTransactions,
            numQueuedTransactions,
            getTotalElements()
        );

        // Update the _nextQueueIndex storage variable.
        // slither-disable-next-line reentrancy-no-eth
        _nextQueueIndex = nextQueueIndex;
    }
```

```solidity
/**
     * Inserts a batch into the chain of batches.
     * @param _transactionRoot Root of the transaction tree for this batch.
     * @param _batchSize Number of elements in the batch.
     * @param _numQueuedTransactions Number of queue transactions in the batch.
     * @param _timestamp The latest batch timestamp.
     * @param _blockNumber The latest batch blockNumber.
     */
    function _appendBatch(
        bytes32 _transactionRoot,
        uint256 _batchSize,
        uint256 _numQueuedTransactions,
        uint40 _timestamp,
        uint40 _blockNumber
    ) internal {
        IChainStorageContainer batchesRef = batches();
        (uint40 totalElements, uint40 nextQueueIndex, , ) = _getBatchExtraData();

        Lib_OVMCodec.ChainBatchHeader memory header = Lib_OVMCodec.ChainBatchHeader({
            batchIndex: batchesRef.length(),
            batchRoot: _transactionRoot,
            batchSize: _batchSize,
            prevTotalElements: totalElements,
            extraData: hex""
        });

        emit TransactionBatchAppended(
            header.batchIndex,
            header.batchRoot,
            header.batchSize,
            header.prevTotalElements,
            header.extraData
        );

        bytes32 batchHeaderHash = Lib_OVMCodec.hashBatchHeader(header);
        bytes27 latestBatchContext = _makeBatchExtraData(
            totalElements + uint40(header.batchSize),
            nextQueueIndex + uint40(_numQueuedTransactions),
            _timestamp,
            _blockNumber
        );

        // slither-disable-next-line reentrancy-no-eth, reentrancy-events
        batchesRef.push(batchHeaderHash, latestBatchContext);
    }
```

DTL 监听TransactionBatchAppended事件，之后做什么事情呢？？？@@@@

```typescript
export const handleEventsSequencerBatchAppended: EventHandlerSet<
  SequencerBatchAppendedEvent,
  SequencerBatchAppendedExtraData,
  SequencerBatchAppendedParsedEvent
> = {
  ....
}
```

上面的事是sequencer做的

proposert提交block.root到StateCommitmentChain合约里

```solidity
 function _appendBatch(bytes32[] memory _batch, bytes memory _extraData) internal {
        address sequencer = resolve("OVM_Proposer");
        (uint40 totalElements, uint40 lastSequencerTimestamp) = _getBatchExtraData();

        if (msg.sender == sequencer) {
            lastSequencerTimestamp = uint40(block.timestamp);
        } else {
            // We keep track of the last batch submitted by the sequencer so there's a window in
            // which only the sequencer can publish state roots. A window like this just reduces
            // the chance of "system breaking" state roots being published while we're still in
            // testing mode. This window should be removed or significantly reduced in the future.
            require(
                lastSequencerTimestamp + SEQUENCER_PUBLISH_WINDOW < block.timestamp,
                "Cannot publish state roots within the sequencer publication window."
            );
        }

        // For efficiency reasons getMerkleRoot modifies the `_batch` argument in place
        // while calculating the root hash therefore any arguments passed to it must not
        // be used again afterwards
        Lib_OVMCodec.ChainBatchHeader memory batchHeader = Lib_OVMCodec.ChainBatchHeader({
            batchIndex: getTotalBatches(),
            batchRoot: Lib_MerkleTree.getMerkleRoot(_batch),
            batchSize: _batch.length,
            prevTotalElements: totalElements,
            extraData: _extraData
        });

        emit StateBatchAppended(
            batchHeader.batchIndex,
            batchHeader.batchRoot,
            batchHeader.batchSize,
            batchHeader.prevTotalElements,
            batchHeader.extraData
        );

        batches().push(
            Lib_OVMCodec.hashBatchHeader(batchHeader),
            _makeBatchExtraData(
                uint40(batchHeader.prevTotalElements + batchHeader.batchSize),
                lastSequencerTimestamp
            )
        );
    }
```







## 疑问

1. 合约调用链 msg.sender到底是谁
2. [函数选择器使用](https://mirror.xyz/wtfacademy.eth/_Q-N_VGUV8F4QZbggR8Swv16LStBdfkeQb8qwSfoNTw)