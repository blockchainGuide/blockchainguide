- AZTEC 放出了[他们的 zk-zk rollup 的代码](https://medium.com/@tompocock/zk-zk-rollup-code-release-i-97216347b3bc)，为单条曲线上的 SNARK 实现递归
- [基于账户的匿名 Rollup 的一种规范](https://ethresear.ch/t/account-based-anonymous-rollup/6657/12)
- [状态通道如何适应 Layer-2 的后 rollup 时代](https://blog.statechannels.org/do-we-still-need-state-channels/)：即时确定性、不必要使用第三方，可执行任意程序
- [zk-rollups 能帮助区块链做什么？](https://bankless.substack.com/p/can-fancy-new-cryptography-scale)
- [通过开发 Optimistic Rollup 来理解它](https://gourmetcrypto.substack.com/p/optimistic-rollups-from-the-bottom)

- Hermez zk rollup [通过拍卖区块生产者职位](https://blog.hermez.io/proof-of-donation-keeping-hermez-permissionless-and-giving-back-to-the-community/)来为公共品提供资金
- Celer 的 [State Guardian Network 测试网启动](https://blog.celer.network/2020/08/10/state-guardian-network-beta-testnet-launches/)
- Livepeer 的[概率性微支付准 Layer-2 方案](https://forum.livepeer.org/t/how-are-livepeers-probabilistic-micropayments-holding-up-with-these-ethereum-high-gas-prices/1127)
- ENS [尝试用 optimistic rollup 类似方案来解决 DNS 域名声明问题](https://discuss.ens.domains/t/improving-gas-efficiency-of-dns-domain-claims-with-optimistic-verification/189)
- [Rollup 重放保护](https://ethresear.ch/t/zkrollup-optimistic-rollup-replay-protection/6999)
- [USDT 上线 OMG 二层网络](https://tether.to/tether-usdt-integration-live-on-omg-network/)
- [Golem 选择 zk sync 作为扩展方案](https://blog.golemproject.net/zksync/)
- dYdX 宣布[他们与 StarkWare 合作，计划在第四季度实现扩展](https://integral.dydx.exchange/scaling-with-starkware/)
- StarkWare 的 [Cairo](https://medium.com/@StarkWare/hello-cairo-3cb43b13b209)：通用计算的 STARK 证明器
- [Rollup 数据压缩技术](https://ethresear.ch/t/rollup-diff-compression/7933)
- 使用该压缩技术后，optimistic rollup 实例 Fuel [基准测试测得代币转账的 2500 TPS](https://twitter.com/fuellabs_/status/1300468388695810050)
- [POA 迁移到 optimistic rollup VM 方案](https://ethresear.ch/t/poa-transition-to-optimistic-rollup-vm/7983)
- [USDC 在 Matic 上的运行演示](https://twitter.com/petejkim/status/1306957817702526976)
- Nick Johnson：[一种以太坊 layer2 的通用桥](https://medium.com/the-ethereum-name-service/a-general-purpose-bridge-for-ethereum-layer-2s-e28810ec1d88)
- [Arbitrum 导入了 Uniswap](https://learnblockchain.cn/article/1712) 到其 rollup 中。每笔交易只需消耗 2k gas，可以在 Kovan 测试网上尝试使用
- Celer 的 [State Guardian Network 第一阶段启动](https://blog.celer.network/2020/11/09/celer-state-guardian-network-launches-on-mainnet/)
- Fuel：[让以太坊成为资产创造和结算层](https://fuellabs.medium.com/the-future-of-l1-ethereum-5ec5d9a01c10)
- Loopring 的 [zkrollup 智能合约钱包](https://medium.com/loopring-protocol/loopring-wallet-ethereum-unleashed-ac4173f940a5)，安卓 app 已可使用，而且在钱包中持有余额还会获得奖励
- Hermez 的[大规模迁移机制](https://blog.hermez.io/hermez-massive-migrations-mechanism/)，防止 rollup 的中心化
- Loopring 的 [AMM 已登陆他们的 zkrollup](https://medium.com/loopring-protocol/looprings-zkrollup-amm-is-live-2f8251cd0fcd)（虽然现在仅限于 LRC/ETH）
- Golem 的最新版本将使用 [Zksync 来转账](https://blog.golemproject.net/new-golem-alpha-iii-reveal/)
- Deversifi 的 validium 已升级到 [StarkEx v2](https://medium.com/starkware/starkex-2-0-is-now-live-on-mainnet-d8768b860bd4)，包含了快速取款功能和一种图灵完备的 STARK 证明语言，虽然不标准的 ERC20 token（如 OMG、USDT）[还要几个月才能用](https://blog.deversifi.com/a-tale-of-intrigue-non-standard-erc20-tokens-and-platform-upgrades/)
- Optimism [测试网加入欺诈证明](https://medium.com/ethereum-optimism/fraud-proof-security-drill-will-you-be-my-1-of-n-654e78c5ee1c)，同时为所有发现了他们的欺诈交易的人提供 3.2 ETH 的奖金
- Vitalik 的 [rollup 指南](https://vitalik.ca/general/2021/01/05/rollup.html) ， [中文翻译版本](https://learnblockchain.cn/article/1977)
- [Optimism 软启动](https://medium.com/ethereum-optimism/mainnet-soft-launch-7cacc0143cd5)
- [Synthetix：转向 optimistic ethereum 的 5 个阶段](https://blog.synthetix.io/the-optimistic-ethereum-transition/)
- [你在 Loopring 上可以给任意的以太坊地址转账了（即使对方还没有 zk-rollup 账户）](https://twitter.com/loopringorg/status/1349427566319456264)
- [Connext Vector 推出了跨 Layer-2 路由网络](https://medium.com/connext/vector-0-1-0-mainnet-release-9496ae52c422)
- [Loopring 的 zk-rollup 支持快速取款到主网](https://twitter.com/daniel_loopring/status/1351467899274240000?)
- Celer 在其状态通道游戏中已获得 [100 万用户](https://blog.celer.network/2021/01/17/celer-network-2021-update-to-a-better-future/)
- 来自 StarkWare 的 [DeFi Pooling](https://medium.com/starkware/defi-pooling-1332ddebff21)，放在 layer-2 上的、类似于 yearn 的金库，可帮助用户节省 gas
- Arbitrum [为许多应用放出了测试网](https://offchain.medium.com/arbitrum-rollout-demo-defi-ecosystem-built-on-arbitrums-layer-2-testnet-e3bee7df1fc3)
- [optimistic rollups](https://research.paradigm.xyz/rollups) 指南及 [Optimism](https://research.paradigm.xyz/optimism) 指南
- [Hop 跨 rollup 代币桥](https://ethresear.ch/t/hop-send-tokens-across-rollups/8581)
- Cartesi Rollup：[使用 TrueBit 类型的 验证/欺诈证明 游戏](https://medium.com/cartesi/scalable-smart-contracts-on-ethereum-built-with-mainstream-software-stacks-8ad6f8f17997)
- Aztec 和 Starkware [为 rollup 代码发布了 Polaris 证明器许可](https://medium.com/aztec-protocol/introducing-polaris-d4eb0c9da1b4)
- 通往 [StarkNet](https://medium.com/starkware/on-the-road-to-starknet-a-permissionless-stark-powered-l2-zk-rollup-83be53640880) 之路：StarkWare 将在 2021 年末推出的 zk-rollup，使用 [Cairo](https://www.cairo-lang.org/playground/) 编写应用
- Celer 的 [cBridge](https://blog.celer.network/2021/02/15/celer-cbridge-fast-and-low-cost-value-transfer-network-for-an-interconnected-layer2-and-layer1-future)：一种跨 layer-2 的网络，用于在 EVM 链之间实现便宜、即时的价值转移。以准备好生产版本，UI 正在开发中
- Loopring 的 rollup dex 问世时间不长，但总交易额已突破 [5 亿美元](https://twitter.com/loopringorg/status/1363195374723284992)，现在每天交易额都有 3000 万美元
- dYdX [为他们的全仓保证金永续产品推出了 Starkware zkrollup](https://dydx.exchange/blog/alpha)；该产品在主网上还是内测版，预计在几周内公开发布
- Optimism [计划在三月推出正式版](https://medium.com/ethereum-optimism/dope-hires-moar-mainnet-in-march-174fa8966361)
- Deversifi 在其 Validium 实现中增加了[互换功能](https://twitter.com/deversifi/status/1363804660847493122)
- [高效证明 ORU calldata 的机制](https://ethresear.ch/t/efficient-mechanism-to-prove-oru-calldata/8770)
- [一种混合了 optimistic/zkrollup 的路线](https://ethresear.ch/t/a-pre-consensus-mechanism-to-secure-instant-finality-and-long-interval-in-zkrollup/8749)
- [Hermez zk-rollup 以上线 Rinkeby 测试网](https://blog.hermez.io/hermez-testnet-is-now-public/)
- [跨 rollup 的 dex 设计](https://ethresear.ch/t/cross-rollup-dex-with-smart-contracts-only-on-the-destination-side/8778)
- [解释 StarkEx 的条件性支付如何实现从 L2 到 L1 的快速取款](https://medium.com/starkware/conditional-transfers-the-key-to-interoperability-2e1de044fb65)
- [以实际案例解释 L2 与侧链的区别](https://gourmetcrypto.substack.com/p/layer-2-for-beginners)
- [zkmoney](https://medium.com/aztec-protocol/launching-aztec-2-0-rollup-ac7db8012f4b)：Aztec 启动 zk-zkrollup，支持隐私交易，当前的单笔交易上限为 1 ETH
- [Aztec：使用 TurboPLONK 的 zk-zkrollup 架构](https://medium.com/aztec-protocol/aztecs-zk-zk-rollup-looking-behind-the-cryptocurtain-2b8af1fca619)

- [Arbitrum 的进展](https://medium.com/offchainlabs/arbitrum-updates-buckle-up-80483d71718c)

- [Hop 跨 EVM 链 AMM 交易所将在 4 月推出，支持 xDai、Polygon 和 Arbitrum](https://medium.com/hop-protocol/hop-send-tokens-across-rollups-30f14c432f7c)

- Optimism [主网发布推迟到 7 月](https://optimismpbc.medium.com/optimistically-cautious-767a898f90c8)

- Zk rollup [Hermez 已上线](https://twitter.com/hermez_network/status/1374663423678689287) ，支持 ETH、HEZ、WBTC、USDT 和 DAI

- [Arbitrum 的进展](https://medium.com/offchainlabs/arbitrum-updates-buckle-up-80483d71718c)

- [Hop 跨 EVM 链 AMM 交易所将在 4 月推出，支持 xDai、Polygon 和 Arbitrum](https://medium.com/hop-protocol/hop-send-tokens-across-rollups-30f14c432f7c)

- Optimism [主网发布推迟到 7 月](https://optimismpbc.medium.com/optimistically-cautious-767a898f90c8)

- Zk rollup [Hermez 已上线](https://twitter.com/hermez_network/status/1374663423678689287) ，支持 ETH、HEZ、WBTC、USDT 和 DAI

- [理解 Arbitrum 的设计](https://developer.offchainlabs.com/docs/inside_arbitrum)

- zkSync [路线图更新](https://medium.com/matter-labs/zksync-2-0-roadmap-update-zkevm-testnet-in-may-mainnet-in-august-379c66995021)

- [给 5 岁小孩解释 Celer’s layer2.finance](https://blog.celer.network/2021/04/02/eli5-layer2-finance-the-modern-subway-of-the-defi-city/)

- GodsUnchained 推出了 [ImmutableX zkrollup 用于免费的 NFT 转移](https://www.immutable.com/blog/immutable-x-alpha-trading-launch)

- dydx 现在推出了其 [全仓永续保证金合约的 zkrollup](https://dydx.exchange/blog/public)（也是运行的 StarkEx）

- StarkWare 提出的 [Caspian l2 AMM](https://medium.com/starkware/caspian-an-l2-powered-amm-f20e93b5421)：按批撮合交易，净差额上 Layer-1 交易

- [L2beat](https://www.l2beat.com/)：跟踪 layer-2 的活动

- Deversifi 的 [layer2 生态系统路线图](https://twitter.com/deversifi/status/1387379996713439237/photo/1)

- Truebit 链下计算验证系统[宣布上线主网](https://truebit.substack.com/p/truebit-early-access)

- [OKEx 将支持存取款到 Arbitrum](https://www.okex.com/support/hc/en-us/articles/360060706511)

- StarkEx 在 dydx、Deversifi 和 Immutable 上交易量已[超过 10 亿美元](https://twitter.com/ukolodny/status/1390601561173381125)

- 两个标准化的工作：[标准化任意消息桥](https://ethereum-magicians.org/t/a-standard-interface-for-arbitrary-message-bridges-between-chains-layers/6163)以及 [token 转账标准接口](https://ethereum-magicians.org/t/outlining-a-standard-interface-for-cross-domain-erc20-transfers/6151)

- [Arbitrum 主网](https://medium.com/offchainlabs/wen-arbitrum-634969c14713)计划于 5 月 28 日向开发者推出。最新的测试网引入了一个排序者以实现即时交易

- [Loopring zkrollup 交易量突破 10 亿美元](https://twitter.com/loopringorg/status/1392676013751078913)

- [Ethhole.link](https://ethhole.link/)：从以太坊主网到 L2 和侧链的代币流动可视化

- Kris Kaczor 长推特比较 [Arbitrum 和 Optimism](https://twitter.com/krzKaczor/status/1395812308451004419)

- Etherscan [现已为 Optimism 提供区块浏览器](https://optimismpbc.medium.com/integrating-etherscan-24a3811a765c)

- [深入解读欺诈证明系统的验证者两难](https://medium.com/onther-tech/optimistic-rollup-is-not-secure-enough-than-you-think-cb23e6e6f11c)

- [Connext 的虚拟 AMMs 和流动性拍卖](https://medium.com/connext/solving-the-liquidity-problem-88bde201501)，用于跨链互换的负载均衡

- [optimistic rollups 和 zk rollups 差异概说](https://insights.deribit.com/market-research/making-sense-of-rollups-part-one-optimistic-vs-zero-knowledge/)

- Patrick McCorry：[桥决定了什么是 Layer 2](https://stonecoldpat.medium.com/a-note-on-bridges-layer-2-protocols-b01f8fc22324)

- [经其社区投票，Uniswap V3 登陆 Arbitrum](https://twitter.com/Uniswap/status/1400847744596598792)

- [通过 Alchemy 与 Arbitrum 主网交互和部署](https://blog.alchemy.com/blog/arbitrum-is-live)

- [深入理解 zkSync 2.0 的架构](https://twitter.com/zksync/status/1399469062539952128)：包括能支持大部分操作码的 zkEVM

- [大白话介绍 rollups](https://www.mechanism.capital/rollups-introduction/)

- [Raiden Bespin](https://medium.com/raiden-network/bespin-mainnet-release-announcement-87f5d5ede018)：稳定版放出，用户需要关闭和清算 Alderan 通道

- [ArbRinkeby](https://twitter.com/arbitrum/status/1403102638699433984)：稳定的 Arbitrum 测试网

- [10 分钟解释 Arbitrum](https://tracer.finance/radar/arbitrum-in-under-10/)

- [EY Nightfall 3](https://www.ey.com/en_gl/news/2021/07/ey-contributes-a-zero-knowledge-proof-layer-2-protocol-into-the-public-domain-to-help-address-increasing-transaction-costs-on-ethereum-blockchain)：带有零知识隐私转账的 optimistic rollup，每笔交易耗费约 8200 gas

- [Rollup diff compression](https://ethresear.ch/t/rollup-diff-compression-application-level-compression-strategies-to-reduce-the-l2-data-footprint-on-l1/9975)：减少 rollup 的主网 gas 用量

- [Synthetix 计划在 7 月 26 号在 Optimism 上开启交易所功能](https://blog.synthetix.io/optimism-launch-announcement/)

- Optimism [停止补贴 L2 gas 费](https://blog.synthetix.io/oks-gas-update-weth-distribution/)，也不再路由用户的取款交易，Synthetix 为现在的用户给出 WETH，足够发起多笔交易

- [Arbitrum 升级](https://offchain.medium.com/arbitrum-updates-standing-up-an-ecosystem-7666260a734b)：dApp 依赖的一些基础设施项目尚未就绪，单个的 ERC20 桥已在使用

- [用户可能不会再关心 rollup 的浪潮了](https://medium.com/dragonfly-research/im-worried--will-care-about-rollups-554bc743d4f1)

- [对比 Optimism 和 Arbitrum 的争议裁决方法](https://insights.deribit.com/market-research/making-sense-of-rollups-part-2-dispute-resolution-on-arbitrum-and-optimism/)

- [Optimism 携 Uniswap V3 上线](https://optimismpbc.medium.com/announcing-uniswap-v3-on-optimism-6fb033398a11)：alpha 版本，每天有 5 万笔交易的上限，使用拥堵定价算法，可能有 计划内/计划外 的停机，7 天的取款等待期，代币桥支持 DAI、WBTX、USDT、EURT、ETH 和 SNX

- [Uniswap 的 Optimism 文档](https://help.uniswap.org/en/collections/3033942-layer-2)

- [Rollup 中的单一许可型定序器](https://twitter.com/krzKaczor/status/1415326134552641536)

- [StarDrop](https://kobi.one/2021/07/14/stardrop.html)：以保护隐私的形式在 StarkNet 上分发奖励的实验性项目

- Synthetix 的 [synth 交易所已登陆 Optimism](https://blog.synthetix.io/synth-exchanges-are-live-on-l2/)，使用 Kwenta [alpha](https://blog.kwenta.io/everything-you-need-to-know-about-using-kwenta-on-l2/)

- [Tenderly 添加对 Optimism 的支持](https://blog.tenderly.co/optimistic-ethereum-integration-now-available-on-tenderly/)

- [Sorare](https://medium.com/sorare/were-live-on-our-scaling-solution-starkware-62438abee9a8) 迁移到 Starkware

- StarkEx [v3.0](https://medium.com/starkware/starkex-3-0-now-live-on-mainnet-57174a5f8beb)：放在 Layer-1 的金库用于 DeFi 的池子和去中心化的 AMM，为多个独立的 dApp 提供单一的 STARK 证明

- Arbitrum [将在本月内开放给用户](https://offchain.medium.com/a-is-for-arbitrum-a-is-for-august-71391582d95b)

- [NXTP](https://twitter.com/ConnextNetwork/status/1423677600350842890)，用于跨链的转账和合约调用，来自 Connext

- [Hermez零知识EVM](https://blog.hermez.io/introducing-hermez-zkevm/) 路线图

- 使用StarkEx担保的无信任[L2到侧链桥](https://medium.com/starkware/a-trustless-sidechain-to-starkex-bridge-secured-by-ethereum-61e00f19f7e0)的建议。

- [Arbitrrum门户网站](https://portal.arbitrum.one/)的dapps、钱包和工具将于8月推出

- [Optimism 开发者入门测试](https://community.optimism.io/docs/developers/l2/deploy.html)，测试网创世可能在 10 月，ETH 可能不再兼容 ERC20，使用 Solidity 编译器，使用 EOA 而非合约钱包，减低 gas 用量

- [1inch Network](https://blog.1inch.io/the-1inch-network-expands-to-optimistic-ethereum-2beb89fa63bf) 已上线 Optimism

- [Teleportr](https://twitter.com/0x_clem/status/1428606240293212167)：低成本的主网到 Optimism 的 ETH 桥，限额 0.02 ETH

- [Warp](https://medium.com/nethermind-eth/warp-your-way-to-starknet-ddd6856875e0)：EVM 到 Cairo（StarkNet 的 智能合约编程语言）转译器

- StarkWare 的[共享证明器](https://twitter.com/ukolodny/status/1428556705525374978)降低小 app 的 layer 2 负担

- [Optimism](https://optimismpbc.medium.com/arbitrary-token-bridging-d552f6bef694)增加了自定义ERC20代币的存款和提款功能

- [Nova](https://twitter.com/transmissions11/status/1431044602287464450)：在L2和L1之间进行合约调用的无信任中继，部署在Optimism和主网，目前仅限于批准的项目。

- [Hop](https://twitter.com/HopProtocol/status/1430630767042904074) 实现了USDC和USDT从Optimism到主网的快速退出，避免了7天 optimistic rollup 提款时间。

- [Loopring zkRollup NFTs](https://medium.com/loopring-protocol/loopring-now-supports-nfts-on-l2-29174a343d0d)：在L2上铸造、交易和转让，存款到L2，提款到L1， 支持ERC721和ERC1155。

- L2Beat增加了[风险视图](https://twitter.com/l2beatcom/status/1431195213113012227)：安全、数据可用性、可以更改的内容以及在审查或系统下线时该怎么做。

- Arbitrum

   

  beta 版，初始上限为每秒 80k arbgas（等价于主网处理量），代币桥仅限于已得到许可的 token，7 天的退出时间

  - Ether 及代币桥 [教程](https://arbitrum.io/bridge-tutorial/)
  - [Arbiscan](https://arbiscan.io/) 区块浏览器
  - Arbitrum 团队的 Reddit [AMA](https://www.reddit.com/r/ethereum/comments/pgl5a4/were_offchain_labs_the_team_behind_arbitrum_the/)
  - [精选的 app 已经上线](https://portal.arbitrum.one/)：Uniswap、 Celer 的 cBridge、Balancer、Sushiswap、Dodo、MCDEX、Swapr。更多 app 即将到来

- [Celer 的 cBridge 已经为 Optimism 添加支持](https://twitter.com/CelerNetwork/status/1432751288681316352)。3 分钟即可退出。可以在主网、Optimism、Arbitrum 和一些侧链间转账

- StarkNet [Alpha 2](https://medium.com/starkware/starknet-alpha-2-4aa116f0ecfc)：可组合性、本地测试、迁移到 Goerli 测试网；OpenZeppelin 正在开发合约库

- [Immutable X](https://twitter.com/Immutable/status/1433703349061242882) 开放免 gas 费的 NFT 铸造和交易

- David Mihal 的 [L2 Fees](https://l2fees.info/)：比较在不同方案上转移 ETH/token 和币币互换的风险

- [zk Rollups 和数据分片可以实现全球级的扩展](https://polynya.medium.com/why-rollups-data-shards-are-the-only-sustainable-solution-for-high-scalability-c9aabd6fbb48)，同时保持技术和经济上的可持续

- [为什么 Arbitrum 要使用交互式欺诈证明](https://medium.com/offchainlabs/interactive-fraud-proofs-arbitrums-secret-sauce-debc3b019418)（[中文译本](https://ethfans.org/posts/interactive-fraud-proofs-arbitrums-secret-sauce)）

- [Arbitrum 的免许可代币桥](https://offchain.medium.com/continued-path-to-decentralization-bridging-tokens-into-arbitrum-42a94b054560)将在 10 月 22 日开放，除非项目自定义，否则 token 会当成一个基本 ERC20 代币来传输

- 呼吁去中心化交易所[支持 rollup](https://twitter.com/evan_van_ness/status/1435277869115191296)

- Arbitrum 上运行的项目的[完整列表](https://twitter.com/playtern/status/1439224019295752194)

- Aztec 将其 zkrollup 上的隐私交易额度提高到 [30 ETH、100k DAI 和 2 renBTC](https://medium.com/aztec-protocol/aztec-2-0-updates-100-000-transaction-limits-more-e55c17c0ecf8)

- Bitfinex 直接支持到 DeversiFi（基于 StarkWare 的 Layer-2）的[桥](https://www.bitfinex.com/posts/710)，从 USDT 开始

- [从 Solidity 到 Cairo 得转译器](https://medium.com/nethermind-eth/solidity-on-starknet-terminal-velocity-e8df5f63e010)切换模式，从转译 EVM 操作码换成使用 Solidity 和 Yul 语言的 AST

- [Rollup 的抗审查性](https://twitter.com/bkiepuszewski/status/1440553708295577607)：Arbitrum 和 Optimism 用户可以强制在 layer-1 上交易，而 StarkWare layer2 使用 app 定制机制

- Optimism [OVM v2.0](https://twitter.com/optimismpbc/status/1442975200752979976) 将在 10 月 14 日登陆 Kovan 测试网，10 月 28 日登录主网，升级时会有 4~6 小时的停机

- [通用 zkEVM 的设计挑战](https://hackmd.io/@yezhang/S1_KMMbGt)，解决方案是多项式承诺、查找表、更灵活的递归证明以及硬件加速

- ·[Cairo 程序执行](https://arxiv.org/abs/2109.14534)的正确性证明

- [StarkNet alpha](https://medium.com/starkware/starknet-alpha-is-coming-to-mainnet-b825829eaf32) 计划在 11 月在主网推出：准入型部署，在 alpha 和 beta 版本间没有后向兼容保证

- [Etherscan 推出的 Arbitrum 测试网浏览器](https://testnet.arbiscan.io/)

- [有效性证明的成本摊销](https://polynya.medium.com/the-dynamics-around-validity-proof-amortization-519e9ae291c1)：zkRollup 交易越多，单交易成本越低

- [无需公开交易历史数据的 zkRollup](https://ethresear.ch/t/a-zkrollup-with-no-transaction-history-data-to-enable-private-smart-contract-execution-with-calldata-efficiency/10961)，可支持隐私合约执行并精简 calldata

- [Uniswap v2 fork](https://medium.com/matter-labs/unisync-a-port-of-uniswap-v2-on-the-zkevm-b12954748504)（Solidity 合约和 dapp）在 zkEVM 测试网上运行的 demo

- zkSync Reddit [AMA](https://www.reddit.com/r/ethereum/comments/q8q822/ama_were_matter_labs_the_team_behind_zksync_the/)

- [Arbitrum Nitro](https://medium.com/offchainlabs/arbitrum-nitro-sneak-preview-44550d9054f5) 升级预览：运行在 WASM 上，使用 Geth 替换了定制化的 EVM 模拟器，预计有 20 到 50 倍的执行速度提升

- zkevm-circuits [v0.0.1](https://github.com/appliedzkp/zkevm-circuits/releases/tag/v0.0.1)：首次发布，实现了 PUSHX、POP、ADD、SUB、LT、GT 操作码

- [Optimism 的 EVM 等价升级推迟到 11 月 11 日](https://twitter.com/optimismpbc/status/1451339513964359682)

- [Springrollup](https://ethresear.ch/t/springrollup-a-zk-rollup-that-allows-a-sender-to-batch-an-unlimited-number-of-transfers-with-only-6-bytes-of-calldata-per-batch/11033) 提议 zkrollup：发送者可以把无限量的转账组合交易包，每个包在链上只需发布 6 字节的 calldata

- [使用 zk rollup 实现具备弱隐私性的AMM](https://ethresear.ch/t/why-you-can-build-a-private-uniswap-with-weak-secrecy-in-zkrollup/11031)

- [对比 Arbitrum 和 Optimism 的错误性证明](https://medium.com/@cpbuckland88/fraud-proofs-and-virtual-machines-2826a3412099)

- 论文：[带验证功能的桥接系统](https://stonecoldpat.github.io/images/validatingbridges.pdf)

- [Phonon](https://blog.gridplus.io/worlds-first-phonon-transfer-601818203a0c)：第一种硬件实现的隐私链下 memecoin 交易

- [主网上的 StarkNet 证明测试交易](https://twitter.com/CairoLang/status/1453287662777929743)

- [Arbitrum ERC20 免许可桥接已开放](https://twitter.com/arbitrum/status/1451661392281579530)

- Optimism [EVM 等价虚拟机](https://medium.com/ethereum-optimism/introducing-evm-equivalence-5c2021deb306)

- [zk rollups 的未来](https://hackmd.io/@canti/rkUT0BD8K)：侧链应采取务实的方法成为 zk-rollup

- [Connext 加入主网支持](https://twitter.com/connextnetwork/status/1456381261036113923)：可在主网、Arbitrum 和一些侧链之间转移稳定币

- [Optimism](https://optimismpbc.medium.com/all-gas-no-brakes-8b0f32afd466) 现在不允许部署；未来的更新会继续维护，交易历史和事件数据

- [Immutable X](https://support.immutable.com/hc/en-us/articles/4411697220879/) 通过与 MoonPay 整合开通法币通道

- [Polygon Nightfall](https://blog.polygon.technology/zk-proofs-protocol-polygon-nightfall-launches-on-testnet-to-provide-low-cost-private-ethereum-transaction/) zk-optimistic rollup 测试网

- Arbitrum [对比 ZK 和 Optimistic Rollups](https://medium.com/offchainlabs/optimistic-rollups-the-present-and-future-of-ethereum-scaling-60fb9067ae87)

- Optimistic Rollups 7天欺诈证明窗口[解释](https://twitter.com/bkiepuszewski/status/1471116288261038088)

- Celer 呼吁建立一个[开放的规范化 token 桥接标准](https://blog.celer.network/2021/12/13/say-no-to-vendor-lock-in-calling-for-an-open-canonical-token-bridge-standard/)

- [Transak](https://twitter.com/transak_finance/status/1473367208474681351) 法币通道支持 Arbitrum 和 Optimism （目前在美国还不支持）

- [Banxa](https://twitter.com/BanxaOfficial/status/1473887349251604483) 法币通道支持 Arbitrum

- [DeversiFi](https://deversifi.com/blog/announcing-our-brand-new-integration-with-moonpay/) 通过与 MoonPay 整合开通法币通道

- [Loopring](https://medium.loopring.io/introducing-l2-counterfactual-wallet-and-fiat-on-ramps-a4b60edf15d6) Layer2 启动 iOS 钱包，支付后部署到Layer1 以主网撤回和更多功能，通过 Ramp 法币通道

- [跨链协议](https://medium.com/across-protocol/the-fastest-cheapest-and-most-secure-bridge-now-supports-l1-l2-transfers-92d7b2bd6666)：现在桥是双向的，增加主网到 Arbitrum，Optimism 和 Boba

- [StarkWare Layer3扩展](https://medium.com/starkware/fractal-scaling-from-l2-to-l3-7fe238ecfb4f)：应用特定层使用递归证明，StarkEx Layer2 的可以迁移到Layer3

- [Huobi Global](https://www.huobi.ma/support/en-us/detail/84894749060248) 支持在Arbitrum 上进行 ETH 存取款

- [Argent zksync wallet](https://www.argent.xyz/blog/trading-is-now-live-on-l2/) 集成 ZigZag 交易所，统一收取 $1 的交易和网络费

- [#L222](https://www.argent.xyz/blog/trading-is-now-live-on-l2/) 是2022年 Layer 2 采用的“官方的”标签🦆

- 来自 ConsenSys Applied R&D 的高效的[zk-EVM](https://ethresear.ch/t/a-zk-evm-specification/11549)算法建议

- 12月，在主网上 Layer2 交易[消耗掉31.7亿gas](https://twitter.com/PaoloRebuffo/status/1476935896553377792)

- Fuel v2 [Sway](https://twitter.com/fuellabs_/status/1478049212357087232) 语言，受Rust语言影响，代码库已经公开，可为本地开发使用

- StarkNet Alpha [交易费方案](https://community.starknet.io/t/fees-in-starknet-alpha/286)

- [Polygon Miden VM](https://twitter.com/0xpolygonmiden/status/1478783518800953347) 将支持无符号32位整型

- 用于StarkNet Warp转译器的[IR语义的形式化规范](https://medium.com/nethermind-eth/securing-warp-a-formal-specification-of-the-yul-ir-85bb3bf51c62)

- [Phonon alpha](https://blog.phonon.network/phonon-alpha-getting-started-f85e107de390)（点对点隐私传输使用硬件保障安全）：在智能卡上提供并执行一个小程序的工具

- Optimism[交易费降低](https://twitter.com/optimismpbc/status/1479840872954875905)，平均每笔交易便宜约 30%

- Arbitrum 因 [Sequencer 硬件故障停机](https://offchain.medium.com/todays-arbitrum-sequencer-downtime-what-happened-6382a3066fbc)，而备份恰好在进行软件升级，Arbitrum 仍处于测试阶段，计划在 Sequencer 更加去中心化。

- [Binance](https://twitter.com/dmihal/status/1481709263932342275) 支持 Arbitrum 网络提款。

- Polygon Zero（以前称为 Mir）[Plonky2](https://blog.polygon.technology/introducing-plonky2/)：基于 PLONK 和 FRI 的递归 SNARK， Macbook Pro 上大约 170 毫秒内可生成递归证明。

- Fuel 提出 Layer 2 代币模型：为Layer 2 区块生产者 [将收取费用的权利代币化](https://fuel-labs.ghost.io/token-model-layer-2-block-production/)；建议避免使用PoS、费用支付和治理模型。

- Celer[跨链消息框架](https://blog.celer.network/2022/01/12/celer-inter-chain-message-framework-the-paradigm-shift-for-building-and-using-multi-blockchain-dapps/)上线测试网，单击 UX 发送任意消息并执行指令。

- 采用 Optimism 和 Arbitrum 与侧链的[对比曲线图](https://twitter.com/ethdreamer/status/1480676820026597377)（线性和对数刻度）

- Connext [Vector v0.1.0](https://medium.com/connext/vector-0-1-0-mainnet-release-9496ae52c422) live, 用状态通道在 layer2 之间转移价值

- [Warp](https://medium.com/nethermind-eth/helm-warp-one-engage-8526233780c4) (Solidity - Cairo 转换器): 第一个主线版本，用 Solidity 编写测试。

- StarkNet Prover 代码 [许可讨论](https://community.starknet.io/t/starknet-prover-code-license/371)

- [Arbitrum 升级](https://twitter.com/bfreshhb/status/1486057303224856578) ：更低的交易费

- [Bybit](https://twitter.com/Bybit_Official/status/1486640127627743235) and [MEXC Global](https://twitter.com/MEXC_Global/status/1485780322176225282) 增加对 Arbitrum 存取的支持

- Arbitrum 抗审查[解释器](https://twitter.com/bkiepuszewski/status/1486416315799769097), 在6545个区块和24小时超时后交易可以强制进入主网

- [MoonPay](https://twitter.com/immutable/status/1486869212295360512) 增加支持用信用卡购买Immutable X ETH

- Polygon Hermez 文档 ：[zkEVM 基于操作码的方法](https://blog.hermez.io/zkevm-documentation/)

- [L2Savings](https://www.l2savings.org/): 显示 Layer 2 交易记录，并且对比在主网上交易节省 gas 的情况

- 挑战过程的[滑动窗口](https://arxiv.org/abs/2201.09009), 允许显著减少挑战响应时限，并可以在拥塞时延长响应时限

- [Secure Asymmetric Frugal Exchange](https://ethresear.ch/t/optimizing-cross-chain-swaps/11924) 原型，用于优化批量跨链交换，从 Layer2 到主网高效转移

- [L2 费](https://l2fees.info/l1-fees): Layer2 为以太坊的安全向 Layer1 支付费用

- [FTX](https://twitter.com/FTX_Official/status/1490903050269396992) 支持 Arbitrum

- [Argent zkSync 钱包](https://www.argent.xyz/blog/how-to-fund-your-argent-l2-account-from-an-exchange/) 通过 LayerSwap 从交易所存款

- [Proof of Efficiency](https://ethresear.ch/t/proof-of-efficiency-a-new-consensus-mechanism-for-zk-rollups/11988): 针对 zk-rollups 的 Polygon Hermez 共识提案

- [OKX](https://www.okx.com/support/hc/en-us/articles/4426572059661-OKx-supports-Arbitrum-s-network) 和 [Bitget](https://bitget.zendesk.com/hc/en-us/articles/4419534112665) 增加 Arbitrum 支持

- [Urbit](https://urbit.org/blog/layer-2-guides) naive rollup 已上线

- [Optimism 费用解释器](https://optimismpbc.medium.com/fancy-numbers-how-we-lowered-fees-for-optimism-users-a3bb80cbc65f), 如何将费用降低30%, 按照1.24的费用标量，每笔交易固定费用2100gas

- [zkSync v2.0](https://matterlabs.medium.com/zksync-2-0-public-testnet-is-live-de870ba9632a) EVM兼容zk-rollup公共测试网

- [Optimism](https://twitter.com/optimismPBC/status/1496241118077505536)减少时间戳，更新到15秒

- Barnabé: [rollup 经济](https://barnabe.substack.com/p/understanding-rollup-economics-from), 关于费用和经济设计的讨论

- Norswap的[关于optimistic rollups 和 rollups 的看法](https://twitter.com/norswap/status/1494763568843132931)

- Polynya的[《区块链的状态》](https://polynya.medium.com/optimistic-rollups-are-brilliant-and-the-state-of-blockchains-a57bc4799dca): optimistic rollups将发展的更快，但是zk-rollups会赶上它，任何没有效性证明、欺诈证明和数据证明的链都会死。

- Fran (OpenZeppelin拥护者): 关于[ERC20 token 在桥接器上可铸造](https://ethereum-magicians.org/t/idea-erc20-bridge-mintable-tokens/8422)的想法

- 通过批量压缩calldata，[Optimism 将费用降低 30-40%](https://medium.com/ethereum-optimism/the-road-to-sub-dollar-transactions-part-2-compression-edition-6bb2890e3e92) ，长期计划与字典一起使用zstd实现尽可能高的压缩比并尽可能低的降低费用

- [Pathfinder](https://github.com/eqlabs/pathfinder#readme) v0.1.0: Rust 中的 StarkNet 全节点, alpha

- [Arbitrum AnyTrust](https://medium.com/offchainlabs/introducing-anytrust-chains-cheaper-faster-l2-chains-with-minimal-trust-assumptions-31def59eb8d7)链宣称：超低成本交易，数据哈希上传到主网，委员会运营，假设最少成员诚实，可以回退到标准rollup协议。

- Polynya: [历史存储](https://polynya.medium.com/the-endgame-bottleneck-historical-storage-e83c101d2a7c)——最后的瓶颈，状态增长将通过无状态、有效性证明、状态到期以及PBS来解决

- Arbitrum [gas 计价变化](https://research.arbitrum.io/t/l2-compute-gas-pricing/22)，在定价算法中添加一个动量项，gas池充满标准修改为80%，系统反应会更快但更温和。

- [DefersiFi](https://twitter.com/deversifi/status/1500853979479261185)添加到 Arbitrum 的桥

- Argent [zkSync 移动钱包](https://twitter.com/argentHQ/status/1500879350677262336) 测试版结束

- Argent X (StarkNet) [v3](https://github.com/argentlabs/argent-x/releases/tag/v3.0.0): 从 v2迁移资产，如[briqs](https://twitter.com/briqNFT/status/1502329495528873994)

- StarkNet [账户抽象设计的提案](https://community.starknet.io/t/starknet-account-abstraction-model-part-1/781)

- Optimism 提出[ Cannon 缺陷证明](https://medium.com/ethereum-optimism/cannon-cannon-cannon-introducing-cannon-4ce0d9245a03); Norswap 关于 Cannon 的[说明](https://twitter.com/norswap/status/1502085000967061504)

- Optimism [多客户端计划](https://medium.com/ethereum-optimism/our-pragmatic-path-to-decentralization-cb5805ca43c1) 去中心化并去掉升级密钥

- Vitalik 关于 EIP 4488 （降低了calldata的gas开销）和 EIP4844(带有一个新的数据字段的交易类型) 在显著降低rollup交易成本方面的[说明](https://www.reddit.com/r/ethereum/comments/t7adbt/hows_eip4844_differ_from_eip4488_where_both/hzi63ig)

- StarkNet Alpha [v0.8.0](https://medium.com/starkware/starknet-alpha-0-8-0-16e046e0f94b) 测试网, 增加费用, 可选到 v.0.9.0

- [部分匿名 rollup 设计](https://ethresear.ch/t/partially-anonymous-rollups-a-new-rollup-design/12206), 运营商可以拥有完整的数据可用性，账户活动信息会在更新账户状态哈希是泄露，但交易细节对无关方不透明

- optimistic rollups 的[欺诈证明攻击向量](https://medium.com/infinitism/optimistic-time-travel-6680567f1864): time-travel 攻击和 reality-distortion 攻击; Optimism 的 Cannon 和 Arbitrum 的 Nitro 有望减少攻击面

- [Optimism 压缩调用数据](https://twitter.com/optimismPBC/status/1507077884594241547), 下周有望降低 40% 费用

- [Huobi](https://twitter.com/huobiglobal/status/1506868755409760259)增加在 Optimism 存取以太币

- [Ramp](https://twitter.com/RampNetwork/status/1506211050907131908) 增加在 Arbitrum 购买以太币

- Raiden 已经实现了 Raiden协议，[提供可扩展的支付解决方案](https://medium.com/raiden-network/the-state-of-the-raiden-network-9aa4b9a9e2fc)

- [Aztec (privacy zk rollup)](https://medium.com/aztec-protocol/privacy-for-pennies-scaling-aztecs-zkrollup-9f2b36615cc6) ：降低交易费，证明成本降低了30%(计划降低到 180K gas)， 每笔主网交易汇总 896 Aztec 交易，吞吐量提高了8倍。

- Optimism [调用数据压缩](https://twitter.com/bkiepuszewski/status/1508740414492323840): 如何解压缩 Layer 2 交易

- [以 Rollup 为中心的未来](https://twitter.com/pseudotheos/status/1509530981581000705) 而不是跨链，因为桥是单点漏洞并会让目标链陷入传染风险

- Arbitrum [Nitro 开发网](https://medium.com/offchainlabs/its-nitro-time-86944693bf29): 使用核心 Geth 对 WASM 进行欺诈证明，调用数据压缩，代码使用商业源许可证；Twitter 水龙头

- Polynya: [rollup 类型](https://twitter.com/poiynya/status/1511623759786307586), 常规的（regular）,不可变的（immutable）, 铭记的（enshrined）和自主的（sovereign）

- [KuCoin](https://www.kucoin.com/news/en-arb-withdrawal-service-for-eth-usdt-usdc-is-now-supported-on-kucoin-20220412) 支持 Arbitrum 提款

- [Worldcoin 开源协议](https://worldcoin.org/open-sourcing-worldcoin) ，包括基于 Hubble Project 和 Semaphore 的optimistic rollup

- [KuCoin](https://www.kucoin.com/news/en-optimism-is-now-supported-on-kucoin-20220416)为Optimism增加存款

- [EIP4844 降低rollup数据可用性成本](https://medium.com/@cpbuckland88/4844-it-aint-all-gravy-fbae93bacf2)，但增加了一个新的诚实方假设

- 基于EIP-4844的去中心化[zk-rollup设计](https://ethresear.ch/t/a-design-of-decentralized-zk-rollups-based-on-eip-4844/12434)

- [Optimism Collective](https://optimism.mirror.xyz/gQWKlrDqHzdKPsB1iUnI-cVN3v0NvsWnazK7ajlt1fI)：第二季度OP Airdrop #1。网络和公共产品资金采用两院制治理

- [Celer 跨链消息传递框架](https://blog.celer.network/2022/04/25/celer-im-launches-on-mainnet-a-new-era-for-inter-chain-dapps-begins/)在主网上线

- [Taiko](https://taikochain.github.io/l2design/)：去中心化 zk-rollup 设计的初稿

- [KZG 承诺解释器](https://twitter.com/bkiepuszewski/status/1518163771788824576)（danksharding 承诺方案），比 Merkle 树更高效，但需要可信设置

- [Raiden](https://medium.com/raiden-network/the-raiden-network-now-live-on-arbitrum-3d57fedf7961) 已上线 Arbitrum

- 由数据压缩导致的 Optimistic rollup [价格差异](https://twitter.com/bkiepuszewski/status/1522152595086839808)

- Kelvin: [hybrid ZK/Optimistic rollup](https://kelvinfichter.com/pages/thoughts/hybrid-rollups/) 的未来

- Polynya: [分离的区块链层](https://mirror.xyz/polynya.eth/C7pabfX3j8r65w8SWlpT_dem1_JXvQxsQao4V0xjsNY)

- [KuCoin](https://www.kucoin.com/news/en-opt-deposit-service-for-usdt-usdc-is-now-supported-on-kucoin-20220511) 支持对 USDC 和 USDT 的Optimism存储服务

- 抗审查[桥](https://twitter.com/bkiepuszewski/status/1524666863858425856)

- Polygon Hermez v2 [deep dive](https://blog.polygon.technology/zkverse-deep-dive-into-polygon-hermez-2-0/)

- Optimism [Bedrock](https://dev.optimism.io/introducing-optimism-bedrock/) 源码可用, MIT 许可

- Binance [支持 Optimism](https://www.binance.com/en/support/announcement/3ee5a24e46754c278eaf769fcd4b7edf?ref=AZTKZ9XS)

- [zk-rollup 提议使用实用的可验证延迟加密](https://ethresear.ch/t/mev-resistant-zk-rollups-with-practical-vde-pvde/12677)来最小化 MEV

- Optimism

  - Hiccups: [JSON-RPC 链读取崩溃](https://twitter.com/optimismPBC/status/1531798286037680129) 但序列完好
  - 只需要简单的 Docker 设置就可以[运行你自己的 Optimism 节点](https://github.com/smartcontracts/simple-optimism-node)

- 为持续发展，ImmutableX 将收取[ 2% 的费用](https://immutablex.medium.com/fees-on-immutable-x-79d3e7207b12)

- [Aztec](https://twitter.com/aztecnetwork/status/1535024615969263616)由于涉及加密交易，Aztec证明超过了300Kb，超过了Geth 128Kb的交易大小限制。Geth是最大的以太坊客户端，这使得发送大型交易变得棘手。

- 关于 L2 交易终结的[解释](https://twitter.com/bkiepuszewski/status/1533347284817174528)

- Arbitrum [的二维 gas 费](https://medium.com/offchainlabs/understanding-arbitrum-2-dimensional-fees-fd1d582596c9): L1 数据调用费 + L2 计算用量

- [Optimism 测试网](https://dev.optimism.io/kovan-to-goerli/)从 Kovan 迁移到 Goerli

- [Aztec Connect](https://medium.com/aztec-protocol/aztec-network-launches-first-ever-private-defi-solution-for-ethereum-e5ec7624d430) (privacy zk rollup) 上线主网；zk.money DeFi 聚合器支持 Element 和 Lido

- [Layer2 桥接风险框架](https://gov.l2beat.com/t/l2bridge-risk-framework/31)的提案

- 在 Arbitrum Odyssey 桥接期间 Hop [转账延迟](https://authereum.notion.site/Arbitrum-Bridge-Week-Delays-Post-Mortem-ac1f6681f8c241a8ad524ba2047d083e) 的事后分析

- Arbitrum 的 [Nova](https://offchain.medium.com/introducing-nova-arbitrum-anytrust-mainnet-is-open-for-developers-9a54692f345e) (用 AnyTrust) 对开发者开放；数据可用性委员会（增加了-假设两个成员是诚实的）

- StarkNet [代币](https://medium.com/starkware/part-1-starknet-sovereignty-a-decentralization-proposal-bca3e98a01ef) 宣布用于治理, 费用和质押; 初始 49.9% 分配给投资者和核心贡献者

- [L2Beat](https://twitter.com/l2beat/status/1547498863921188864)添加关于授权地址的信息，帮助用户了解每个汇总所做的临时中心化权衡

- Polygon zkEVM [开源实现](https://blog.polygon.technology/the-future-is-now-for-ethereum-scaling-introducing-polygon-zkevm)

- Scroll [pre-alpha 测试网](https://mirror.xyz/scroll.eth/XQyXDgyxoefag6hcBgGJFz8qrb10rmSU-zUBvY3Q9_A)

- zkSync v2.0 [路线图](https://matterlabs.medium.com/100-days-to-mainnet-6f230893bd73), 100天内上主网

- [TxStreet 交易可视化工具](https://twitter.com/txstreetCom/status/1549016696618377216) 添加 Arbitrum, 测试版

- EIP4844 (proto-danksharding) [视频会议](https://www.youtube.com/watch?v=t8B7NBRPBfg) 和 [记录](https://docs.google.com/document/d/1KgKZnb5P07rdLBb_nRCaXhzG_4PBoZXtFQNzKO2mrvc/edit#)

- [BLS 交易类型](https://research.arbitrum.io/t/draft-for-bls-transaction-type-erc/117)草案

- Layer 2 修复， 可变的[实际 gas 消耗](https://twitter.com/sanjaypshah/status/1552679502018478084)

- [Arbitrum Rinkeby 测试网](https://twitter.com/arbitrum/status/1552759659139944453) 升级到 Nitro

- [Rainbow 手机钱包](https://twitter.com/rainbowdotme/status/1551978336359890944) 在 Arbitrum 和 Optimism 上添加了 NFT

- 给初学者: [Optimism 初学者指南](https://app.optimism.io/get-started)



- Arbitrum One 8月31日[升级到 Nitro](https://medium.com/offchainlabs/prepare-your-engines-nitro-is-imminent-a46af99b9e60)
- Delphi Digital: [rollup 指南](https://members.delphidigital.io/reports/the-complete-guide-to-rollups)
- Vitalik: [zk-EVM 不同类型和其优缺点对比](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)
- BLS 钱包[概述与演示](https://medium.com/privacy-scaling-explorations/bls-wallet-bundling-up-data-fb5424d3bdd3),绑定带有BLS签名的操作，减少数据的链上存储和交易成本
- Arbitrum [Nova](https://medium.com/offchainlabs/its-time-for-a-new-dawn-nova-is-open-to-the-public-a081df1e4ad2) (用 AnyTrust)对公众开放; 初始数据提供委员会包括 Consensys, FTX, Google Cloud, Offchain Labs, P2P, Quicknode 和 Reddit
- Arbitrum [Nitro 白皮书](https://github.com/OffchainLabs/nitro/blob/master/docs/Nitro-whitepaper.pdf)
- [创建缓存合约](https://ethereum.org/en/developers/tutorials/all-you-can-cache/)来降低 Layer 2 成本
- Optimism [Bedrock alpha 测试网](https://oplabs.notion.site/External-Optimism-Bedrock-Alpha-Testnet-454a37e469af4658b89a9d766334e331)
- Polynya 认为[EIP4488（降低 calldata gas 成本）](https://polynya.mirror.xyz/eq4PNsg-ld4V2LDB7HlHuNm-ULFAtvmVH--utaxAAEs)应该是扩展的重点
- Dankrad：[在证明中用 KZG 承诺](https://notes.ethereum.org/@dankrad/kzg_commitments_in_proofs)
- Justin Thaler：[SNARK 80 位的安全性太低](https://a16zcrypto.com/snark-security-and-performance/)，应该至少 100 位
- [Optimism Quests](https://dev.optimism.io/quests/)：鼓励尝试使用 NFT 的 dapp
- Vitalik：[Layer3是有意义的](https://vitalik.ca/general/2022/09/17/layer_3.html)
- Norswap 对比了 [Optimism Bedrock 和 Arbitrum Nitro](https://norswap.com/bedrock-vs-nitro/)
- Polynya: [rollup 比 Layer 1 提供更高的吞吐量](https://polynya.mirror.xyz/zYV0g6II0iREbcaph8362sRBC2_a2hm6NlKj1-yuh6Q)，因为 rollup 需要的节点更少
- Polygon zkEVM [公共测试网](https://blog.polygon.technology/polygon-zkevm-public-testnet-the-next-chapter-for-ethereum/), 开源 zk 验证系统
- Polynya: [最小可行的 rollup 去中心化](https://polynya.mirror.xyz/0hmnwXCiainU7HMYes73uH_kjQABRwk6b1WrqgQPAMg) ([Rollup 里程碑](https://ethereum-magicians.org/t/proposed-milestones-for-rollups-taking-off-training-wheels/11571)的第一阶段: 有限依赖运营商节点 - 辅助轮)
- [Arbitrum One](https://offchain.medium.com/arbitrum-decentralization-update-39f093768c42)现在有 9 个验证者
- [类型理论争议协议](https://ethresear.ch/t/type-theoretic-dispute-protocols/14213)提案
- Christine Kim：[zkEVM 概述](https://www.galaxy.com/research/whitepapers/zkevms-the-future-of-ethereum-scalability/)
- zkSync v2.0 [L1 到 L2](https://twitter.com/bkiepuszewski/status/1600792830129016832)和[L2 到 L1](https://twitter.com/bkiepuszewski/status/1601138527508393990)消息传递
- Tincho：[Arbitrum 桥恶意攻击](https://www.notonlyowner.com/research/message-traps-in-the-arbitrum-bridge)，中继可以无限消耗 gas
  - 由于没有第三方中继的计划，Arbitrum 桥的[攻击被认为是不现实的](https://twitter.com/hkalodner/status/1601788042431397889)
  - Scroll：[证明生成](https://scroll.io/blog/proofGeneration)的计算瓶颈，以及主要的加速方法。
  - Polygon [zkEVM](https://twitter.com/jbaylina/status/1603144831978831872)在每小时7美元的 AWS 实例上证明时间为2.5分钟
  - Consensys [Vortex zk 证明](https://ethresear.ch/t/vortex-building-a-prover-for-the-zk-evm/14427)：在 AWS hpc6a.48xlarge 上 5 分钟内产生 3000 万 gas 的块
  - zkSync [v2 alpha 延迟](https://twitter.com/zksync/status/1602687972108664832)到 2023 年第二季度
- Polygon zkEVM [测试网升级](https://polygon.technology/blog/final-approach-last-testnet-for-an-upgraded-polygon-zkevm) ：使用递归，更快的证明时间和打包聚合
- Intmax (zk-rollup) [testnet](https://medium.com/@intmax/the-missing-piece-for-the-ethereum-endgame-intmax-testnet-v1-launch-a2f40d406b42), 仅有命令行
- Justin Drake: [SGX作为务实的对冲措施来应对ZK-rollup SNARK漏洞](https://ethresear.ch/t/2fa-zk-rollups-using-sgx/14462)
- Arbitrum:
  - [rollups上应对延迟攻击的方法](https://offchain.medium.com/solutions-to-delay-attacks-on-rollups-434f9d05a07a)
  - [在主网上的批量发布策略](https://arxiv.org/abs/2212.10337)
- Taiko: [rollup 去中心化](https://mirror.xyz/labs.taiko.eth/sxR3iKyD-GvTuyI9moCg4_ggDI4E4CqnvhdwRq5yL0A), 定义 & 整体思路
- Taiko zkEVM [alpha-1 测试网](https://mirror.xyz/labs.taiko.eth/-lahy4KbGkeAcqhs0ETG3Up3oTVzZ0wLoE1eK_ao5h4)
- [Rock 5B ARM64 板上的 Optimism 节点](https://twitter.com/EthereumOnARM/status/1607373283082792961)，概念证明
- Layer 2（验证交易 + 桥）[在主网上的 gas 消耗](https://twitter.com/PaoloRebuffo/status/1608793955671605249)创纪录的月份
- [zkEVM 的 gas 定价](https://twitter.com/_bfarmer/status/1611223861604892673)选项
- L2beat：[LayerZero 网桥安全](https://medium.com/l2beat/circumventing-layer-zero-5e9f652a5d3e)从根本上说是一种可信模型
- Optimism 的[ Bedrock 升级解释](https://community.optimism.io/docs/developers/bedrock/explainer/)
- Kelvin Fichter：为什么在 Optimistic Rollups 中[以 7 天为挑战期](https://kelvinfichter.com/pages/thoughts/challenge-periods/)
- [Ticking-Optimism](https://therealbytes.substack.com/p/presenting-ticking-optimism)：基于 Optimism Bedrock 的 ticking 链，概念证明
- [Optimism 空投2](https://optimism.mirror.xyz/lPZEkFF7LU2ZlrO-dsV3p_LtWQUaknFGfxFMgSz3vGA)：OP 代币直接发送给委托人和高级用户
- Arbitrum [Stylus](https://offchain.medium.com/hello-stylus-6b18fecc3a22)公告、Arbitrum One/Nova 的编程环境和 WASM VM，预计 2023 年将允许非 EVM 语言（如 Rust、C、C++）的程序

- zkEVMs 争先恐后地进入主网：
  - [zkSync Era](https://blog.matter-labs.io/all-aboard-zksync-era-mainnet-8b8964ba7c59) (zkSync v2) 在 alpha 发布前启动项目
  - [Polygon zkEVM](https://polygon.technology/blog/to-ethereum-with-love-announcing-polygon-zkevm-mainnet-beta-on-march-27th)宣布将于 3 月 27 日发布 Beta 版
- [Optimism 推迟了 Bedrock 升级投票](https://twitter.com/optimismfnd/status/1626380025418113024)以解决赏金竞赛发现的问题
- [Coinbase](https://twitter.com/coinbaseassets/status/1626296045184376833)终于支持了 Arbitrum 网络上进行 ETH 和 DAI 的存取款
- Patrick McCorry：[去中心化 rollup ](https://stonecoldpat.substack.com/p/what-does-it-mean-to-decentralise)需要确保当系统受到对手攻击时，[一个诚实方可以将所有潜在的决定发送到合约桥](https://stonecoldpat.substack.com/p/where-is-the-one-honest-party-for)
- EF Layer 2 为 22 个项目[提供 94.8 万美元的资助](https://blog.ethereum.org/2023/02/14/layer-2-grants-roundup)
- [Arbitrum One](https://twitter.com/l2beat/status/1628356180769636353)在交易量上领先以太坊主网，Rollup 首次实现一天内超过主网交易量
- [Base](https://base.mirror.xyz/jjQnUq_UNTQOk7psnGBFOsShi7FlrRp8xevQUipG_Gk)：建立在 Optimism 的 OP Stack 上的 Coinbase 第 2 层，在测试网上运行，以 ETH 支付交易费用，其中一部分 sequencer 收入用于资助公共产品
- Optimism 的[超级链概念](https://stack.optimism.io/docs/understand/explainer/)：具有共享桥和通信层的 OP 链网络
- Patrick McCorry: [rollup 交易最终性](https://stonecoldpat.substack.com/p/tiers-of-transaction-finality-for) : sequencer 承诺提交, 交易最终排序 及 执行层确认执行  
- Scroll zkEVM [alpha 测试网](https://scroll.io/blog/alphaTestnet)
- Arbitrum 提出交易[时间加速](https://offchain.medium.com/time-boost-a-new-transaction-ordering-policy-for-arbitrum-5b3066382d62)替代先到先得(FCFS)，MEV 搜索者最多可以为交易购买0.5秒的提升
- Jordi Baylina：[证明者不是 zk rollup 可扩展性的瓶颈](https://twitter.com/jbaylina/status/1629352597445394432)，因为它们可以并行运行
- Patrick McCorry：在 rollup 之上的[链下系统（Layer 3）](https://stonecoldpat.substack.com/p/sharp-superchain-layer-3s-temporary)