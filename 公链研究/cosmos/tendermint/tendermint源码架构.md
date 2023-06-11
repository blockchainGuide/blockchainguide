consensus 

OnStart -> timeoutRoutine start  超时处理

​			-> go cs.receiveRoutine(0) 开启接收协程





## cs.receiveRoutine
```GO
for {
		select {
		case <-cs.txNotifier.TxsAvailable():
			cs.handleTxsAvailable()

		case mi = <-cs.peerMsgQueue:
			if err := cs.wal.Write(mi); err != nil {
				cs.Logger.Error("failed writing to WAL", "err", err)
			}

			// handles proposals, block parts, votes
			// may generate internal events (votes, complete proposals, 2/3 majorities)
			cs.handleMsg(mi)

		case mi = <-cs.internalMsgQueue:
			err := cs.wal.WriteSync(mi) // NOTE: fsync
			if err != nil {
				panic(fmt.Sprintf(
					"failed to write %v msg to consensus WAL due to %v; check your file system and restart the node",
					mi, err,
				))
			}
		...
			// handles proposals, block parts, votes
			cs.handleMsg(mi)

		case ti := <-cs.timeoutTicker.Chan(): // tockChan:
			if err := cs.wal.Write(ti); err != nil {
				cs.Logger.Error("failed writing to WAL", "err", err)
			}

			// if the timeout is relevant to the rs
			// go to the next step
			cs.handleTimeout(ti, rs)
	}
}
```

一共做3件事：

- 处理外部接收的消息
- 处理内部接收的消息
- 处理超时消息



## 共识流程

```markdown
(谁，在什么时候做) createProposalBlock ->

​								-> （从交易池中获取交易） blockExec.mempool.ReapMaxBytesMaxGas(maxDataBytes, maxGas)

​								-> 组装区块（构建基本区块，使用状态数据填充）



createProposalBlock -> NewProposal(根据上面区块以及高度round)->SignProposal-> SetProposalAndBlock (ProposalMessage和BlockPartMessage （如何处理）)->signVote(对区块投票,塞入VoteMessage消息)->	
```



## 编程技巧

- 对于一个通知通道，为什么还要用接口封装

  ```go
  type txNotifier interface {
  	TxsAvailable() <-chan struct{}
  }
  
  ```

  





