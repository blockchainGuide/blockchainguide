```go
type Header struct {
	ParentHash  common.Hash    // 前一个的区块哈希
	UncleHash   common.Hash    // 挖出这个块的矿工地址，因为挖出块所奖励的ETH就会发放到这个地址
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	Root        common.Hash    // 状态树的根节点哈希
	TxHash      common.Hash    // 交易树的根哈希，由本区块所有交易的交易哈希算出
	ReceiptHash common.Hash    //收据树的哈希
	Bloom       Bloom          // 布隆过滤器，快速定位日志是否在这个区块中
	Difficulty  *big.Int       // 当前工作量证明（Pow）算法的复杂度
	Number      *big.Int       // 区块号
	GasLimit    uint64         // 每个区块Gas的消耗上限
	GasUsed     uint64        // 当前区块所有交易使用的Gas之和
	Time        uint64        // 区块产生的时间戳
	Extra       []byte         
	MixDigest   common.Hash    // 挖矿得到的Pow算法证明的摘要，也就是挖矿的工作量证明
	Nonce       BlockNonce      // 挖矿找到的满足条件的值

	// BaseFee was added by EIP-1559 and is ignored in legacy headers.
	BaseFee *big.Int `json:"baseFeePerGas" rlp:"optional"`
}
```





> https://ethbook.abyteahead.com/ch4/transroot.html
>
> 公众号：区块链技术栈

