op-node主要有以下几件事：

- 从L1导出区块并将其驱动到引擎中来同步区块
- 如果启用了备份不安全同步客户端，请启动其事件循环



下面将主要分析event loop 做的事情：

```go
func (s *Driver) eventLoop() {
  ...
  select {
		case <-sequencerCh:
			payload, err := s.sequencer.RunNextSequencerAction(ctx)
			if err != nil {
				s.log.Error("Sequencer critical error", "err", err)
				return
			}
			if s.network != nil && payload != nil {
				// Publishing of unsafe data via p2p is optional.
				// Errors are not severe enough to change/halt sequencing but should be logged and metered.
				if err := s.network.PublishL2Payload(ctx, payload); err != nil {
					s.log.Warn("failed to publish newly created block", "id", payload.ID(), "err", err)
					s.metrics.RecordPublishingError()
				}
			}
			planSequencerAction() // schedule the next sequencer action to keep the sequencing looping
		case <-altSyncTicker.C:
			// Check if there is a gap in the current unsafe payload queue.
			ctx, cancel := context.WithTimeout(ctx, time.Second*2)
			err := s.checkForGapInUnsafeQueue(ctx)
			cancel()
			if err != nil {
				s.log.Warn("failed to check for unsafe L2 blocks to sync", "err", err)
			}
		case payload := <-s.unsafeL2Payloads:
			s.snapshot("New unsafe payload")
			s.log.Info("Optimistically queueing unsafe L2 execution payload", "id", payload.ID())
			s.derivation.AddUnsafePayload(payload)
			s.metrics.RecordReceivedUnsafePayload(payload)
			reqStep()

		case newL1Head := <-s.l1HeadSig:
			s.l1State.HandleNewL1HeadBlock(newL1Head)
			reqStep() // a new L1 head may mean we have the data to not get an EOF again.
		case newL1Safe := <-s.l1SafeSig:
			s.l1State.HandleNewL1SafeBlock(newL1Safe)
			// no step, justified L1 information does not do anything for L2 derivation or status
		case newL1Finalized := <-s.l1FinalizedSig:
			s.l1State.HandleNewL1FinalizedBlock(newL1Finalized)
			s.derivation.Finalize(newL1Finalized)
			reqStep() // we may be able to mark more L2 data as finalized now
		case <-delayedStepReq:
			delayedStepReq = nil
			step()
		case <-stepReqCh:
			s.metrics.SetDerivationIdle(false)
			s.log.Debug("Derivation process step", "onto_origin", s.derivation.Origin(), "attempts", stepAttempts)
			err := s.derivation.Step(context.Background())
			stepAttempts += 1 // count as attempt by default. We reset to 0 if we are making healthy progress.
			if err == io.EOF {
				s.log.Debug("Derivation process went idle", "progress", s.derivation.Origin())
				stepAttempts = 0
				s.metrics.SetDerivationIdle(true)
				continue
			} else if err != nil && errors.Is(err, derive.ErrReset) {
				// If the pipeline corrupts, e.g. due to a reorg, simply reset it
				s.log.Warn("Derivation pipeline is reset", "err", err)
				s.derivation.Reset()
				s.metrics.RecordPipelineReset()
				continue
			} else if err != nil && errors.Is(err, derive.ErrTemporary) {
				s.log.Warn("Derivation process temporary error", "attempts", stepAttempts, "err", err)
				reqStep()
				continue
			} else if err != nil && errors.Is(err, derive.ErrCritical) {
				s.log.Error("Derivation process critical error", "err", err)
				return
			} else if err != nil && errors.Is(err, derive.NotEnoughData) {
				stepAttempts = 0 // don't do a backoff for this error
				reqStep()
				continue
			} else if err != nil {
				s.log.Error("Derivation process error", "attempts", stepAttempts, "err", err)
				reqStep()
				continue
			} else {
				stepAttempts = 0
				reqStep() // continue with the next step if we can
			}
		case respCh := <-s.stateReq:
			respCh <- struct{}{}
		case respCh := <-s.forceReset:
			s.log.Warn("Derivation pipeline is manually reset")
			s.derivation.Reset()
			s.metrics.RecordPipelineReset()
			close(respCh)
		case resp := <-s.startSequencer:
			unsafeHead := s.derivation.UnsafeL2Head().Hash
			if !s.driverConfig.SequencerStopped {
				resp.err <- errors.New("sequencer already running")
			} else if !bytes.Equal(unsafeHead[:], resp.hash[:]) {
				resp.err <- fmt.Errorf("block hash does not match: head %s, received %s", unsafeHead.String(), resp.hash.String())
			} else {
				s.log.Info("Sequencer has been started")
				s.driverConfig.SequencerStopped = false
				close(resp.err)
				planSequencerAction() // resume sequencing
			}
		case respCh := <-s.stopSequencer:
			if s.driverConfig.SequencerStopped {
				respCh <- hashAndError{err: errors.New("sequencer not running")}
			} else {
				s.log.Warn("Sequencer has been stopped")
				s.driverConfig.SequencerStopped = true
				respCh <- hashAndError{hash: s.derivation.UnsafeL2Head().Hash}
			}
		case <-s.done:
			return
}
```

### 发布新创建的区块

