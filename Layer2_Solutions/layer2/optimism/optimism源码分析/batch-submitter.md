> 定期从L2区块中将交易数据以打包的形式组装到交易

- 打包批量交易 `txBatch` 提交到 L1 的 `CTC` 合约；
- 打包批量状态 `stateBatch` 提交到 L1 的`StateCommitmentChain.sol`
- 之后这些交易进入`等待挑战`窗口，挑战方式就是`欺诈证明`；