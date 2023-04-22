![image-20221118125400482](/Users/carver/Library/Application Support/typora-user-images/image-20221118125400482.png)

用户需要在bsc上的ERC20代币绑定到zkbnb 上，无法将代币在上面流通

支持用户在Layer1和Layer2之间进行NFT和FT的传输

用户将代币存储在BSC上的rollup合约上进入到ZKroullup,ZkBNB监视器将跟踪存款并将其作为第二层交易提交，一旦提交人验证了交易，用户就可以在其账户上获得资金，他们可以通过将交易发送给提交人进行处理来开始交易。

用户可以通过向网络发送签名的交易来将任何数量的资金转移到ZKBNB上的任何帐户。

用户启动提款交易，资金将在 ZkBNB 上烧毁。一旦对下一批中的事务进行了rolliup，相关数量的token将从rollup contract解锁到目标帐户。



所有的买入/卖出报价、NFT/Collection的元数据、媒体资源、账户配置文件都存储在NFT市场的后端，只有竞争的Hash、所有权、creatorTreasuryRate和其他字段记录在ZkBNB上。为了鼓励价格发现，**任何人都可以在市场上购买/出售报价，而无需支付任何费用**，因为报价缓存在后端，而不是发送到ZkBNB。一旦报价匹配，由买入和卖出报价组成的AtomicMatch交易将发送给ZkBNB，以实现交易。用户还可以通过发送取消报价事务以禁用后端缓存的报价来手动取消报价



ZkBNB 上的每个帐户都有其简称，用户可以使用它来存储资金并接收任何加密货币、令牌或 NFT。

ZkBNB本机支持ECDSA签名并遵循EIP712签名结构，这意味着大多数以太坊钱包可以无缝支持ZkBNB。BSC用户没有额外的努力来利用ZkBNB。



与大多数将状态树放入内存的汇总解决方案不同，BAS-SMT 是一种用于持久数据的版本化、快照表(不可变)稀疏树。BAS-SMT 是 ZkBNB 大规模应用的关键因素https://github.com/bnb-chain/zkbnb-smt/



ZkBNB Crypto是描述证明电路的库。一旦ZK汇总节点有了足够的事务，它就将它们聚合成一批，并为证明电路编译输入，以编译成一个简洁的ZK证明 https://github.com/bnb-chain/zkbnb-crypto



## ZKbnb contract

这是layer 2的入口和出口。

zkBNB合约提供以下特性：

L1安全：ZkBNBVerifier 协议可以验证 **Layer2**生成的 SNARK 证明(简洁的非交互式知识论证) ，从而证明汇总块中每个事务的有效性。所以 ZkBNB 和 BSC 共享同样的安全系统。由于 zkSNARK 的证明，安全性是由密码学保证的。用户不必信任任何第三方，也不必为了防止欺诈而不断监视 Rollup 块。

L1到L2沟通：ZkBNB合同公开了几个支持BNB的接口，在BSC或ZkBNA上创建的BEP20/BEP721可以自由地流向ZkBNC。

L2到L1的沟通：每个rollup L2块包括一批需要由 L1 合约处理的 L2操作

BSC上的完全退出：如果用户认为自己的交易受到ZkBNB的审查，可以通过L1智能合约请求提取资金

验证器将块从L2提交到L1，这些块将存储在L1上，以供以后验证。提交一个块包括以下步骤：

- 必须是治理合约中的validator(怎么成为validator
- 上一个块必须是存储在合约中的最新快 
- commitOneBlock
  - 校验number和timestamp
  - 检查链上操作
    - 交易类型RegisterZNS CreatePair UpdatePairRate Deposit DepositNft Withdraw WithdrawNft FullExit FullExitNft
    - 这些交易类型是定义在layer 2上的 
      - 注册ZNS名称
      - 为 L2上的token swap创建token Pair
      - 令牌对的更新费率
      - 将代币从L1存入L2
      - 将NFT从L1存放到L2
      - 从 L2提取令牌到 L1，向 L2发送请求
      - 将 NFT 从 L2撤回到 L1，向 L2发送请求
      - 从L2向L1请求退出令牌，向L1发送请求
      - 请求从L2到L1退出NFT，向L1发送请求
    - 最终生成一个可执行的操作hash 和处理的优先级操作
  - 为验证证明创建区块承诺（只跟前一个区块和新区块有关，需要上个区块的state root ）
  - 最后返回这些验证信息（需要详细列出这些信息）

CommitBlock包含块信息、事务数据和事务执行后的状态根。块信息包含时间戳、blockNumber和blockSize。二级事务打包在CommitBlockInfo.publicData中
