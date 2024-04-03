RunNextSequencerAction 

-  d.StartBuildingBlock(ctx)

  - d.l1OriginSelector.FindL1Origin ，找出在需要在哪个L1块上构建，要么是l2Head.L1Origin，要么是l2Head.L1Origin+1 

  - d.attrBuilder.PreparePayloadAttributes(fetchCtx, l2Head, l1Origin.ID()) 

    - ba.l2.SystemConfigByL2Hash(ctx, l2Parent.Hash)， 从上个L2区块中获取到L1info信息并将其转为SystemConfig（rollup 配置）， 这个可以通过 L1 系统配置事件进行更改 

    ```go
    type SystemConfig struct {
    	// BatcherAddr identifies the batch-sender address used in batch-inbox data-transaction filtering.
    	BatcherAddr common.Address `json:"batcherAddr"`
    	// Overhead identifies the L1 fee overhead, and is passed through opaquely to op-geth.
    	Overhead Bytes32 `json:"overhead"`
    	// Scalar identifies the L1 fee scalar, and is passed through opaquely to op-geth.
    	Scalar Bytes32 `json:"scalar"`
    	// GasLimit identifies the L2 block gas limit
    	GasLimit uint64 `json:"gasLimit"`
    	// More fields can be added for future SystemConfig versions.
    }
    ```

  - 组装序列化的 L1-info 属性事务，这必须是L2区块的第一笔交易,下面就是完整的payload属性

    ```go
    eth.PayloadAttributes{
    		Timestamp:             hexutil.Uint64(nextL2Time),
    		PrevRandao:            eth.Bytes32(l1Info.MixDigest()),
    		SuggestedFeeRecipient: predeploys.SequencerFeeVaultAddr, // TODO 
    		Transactions:          txs,
    		NoTxPool:              true,
    		GasLimit:              (*eth.Uint64Quantity)(&sysConfig.GasLimit),
    	}
    ```

  - 判断新区块的L1Origin是否和上一个区块的不同。如果不同,说明进入了新的L1 epoch。

  - 如果进入新的epoch,需要从L1获取该区块的所有交易收据(receipts),扫描里面的用户存款事件。

  - 同时更新系统配置(sysConfig)。

  - 如果L1Origin没有变化,则复用上一个区块的信息,序列号+1。

  - 在任何情况下,都需要验证新区块的L1Origin必须与epoch的hash匹配,不能冲突。

  - 其中有段代码：根据扫描了L1的log ，根据topic 对照此 `TransactionDeposited(address,address,uint256,bytes)` 函数选择器，将其unmarshal 出来并转成depositTx ,详细的信息被解析出来后，需要在L2执行，算是质押交易的下半段内容（这是如何做到原子性的 TODO），如果有L1区块收据里有系统更新的就把信息更新到sysCfg

    ```GO
    DeriveDeposits(receipts, ba.cfg.DepositContractAddress)
    
    func UserDeposits(receipts []*types.Receipt, depositContractAddr common.Address) ([]*types.DepositTx, error) {
    	var out []*types.DepositTx
    	var result error
    	for i, rec := range receipts {
    		if rec.Status != types.ReceiptStatusSuccessful {
    			continue
    		}
    		for j, log := range rec.Logs {
    			if log.Address == depositContractAddr && len(log.Topics) > 0 && log.Topics[0] == DepositEventABIHash {
    				dep, err := UnmarshalDepositLogEvent(log)
    				if err != nil {
    					result = multierror.Append(result, fmt.Errorf("malformatted L1 deposit log in receipt %d, log %d: %w", i, j, err))
    				} else {
    					out = append(out, dep)
    				}
    			}
    		}
    	}
    	return out, result
    }
    ```

- 下一个 L2 块时间戳超出了 Sequencer 漂移阈值，那么我们必须生成空块（L1 信息存款和任何用户存款除外）， 然后开始构建上面准备好的Attributes 通过 d.engine.StartPayload(ctx, l2Head, attrs, false)

  ```go
  d.engine.StartPayload  
  eng.ForkchoiceUpdate(ctx, &fc, attrs)
  直接调用 op-geth 里面的 buildpayload ,将sequener搞到的depositTx，以及L2交易池子中的交易一起打包成区块，
  Op-node层就不需要保存区块信息了，区块状态和信息都在L2 层，（TODO，我只是看到了打包成区块，没有见到共识插入数据库）
  
  ```

到此为止 L2将sequencer提供的L1相关的质押交易提交给了L2 执行，并生成L2区块，保存在L2链上，接下来sequner 会把L2payaload 发送到L1，这样一来 其他的验证者是可以根据l2数据来验证L2链的状态是否是真实的（不过为什么要7天呢 TODO），请参考下面PublishL2Payload

## PublishL2Payload

上面所有的基本都是进入的sequencerCh， 这PublishL2Payload即将进入stepReqCh，里面的主要方法是：s.derivation.Step(context.Background())， 里面的enqinequeque到底干嘛的