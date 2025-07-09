```GO
		case <-ticker.C:
			if err := l.loadBlocksIntoState(l.shutdownCtx); errors.Is(err, ErrReorg) {
				err := l.state.Close()
				if err != nil {
					l.log.Error("error closing the channel manager to handle a L2 reorg", "err", err)
				}
				l.publishStateToL1(queue, receiptsCh, true)
				l.state.Clear()
				continue
			}
			l.publishStateToL1(queue, receiptsCh, false)
```

这段代码将所有l2区块加载到channelManager中，这是个通道管理器，然后再将其切成帧发布到L1



```go
// publishTxToL1 submits a single state tx to the L1
func (l *BatchSubmitter) publishTxToL1(ctx context.Context, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) error {
	// send all available transactions
	l1tip, err := l.l1Tip(ctx)
	if err != nil {
		l.log.Error("Failed to query L1 tip", "error", err)
		return err
	}
	l.recordL1Tip(l1tip)

	// Collect next transaction data
	txdata, err := l.state.TxData(l1tip.ID())
	if err == io.EOF {
		l.log.Trace("no transaction data available")
		return err
	} else if err != nil {
		l.log.Error("unable to get tx data", "err", err)
		return err
	}

	l.sendTransaction(txdata, queue, receiptsCh)
	return nil
}
```

```GO
func (l *BatchSubmitter) sendTransaction(txdata txData, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) {
	// Do the gas estimation offline. A value of 0 will cause the [txmgr] to estimate the gas limit.
	data := txdata.Bytes()
	intrinsicGas, err := core.IntrinsicGas(data, nil, false, true, true, false)
	if err != nil {
		l.log.Error("Failed to calculate intrinsic gas", "error", err)
		return
	}

	candidate := txmgr.TxCandidate{
		To:       &l.Rollup.BatchInboxAddress, // 常规的EOA地址
		TxData:   data,
		GasLimit: intrinsicGas,
	}
	queue.Send(txdata, candidate, receiptsCh)
}
```

最终op-batcher根据L1Id 将相关的交易全部提交到了BatchInboxAddress
