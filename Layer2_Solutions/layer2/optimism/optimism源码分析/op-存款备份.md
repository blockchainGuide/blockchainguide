```solidi
 function _isOptimismMintableERC20(address _token) internal view returns (bool) {
        return
            ERC165Checker.supportsInterface(_token, type(ILegacyMintableERC20).interfaceId) ||
            ERC165Checker.supportsInterface(_token, type(IOptimismMintableERC20).interfaceId);
    }
    这个判断是不是op可以mint的ERC20
    
    
```

当用户通过optimal 合约发出质押交易事件（TransactionDeposited），接下来会由OP RollUp client 来处理

```GO
// UserDeposits transforms the L2 block-height and L1 receipts into the transaction inputs for a full L2 block
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

实际是有协程一直在做这个事

```

sequencer 从L1中获取到质押事件，再将其组成质押交易，同时构造attr(需要详细了解），然后将这些必要数据提交给引擎来构建包含attr的区块。StartPayload

ConfirmPayload 完成块的构建， 搞了半天构建区块都是seqencer在干， payload 就是区块，sequncer 最终也会调用finalizeAndAssemble,就是不清楚是走的l2的哪个共识引擎。

sequencer组装完毕后进行发布L2payload,通过Op-node 的gossip功能发布到本地，通过channel s.unsafeL2Payloads，来接收，最后扔到一个engine_equeqe 队列里了， 接着执行 reqStep()

sequencer实际是要从L1获取质押事件，并将交易在L2执行，并打包到区块中。 



接下来batcher submitter登场： 他会定期从L2区块中将交易数据以打包的形式组装到交易 并提交到合约中

batch submitter 走的是op-batcher的组件，需要l2,l1,rollup-node 的客户端信息

```go
// 他会有个轮询间隔，会将L2状态信息（交易）提交给L1
func (l *BatchSubmitter) publishStateToL1(queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData], drain bool) {
  l.sendTransaction(txdata, queue, receiptsCh)
}


```

这个后面会发交易提到BatchInboxAddress， 

Op-proposer 提交proof 到l200contractx x x 地址，然后就等待挑战期了。

o p-geth 保存了完整的状态数据



将L2交易收到后再提交给L1，然后得到L1 执行的receipt，进行处理，结束，那么问题是L1 执行完干嘛呢？？？

op-geth 应该是proposer 要执行的 













