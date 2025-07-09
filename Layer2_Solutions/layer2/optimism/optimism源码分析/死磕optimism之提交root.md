```go
func (l *L2OutputSubmitter) loop() {
	defer l.wg.Done()

	ctx := l.ctx

	ticker := time.NewTicker(l.pollInterval)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			output, shouldPropose, err := l.FetchNextOutputInfo(ctx)
			if err != nil {
				break
			}
			if !shouldPropose {
				break
			}

			cCtx, cancel := context.WithTimeout(ctx, 10*time.Minute)
			if err := l.sendTransaction(cCtx, output); err != nil {
				l.log.Error("Failed to send proposal transaction", "err", err)
				cancel()
				break
			}
			l.metr.RecordL2BlocksProposed(output.BlockRef)
			cancel()

		case <-l.done:
			return
		}
	}
}
```





```GO
func (n *nodeAPI) OutputAtBlock(ctx context.Context, number hexutil.Uint64) (*eth.OutputResponse, error) {
	recordDur := n.m.RecordRPCServerRequest("optimism_outputAtBlock")
	defer recordDur()

	ref, status, err := n.dr.BlockRefWithStatus(ctx, uint64(number))
	if err != nil {
		return nil, fmt.Errorf("failed to get L2 block ref with sync status: %w", err)
	}

	head, err := n.client.InfoByHash(ctx, ref.Hash)
	if err != nil {
		return nil, fmt.Errorf("failed to get L2 block by hash %s: %w", ref, err)
	}
	if head == nil {
		return nil, ethereum.NotFound
	}

	proof, err := n.client.GetProof(ctx, predeploys.L2ToL1MessagePasserAddr, []common.Hash{}, ref.Hash.String())
	if err != nil {
		return nil, fmt.Errorf("failed to get contract proof at block %s: %w", ref, err)
	}
	if proof == nil {
		return nil, fmt.Errorf("proof %w", ethereum.NotFound)
	}
	// make sure that the proof (including storage hash) that we retrieved is correct by verifying it against the state-root
	if err := proof.Verify(head.Root()); err != nil {
		n.log.Error("invalid withdrawal root detected in block", "stateRoot", head.Root(), "blocknum", number, "msg", err)
		return nil, fmt.Errorf("invalid withdrawal root hash, state root was %s: %w", head.Root(), err)
	}

	var l2OutputRootVersion eth.Bytes32 // it's zero for now
	l2OutputRoot, err := rollup.ComputeL2OutputRootV0(head, proof.StorageHash)
	if err != nil {
		n.log.Error("Error computing L2 output root, nil ptr passed to hashing function")
		return nil, err
	}

	return &eth.OutputResponse{
		Version:               l2OutputRootVersion,
		OutputRoot:            l2OutputRoot,
		BlockRef:              ref,
		WithdrawalStorageRoot: proof.StorageHash,
		StateRoot:             head.Root(),
		Status:                status,
	}, nil
}
```



// proposer 获取到了相关的Proof 和root 之后，提交给L2OutputOracle 合约

```GO


func (l *L2OutputSubmitter) sendTransaction(ctx context.Context, output *eth.OutputResponse) error {
	data, err := l.ProposeL2OutputTxData(output)
	if err != nil {
		return err
	}
	receipt, err := l.txMgr.Send(ctx, txmgr.TxCandidate{
		TxData:   data,
		To:       &l.l2ooContractAddr,
		GasLimit: 0,
	})
	if err != nil {
		return err
	}
	if receipt.Status == types.ReceiptStatusFailed {
		l.log.Error("proposer tx successfully published but reverted", "tx_hash", receipt.TxHash)
	} else {
		l.log.Info("proposer tx successfully published", "tx_hash", receipt.TxHash)
	}
	return nil
}
```



## QA

https://eips.ethereum.org/EIPS/eip-1186