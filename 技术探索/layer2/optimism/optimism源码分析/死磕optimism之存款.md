> 
>
> pre Bed-rock

# 存款

存款交易也叫存款，是在 L1 上发起并在 L2 上执行的交易。



## L1 存入金额到L2

存入金额调用的是depositETH，这里的calldata🚩, 在存入的时候L1桥合约会记录L1地址映射到L2地址所存入的金额.

QA： 为什么需要地址别名

- https://community.optimism.io/docs/developers/build/differences/#using-eth-in-contracts
- https://community.optimism.io/docs/developers/build/differences/#accessing-the-latest-l1-block-number

QA : l2gas 是如何获取L2上的gas的

QA ： finalizeDeposit传输零地址的意思



### 用户质押

```solidity
  
 function depositETH(uint32 _l2Gas, bytes calldata _data) external payable onlyEOA {
        _initiateETHDeposit(msg.sender, msg.sender, _l2Gas, _data);
    }
    
function _initiateETHDeposit(
        address _from,
        address _to,
        uint32 _l2Gas,
        bytes memory _data
    ) internal {
        bytes memory message = abi.encodeWithSelector(
            IL2ERC20Bridge.finalizeDeposit.selector,
            address(0),
            Lib_PredeployAddresses.OVM_ETH,
            _from,      // 从L1提取存款的帐户 msg.sender
            _to,        // L2上的存款账户 
            msg.value,
            _data
        );

        // Send calldata into L2
        sendCrossDomainMessage(l2TokenBridge, _l2Gas, message);
        emit ETHDepositInitiated(_from, _to, msg.value, _data);
    }

```

```solidity
//finalizeDeposit 完成从L1到L2的存款，并将资金记入该L2代币的接收者余额。如果该调用不是来自L1StandardTokenBridge中的相应存款，则该调用将失败。这个只能跨链账户去调用,将相同数量的金额存入到L2的账户
function finalizeDeposit(
        address _l1Token,  // 用于调用的l1令牌的地址
        address _l2Token,  // 用于调用的l2令牌的地址
        address _from,     // 从L2提取存款的帐户。
        address _to,       // 接收取款的地址
        uint256 _amount,
        bytes calldata _data
    ) external;
```

用户质押ETH，将钱打入L1桥里，L1桥会构造一个在L2桥上执行的消息（finalizeDeposit），通过L1的跨链信使（L1CrossDomainManager）发送消息给L2桥, CTC合约会将消息等信息hash作为一笔**op交易**记录到CTC合约中。同时我们在过段时间也能知道这笔OP交易打在L1的哪个区块中。

到了这里实际是L1的合约跨链调用L2 的合约， sendCrossDomainMessage(l2TokenBridge, _l2Gas, message);，调用的CrossDomainEnabled（专门做跨链消息传送的合约）

```solidity
// L1StandardBridge
// Send calldata into L2
sendCrossDomainMessage(l2TokenBridge, _l2Gas, message);

function sendCrossDomainMessage(
        address _crossDomainTarget,  // l2TokenBridge地址
        uint32 _gasLimit,
        bytes memory _message
    ) internal {
        getCrossDomainMessenger().sendMessage(_crossDomainTarget, _message, _gasLimit);
    }
```



其实到这里，组装了一个调用消息，是让L2去调用自己的方法，包括L2的合约地址，L2gas和L2的ABI信息（传入了L1的参数），

最终会把组装的调用消息交给跨链合约。

跨链合约主要做了以下事情：

- 将CTC队列目前长度作为nonce 
- 构造跨链calldata（编码的跨链da ta中的m s g.sender应该是L1桥的地址）
- **发送跨链消息（最终出口）**给CanonicalTransactionChain（**L1上的合约**） 合约， 同时会将此消息存储在CanonicalTransactionChain合约上通过enqueue，这样L1上的工作完成，L1保存了这个消息
  - enqueue 会将整个L1存入token的一连串的数据包括发送者，L2目标token合约，交易data keccak256作为一笔交易以及时间戳和当前区块打包成Elements存入CanonicalTransactionChain，并触发TransactionEnqueued事件（此事件由DTL监听）
- ovmCanonicalTransactionChain（预先部署） 合约地址：0x4200000000000000000000000000000000000007

```solidity
// L1CrossDomainMessenger.sol
function sendMessage(
        address _target,
        bytes memory _message,
        uint32 _gasLimit
    ) public {
        address ovmCanonicalTransactionChain = resolve("CanonicalTransactionChain");
        // Use the CTC queue length as nonce
        uint40 nonce = ICanonicalTransactionChain(ovmCanonicalTransactionChain).getQueueLength();

        bytes memory xDomainCalldata = Lib_CrossDomainUtils.encodeXDomainCalldata(
            _target, // L2tokenBridge
            msg.sender,  // L1DomainMessenger
            _message,  // 存款消息
            nonce
        );

        _sendXDomainMessage(ovmCanonicalTransactionChain, xDomainCalldata, _gasLimit);
        
        emit SentMessage(_target, msg.sender, _message, nonce, _gasLimit);
    }
```

```solidity
    function _sendXDomainMessage(
        address _canonicalTransactionChain,
        bytes memory _message,
        uint256 _gasLimit
    ) internal {
        ICanonicalTransactionChain(_canonicalTransactionChain).enqueue(
            Lib_PredeployAddresses.L2_CROSS_DOMAIN_MESSENGER, // 将交易发送到的目标L2合同
            _gasLimit,
            _message
        );
    }
```

将一笔交易添加到队列：

- calldata的数据不要大于50000字节
- L2 tx gas 相关最大100000
- 触发 TransactionEnqueued事件

```solidity
    function enqueue(
        address _target, // L2DomainMessager 
        uint256 _gasLimit,
        bytes memory _data  // 消息数据，包括L2tokenBridge消息，value，from to 等等
    ) external {
        require(
            _data.length <= MAX_ROLLUP_TX_SIZE,
            "Transaction data size exceeds maximum for rollup transaction."
        );

        require(
            _gasLimit <= maxTransactionGasLimit,
            "Transaction gas limit exceeds maximum for rollup transaction."
        );

        require(_gasLimit >= MIN_ROLLUP_TX_GAS, "Transaction gas limit too low to enqueue.");

        if (_gasLimit > enqueueL2GasPrepaid) {
            uint256 gasToConsume = (_gasLimit - enqueueL2GasPrepaid) / l2GasDiscountDivisor;
            uint256 startingGas = gasleft();
            require(startingGas > gasToConsume, "Insufficient gas for L2 rate limiting burn.");

            uint256 i;
            while (startingGas - gasleft() < gasToConsume) {
                i++;
            }
        }
        address sender;
        if (msg.sender == tx.origin) {
            sender = msg.sender;
        } else {
            sender = AddressAliasHelper.applyL1ToL2Alias(msg.sender);
        }

        bytes32 transactionHash = keccak256(abi.encode(sender, _target, _gasLimit, _data));

        queueElements.push(
            Lib_OVMCodec.QueueElement({
                transactionHash: transactionHash,
                timestamp: uint40(block.timestamp), // 当前区块时间戳
                blockNumber: uint40(block.number)  // 当前区块高度
            })
        );
        uint256 queueIndex = queueElements.length - 1;
        emit TransactionEnqueued(sender, _target, _gasLimit, _data, queueIndex, block.timestamp);
    }

```

DTL组件是由typescript写的，TransactionEnqueued事件的处理如下,会解析事件并查出calldata,写入数据库

```typescript
export const handleEventsTransactionEnqueued: EventHandlerSet<
  TransactionEnqueuedEvent,
  null,
  EnqueueEntry
> = {
  getExtraData: async () => {
    return null
  },
  parseEvent: (event) => {
    return {
      index: event.args._queueIndex.toNumber(),
      target: event.args._target,
      data: event.args._data,
      gasLimit: event.args._gasLimit.toString(),
      origin: event.args._l1TxOrigin,
      blockNumber: BigNumber.from(event.blockNumber).toNumber(),
      timestamp: event.args._timestamp.toNumber(),
      ctcIndex: null,
    }
  },
  storeEvent: async (entry, db) => {
    ...
    await db.putEnqueueEntries([entry])
  },
}
```

接着就是L2geth(sequencer)从DTL同步`TransactionEnqueued` 事件，转为交易并执行,这部分在L2geth/rollup/sync_service下面，专门由SequencerLoop执行（前提是配置为不是验证者。）这里同步事件实际还在依赖以太坊上的处理速度 ，但是从L1质押到L2应该不是很平常的动作，无所谓。

L2geth 里面存储着一个rollupclient,用来对DTL进行Http请求的。请求注册的路由在DTL/service.ts ， Rollup节点和DTL建立HTTP连接。

```go
//SequencerLoop 是在 sequencer 模式下运行的轮询循环。它排序
//交易，然后更新 EthContext。

func (s *SyncService) SequencerLoop() {
  ...
	s.sequence();
  ...
}
```

执行交易主要由s.applyTransaction(tx)实现，会调用applyIndexedTransaction，交易的来源是指L1 batch，或者是sequencer同步DTL中的Transactionenqueued事件的交易。

```go
func (s *SyncService) syncQueueTransactionRange(start, end uint64) error {
	log.Info("Syncing enqueue transactions range", "start", start, "end", end)
	for i := start; i <= end; i++ {
		tx, err := s.client.GetEnqueue(i)
		if err != nil {
			return fmt.Errorf("Canot get enqueue transaction; %w", err)
		}
		if err := s.applyTransaction(tx); err != nil {
			return fmt.Errorf("Cannot apply transaction: %w", err)
		}
	}
	return nil
}
```

applyTransaction分为 applyTransactionToTip 和applyIndexedTransaction ，两种处理方式不一样，下面来单独分析下：现在我还不能确定tx.meta到底有没有赋值 ,有赋值，从syncTransactionRange看

applyTransactionToTip：

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