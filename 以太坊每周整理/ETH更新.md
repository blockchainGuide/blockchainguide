- [用 DHT+SkipGraph 解决链和状态数据检索问题](https://ethresear.ch/t/explorations-into-using-dht-skipgraph-for-chain-and-state-data-retrieval/7337)
- [Vitalik 写的 EIP1559 （手续费机制变更）FAQ](https://notes.ethereum.org/Wjr1SnW-QaST7phX9C5wkg?view)

- Péter Szilágyi 的 [snap sync 模式详述](https://github.com/ethereum/devp2p/blob/3fe9713658f3b3b56e4e99493c54f313e11b43a0/caps/snap.md)；[snap 对比 fast sync 的基准测试结果](https://twitter.com/peter_szilagyi/status/1263668104493662210)
- phase 0 spec [v0.12](https://github.com/ethereum/eth2.0-specs/releases/tag/v0.12.0)：加入了最新的 IETF 标准。这一版的 spec 就是 Eth2 Phase 0 启动时会用到的最终规范
- [Lighthouse 客户端更新](https://lighthouse.sigmaprime.io/update-25.html)：BLS 签名实现，已经过 Trail of Bits 审计，可使用 300MB 内存运行 2000 个验证者

- [用 Solidity 案例来解释 EIP2938 账户抽象](https://hackmd.io/@SamWilsn/ryhxoGp4D)
- Piper 撰写的 “[状态可得性](https://notes.ethereum.org/e8VFLDiUSPSn2v7VVM1CXw)” 文档

- [持久化储存 txpool 提案](https://gist.github.com/karalabe/821a1cd0270984a4198e904d34623b6c)

- [彻底改变无状态以太坊的路线图](https://ethresear.ch/t/complete-revamp-of-the-stateless-ethereum-roadmap/8592)

- [另一种对打包状态友好的地址方案](https://ethresear.ch/t/alternative-bounded-state-friendly-address-scheme/8602)

- [以太坊状态管理诸提议](https://hackmd.io/@HWeNw8hNRimMm2m2GH56Cw/state_size_management)

- [使用 Verkle tries 管理以太坊状态](https://notes.ethereum.org/_N1mutVERDKtqGIEYc-Flw)

- [一个可应用于不定型树状哈希数据结构的存储槽租金方案](https://gist.github.com/zsfelfoldi/a207d216b3fa9ae4be6abe7a5d8e68d8)

- [促进状态可得性的 GetNodeData DHT 方法](https://ethresear.ch/t/state-availability-getnodedata-dht-approach-dev-update/8657)

- [应用于轻客户端的事务 gossip 网络](https://ethresear.ch/t/scalable-transaction-gossip/8660)

- Vitalik 和 Micah 提议了[一种另类的状态收缩方法](https://ethresear.ch/t/resurrection-conflict-minimized-state-bounding-take-2/8739)，一种以一年为长度的 regenesis

- [将地址长度从 20 字节延长到 32 字节的提议](https://ethereum-magicians.org/t/increasing-address-size-from-20-to-32-bytes/5485)

- [弱无状态性 以及/或者 状态过期方案 即将到来](https://www.reddit.com/r/ethereum/comments/lw7ug3/weak_statelessness_andor_state_expiry_coming_soon/)

- [为什么我们应该把 verkle trees 和定期的状态灭活方案结合起来](https://hackmd.io/@HWeNw8hNRimMm2m2GH56Cw/state_expiry_paths)

- [通过 Flashbots 实现账户抽象？](https://github.com/flashbots/pm/issues/24)

- [提议将以太坊的状态树格式改为 verkle tree](https://www.reddit.com/r/ethereum/comments/mdbkp2/proposed_verkle_tree_scheme_for_ethereum_state/)

- [状态检索网络的一个初步规范](https://ethresear.ch/t/state-network-dht-development-update-2/9005)

- [提议将以太坊的状态树格式改为 verkle tree](https://www.reddit.com/r/ethereum/comments/mdbkp2/proposed_verkle_tree_scheme_for_ethereum_state/)

- [状态检索网络的一个初步规范](https://ethresear.ch/t/state-network-dht-development-update-2/9005)

- [启用 BLS12-381 支持的路径：EIP2537 和 EVM384](https://docs.google.com/document/d/1DdA1IxSDK5ppC1C2W9uvMW66Y8VToc-vPV5B68ZQvOg)

- [EVM384 进展](https://notes.ethereum.org/--JjliY8T_-qIdvAQKQlcg?view)：基准测试和预编译

- [提议使用 GASMETER 操作码来移除 GAS 操作码](https://ethresear.ch/t/new-opcode-gasmeter-as-a-step-towards-removing-gas-and-gas-observability/9067)

- [让 eip1559 更像一条 AMM 曲线](https://ethresear.ch/t/make-eip-1559-more-like-an-amm-curve/9082)（这是未来的目标，不会放在伦敦升级中）

- [无状态以太坊路线图更新](https://ethresear.ch/t/an-updated-roadmap-for-stateless-ethereum/9046)

- 无状态以太坊创意：[插入顺序索引型默克尔树](https://ethresear.ch/t/statelessness-by-insertion-order-indexed-merkle-tries/9095) 以及 [见证数据生成的贝叶斯网络模型](https://ethresear.ch/t/bayesian-network-model-of-witness-creation-feedback-request/9115)

- [状态存储和执行应该有相互独立的定价吗？](https://ethresear.ch/t/should-storage-be-priced-separately-from-execution/9101)[为什么我们需要提高状态访问的 gas 开销？](https://www.reddit.com/r/ethereum/comments/mrl5wg/a_quick_explanation_of_what_the_point_of_the_eip/)

- [FuzzyVM](https://mariusvanderwijden.github.io/blog/2021/05/02/FuzzyVM/)：一种 EVM 模糊测试框架

- [切换到 Verkle 树之后的 witness gas 消耗量设计提案](https://notes.ethereum.org/@vbuterin/witness_gas_cost_2)

- [为什么要给状态访问操作码提高 Gas 消耗量](https://blog.ethereum.org/2021/05/18/eth_state_problems/)

- [给钱包开发者和用户的 EIP1559 说明](https://hackmd.io/@q8X_WM2nTfu6nuvAzqXiTQ/1559-wallets)

- “伦敦” 分叉测试网 Baikal [现已在区块浏览器上显示 EIP-1559 交易数据](https://twitter.com/timbeiko/status/1395416223395975169)

- Beiko 的[核心开发者系列更新](https://hackmd.io/@timbeiko/acd/https%3A%2F%2Fhackmd.io%2F%40timbeiko%2Facd-update-003)

- [Turbo-Geth 项目重命名为 Erigon](https://twitter.com/ErigonEth/status/1394273529613389825)，还计划将 C++ 和 Rust 语言实现的客户端分别命名为 Silkworm 和 Akula

- Vitalik：[粉尘账户清理办法](https://ethereum-magicians.org/t/some-medium-term-dust-cleanup-ideas/6287)

- Geth 的[默认同步模式从快速同步切换为快照同步](https://github.com/ethereum/go-ethereum/pull/22973)（[11 小时 vs. 2 小时](https://blog.ethereum.org/2021/03/03/geth-v1-10-0/)）

- “伦敦” 升级的开发者网络命名为 “[Calaveras](https://github.com/ethereum/eth1.0-specs/blob/master/network-upgrades/client-integration-testnets/calaveras.md)”，修复了 Martin Swendo 发现的一些问题

- [分析为见证数据的每个代码片收取 350 Gas 的效能](https://notes.ethereum.org/@ipsilon/code-chunk-cost-analysis)

- [使用 verkle tree 的 Geth 概念验证开发者网络](https://twitter.com/gballet/status/1400815481569923075)

- Vitalik 提议使用[单层的 Verkle 树](https://www.reddit.com/r/ethereum/comments/nve036/ethmag_proposed_scheme_for_encoding_ethereum/)（而不是当前提议的、在树结构中安排双层树的做法）

- 提议使用[预编译来抽象历史和分片证据的验证方法](https://ethresear.ch/t/future-proof-shard-and-history-access-precompiles/9781)，以允许未来更改格式

- [提议使用另类的交易池（比如 Flashbots）来实现账户抽象](https://notes.ethereum.org/@vbuterin/alt_abstraction)

- [Piper 提议使用一个可执行的 markdown 格式规范来替代晦涩的黄皮书](https://ethereum-magicians.org/t/replace-the-yellow-paper-with-executable-markdown-specification/6430)

- [状态过期会议](https://consensysmesh.zoom.us/rec/play/VoftjeRO0xNFHObLZpot-r0hTj0U6ZG0fUKORwQzy2qoh-JlweRgL6hj5UxqC8WsdYW3IOVBc6l_912R.ND8fJB672c-EpG1T?_x_zm_rhtaid=872&_x_zm_rtaid=Jhbf_Q6fTx-tPJB7vFllMg.1623863913448.e6b0045131eec8ea5666c579c3b613b7&autoplay=true&continueMode=true&startTime=1623848674000)解读了 Vitalik 的[状态过期和无状态性路线图](https://notes.ethereum.org/@vbuterin/verkle_and_state_expiry_proposal)：每年一次状态过期、仅要求区块生产者存储状态，其它节点可以是无状态的

- Vitalik 解释 [Verkle tree](https://vitalik.ca/general/2021/06/18/verkle.html)：Verkle tree 使得证明可以小于 150 字节，使无状态客户端成为可行

- [Trin （Rust 语言的 portal network 客户端）更新](https://snakecharmers.ethereum.org/trin-development-update/)：全功能的 JSON RPC 轻客户端，正在与其他客户端沟通，下一步是传输数据

- [EIP3074 的替代方案以及批评](https://ethereum-magicians.org/t/a-case-for-a-simpler-alternative-to-eip-3074/6493)

- [解释 EVM 的对象格式](https://notes.ethereum.org/@ipsilon/evm-object-format-overview)

- [分析合约中的 memory 复制，以及使用 MCOPY 操作码的提议](https://notes.ethereum.org/@ipsilon/evm-mcopy-analysis)

- [使用 EVM 对象格式实现账户抽象](https://notes.ethereum.org/@axic/rybPKSz2_)

- （状态保质期方案）使用[带前缀的地址时期](https://ethereum-magicians.org/t/simple-non-address-length-extending-address-periods/6536)替代当前的延长地址（从 20 字节延长到 32 字节）方案

- Vitalik 的 [无状态性、沃克尔树和状态保质期 AMA](https://www.reddit.com/r/ethereum/comments/o9s15i/impromptu_technical_ama_on_statelessness_and/)

- [SELFDESTRUCT](https://hackmd.io/@albus/HJ6EBiTn_) 在沃克尔树方案（已移除大部分的自毁功能）下效果的初步分析

- 使用 EIP3074，将 EOA 迁移到[基于验证的智能合约钱包](https://ethereum-magicians.org/t/validation-focused-smart-contract-wallets/6603)
  [Spreadsheet](https://docs.google.com/spreadsheets/d/1Ld4JSyaz-gvTx-4xIaxe4qU2wlaG1XAQfMiKl9CKCY0/edit#gid=0) 展示 EIP1559 机制下 baseFeePerGas 的波动速度

- [evmodin](https://github.com/vorot93/evmodin)：Rust 语言的 EVM 实现，使用 evmone（C++ 语言） 作为接口

- [使用桥合约的地址空间拓展（ASE）](https://notes.ethereum.org/@axic/B1BUs526u)，作为使用转换映射的 ASE 的替代

- [提议使用一套覆盖层协议来建立和管理一个可使用任意数量子协议的覆盖网络](https://notes.ethereum.org/tPzmxQD_S3S3uvtpUSA0-g)

- [链历史存储网络](https://hackmd.io/ctTNH9xsSu2ci9DeGidUsQ?view)首个规范草案：提供历史区块头和区块体的按需获取

- [状态网络的存储格式提议](https://notes.ethereum.org/h58LZcqqRRuarxx4etOnGQ)

- 提议使用 [EEICALL 操作码](https://ethresear.ch/t/eeicall-opcode-to-execute-bytecode-over-given-eei/10161)以在给定的执行环境接口下执行字节码

- [EIP3709](https://github.com/ethereum/EIPs/blob/d67798b2646da04dfa82dd1443cc7c0a6d90e60e/EIPS/eip-3709.md)：弃用类型 1 的交易（即 “伦敦” 分叉以前以太坊使用的交易格式）

- [Verkle树的影响](https://docs.google.com/document/d/1s3qqzbkQFPcNvhzKPdnxg3MlFbv0YjK1z02SxRtdMs8/edit)对现有合约的影响，~26%的平均gas增长

- Geth 正在考虑放弃对 [archive nodes](https://twitter.com/peter_szilagyi/status/1436226033934606338) 的支持

- [强化 BLS12-381 椭圆曲线](https://blog.ethereum.org/2021/09/09/secured-no-1/)

- 验证者[隐私消息分享协议](https://ethresear.ch/t/private-message-sharing-for-eth2-validators/10664)，为了隐私性以及抵抗洪泛攻击

- Geth [v1.0.9](https://github.com/ethereum/go-ethereum/releases/tag/v1.10.9)：修复了一些 bug，弃用 eth/65 联网协议

- [EIP4361](https://github.com/ethereum/EIPs/blob/27b5497268bab4449cbe815ae9812388005d763b/EIPS/eip-4361.md)：使用以太坊账户来登录。项目网站：[login.xyz](https://login.xyz/)

- [EIP4363](https://github.com/ethereum/EIPs/blob/556533cbe1257d570da179a20ccf1f2dcd4ff224/eip-4363.md)：交易索引操作码

- [无状态以太坊资源表](https://notes.ethereum.org/@gballet/Sy-a6T5St)：沃克尔树开发进度、地址空间扩展以及状态网络

- **请立即升级 Geth 客户端**：v1.10.9 以前的版本披露了[安全漏洞](https://github.com/ethereum/go-ethereum/security/advisories/GHSA-59hh-656j-3p7v)，可通过构造 p2p 消息发起 DoS 攻击

- 关于精简默克尔树[ gas 计算](https://notes.ethereum.org/@vbuterin/verkle_write_gas_extension)的提议

- Geth [v1.10.15](https://github.com/ethereum/go-ethereum/releases/tag/v1.10.15):解bug，点对点eth网络可以上锁

- Besu [v21.10.6](https://github.com/hyperledger/besu/releases/tag/21.10.6): 更新解决log4j的问题

- 有些交易所仍然[没有实现EIP1559](https://twitter.com/0xtrent/status/1476666765111439362)类型2交易

- Vitalik的多维EIP1559协议，从EVM执行，交易调用数据，存证数据和存储大小增长开始为每个资源创建突发限制

- [EIP1559 使用分析](https://arxiv.org/abs/2201.05574): 费用估算限制更容易，区块间的 gas 价格波动减少，用户等待时间减少，MEV 在矿工收入的占比更大

- Flashbots 研究: [并行 EVM](https://writings.flashbots.net/research/speeding-up-evm-part-1/), 同时执行无存储冲突交易，通过可选的访问列表进行预加载存储

- Erigon [v2022.01.03](https://github.com/ledgerwatch/erigon/releases/tag/v2022.01.03): 改进了交易池中的分类，实验发行及销毁跟踪功能，bug修复。

- 提议[主网的 rollup gas 归入 layer2 消息](https://ethereum-magicians.org/t/charging-rollup-gas-for-l1-to-l2-messages-extractgas-or-payfromorigin/8104)

- 执行层的[执行规范](https://ethereum.github.io/execution-specs/)预览版

- [包含Verkle证明的样本区块](https://github.com/gballet/verkle-block-sample/#readme) ，以及解码、验证这个区块的实用程序

- [Mini-danksharding 原型](https://twitter.com/protolambda/status/1495538286332624898) (data-blob-transaction): 在ETHDenver共同开发, 下一步是开发网和EIP草案

- 使用检查点同步的[Nimbus迁移指南](https://twitter.com/ethnimbus/status/1496404992575610880)

- Vouch [v1.4.0](https://github.com/attestantio/vouch/releases/tag/v1.4.0) (多节点验证)增加支持Nimbus

- Vitalik: [数据可用性抽样，可以用IPAs替换KZG吗](https://ethresear.ch/t/what-would-it-take-to-do-das-with-inner-product-arguments-ipas/12088)

  - Kiln 测试网成功过渡到 POS ，存在一些问题，需要更多的测试，包括开发网和fork主网
  - EIP4895 推送提现作为上海升级的选择
  - EIP4844 数据blob交易类型更新
  - [核心 EIP 流程与可执行规范协调一致](https://notes.ethereum.org/@timbeiko/executable-eips)的提案

- Erigon [v2022.03.01](https://github.com/ledgerwatch/erigon/releases/tag/v2022.03.01): 修复漏洞

- Besu [v22.1.2](https://github.com/hyperledger/besu/releases/tag/22.1.2): 支持 Kiln v2.1 规范，跟踪 API 改进

- PluGeth [Parity 跟踪插件](https://blog.openrelay.xyz/parity-trace/): 4 种相当于 OpenEthereum 的追踪方法

- EIP4844 数据 blob 交易类型 [meta-spec](https://hackmd.io/@protolambda/eip4844-meta) 和 [推广网站](https://www.eip4844.com/)

- Dankrad 的 [EIP1559 指数版本解释器](https://dankradfeist.de/ethereum/2022/03/16/exponential-eip1559.html), 针对数据blob 交易的提议

- 如果在2TB SSD上运行，[质押者应该修剪 Geth 节点](https://www.reddit.com/r/ethstaker/comments/tgs9qy/if_you_are_running_your_node_on_a_2tb_ssd_its_a/)

- 关于账户的需求：[将 EOA 迁移到合约钱包的选项](https://ethresear.ch/t/a-brief-note-on-the-future-of-accounts/12395)

- - 关于 RPC 安全/不安全/最新标签的讨论
  - MEV boost: validators/proposers to set gas limit rather than builders
  - MEV 升级：验证者/提议者设置gas限制，而不是建设者

- Erigon [v2022.04.04-alpha](https://github.com/ledgerwatch/erigon/releases/tag/v2022.04.04)：通过 BitTorrent 提高快照文件下载速度的解决方法

- [Verkle 树迁移](https://notes.ethereum.org/@XwIwCeCIQMeAl00PUAbEDw/verkle-migration-sketch)草图

- [NiceNode](https://mirror.xyz/johnsgresham.eth/BqQ92jtwu3hl6Ri2-giRLd0hequOt8Ya6ULAyCR-3ls)：在 Linux、Mac 或 Windows 上运行 Geth 节点的 alpha 接口

- [EIP4844 交易验证提速](https://github.com/ethereum/EIPs/pull/5088) (用 KZG 证明), 验证时间减少到 3.5ms

- 了解 Verkle 树中的密码学: [bandersnatch 和 banderwagon](https://hackmd.io/@6iQDuIePQjyYBqDChYw_jg/BJ2-L6Nzc)的区别

- [MEV-boost](https://writings.flashbots.net/writings/why-run-mevboost/) 是一个新的中间件，在这个中间件中，验证者不仅可以向Flashbots出售他们的区块空间，还可以向其他构建者出售。这为更多的构建者打开了市场，并在他们之间创造了竞争，为验证者带来了更多的收入和选择。

- 共识层客户端的

  性能测试

  - TLDR: 对比了6个以太坊共识层客户端: Lighthouse, Lodestar, Nimbus, Prysm, Teku 和Grandine

- 个人质押者的[合并升级清单](https://www.coincashew.com/coins/overview-eth/ethereum-merge-upgrade-checklist-for-home-stakers-and-validators)

- 如果你觉得你的[验证者被DOS攻击](https://ethereum-magicians.org/t/how-i-learned-to-stop-worrying-about-the-dos-and-love-the-chain/9941)，那么运行一个岗哨节点直到以太坊发布单一秘密领导选举协议(SSLE)来修复问题

- [Constantine](https://our.status.im/fastest-bls-signature-implementation/) BLS 实现，相比BLST，签名速度快 14% ，验证速度快 18%

- 提议支持[基于中间件的分布式验证客户端](https://ethresear.ch/t/distributed-validator-middlewares-and-the-aggregation-duty/13044)

- Vitalik 关于[调整内存 gas 成本的建议](https://notes.ethereum.org/@vbuterin/proposals_to_adjust_memory_gas_costs)

- MEV-boost [信息网站](https://boost.flashbots.net/)

- Obol的[Athena](https://blog.obol.tech/the-athena-testnet/)公共测试网——分布式验证者中间件客户端

- Teku [v22.8.0](https://github.com/ConsenSys/teku/releases/tag/22.8.0): MEV-boost 支持, libp2p 及 分叉选择优化

- Prysm [v2.1.4-rc.1](https://github.com/prysmaticlabs/prysm/releases/tag/v2.1.4-rc.1): 支持 Goerli 合并

- [Flashbots](https://writings.flashbots.net/writings/understanding-mev-boost-liveness-risks/) 为MEV-boost构建了一个[中继监视器](https://hackmd.io/@ralexstokes/SynPJN_pq)和断路器，以防遭遇区块扣留攻击

- 用阀值加密[删除 MEV-boost 中的可信中继](https://ethresear.ch/t/removing-trusted-relays-in-mev-boost-using-threshold-encryption/13449)的提议

- [view-merge](https://ethresear.ch/t/view-merge-as-a-replacement-for-proposer-boost/13739)代替 propose boost ，防御balancing & ex-ante攻击

- [抗审查列表](https://notes.ethereum.org/@fradamt/H1ZqdtrBF)（crList）提案，创建者被迫充分使用区块空间，否则他们必须在未使用的空间中包含提案人选择的交易

- 共识层[视频会议](https://www.youtube.com/watch?v=_yogw67HxZY&t=298s)。[来自Christine Kim](https://twitter.com/christine_dkim/status/1573105520080289792)的笔记，11 月 3 日下一次 CL 电话会议

- Lodestar [v1.1.0](https://github.com/ChainSafe/lodestar/releases/tag/v1.1.0)：稳定性改进，为每个验证者配置提案人元数据

- Nimbus [v22.9.1](https://github.com/status-im/nimbus-eth2/releases/tag/v22.9.1)：修复了几个报告的问题和小的性能改进

- Teku [v22.9.1](https://github.com/ConsenSys/teku/releases/tag/22.9.1)：性能改进

- 共识规范[v1.2.0](https://github.com/ethereum/consensus-specs/releases/tag/v1.2.0)：主网 Bellatrix 规范、提款和 EIP4844 研发

- MEV[订单流拍卖](https://collective.flashbots.net/t/order-flow-auctions-and-centralisation-ii-order-flow-auctions/284)，以解决独家订单流

- 具有对数同步时间的[超轻客户端](https://twitter.com/dionyziz/status/1572068211465519108)，假设连接到一个诚实的完整节点

- 通过ERC4337 + EIP3074 + EIP5003 +[交易包含列表](https://notes.ethereum.org/@vbuterin/account_abstraction_roadmap#Transaction-inclusion-lists)实现[帐户抽象](https://twitter.com/vitalikbuterin/status/1576199517434949634)

- EIP4844 (proto-danksharding):[KZG的算术哈希替代方案](https://ethresear.ch/t/arithmetic-hash-based-alternatives-to-kzg-for-proto-danksharding-eip-4844/13863)是可能的，但需要进行很多权衡

- Besu [v22.7.5](https://github.com/hyperledger/besu/releases/tag/22.7.5): 修复空块提案和 RPC 缺陷

- Erigon [v2.27.0](https://github.com/ledgerwatch/erigon/releases/tag/v2.27.0): 实验性[嵌入式共识层客户端](https://giulioswamp.substack.com/p/erigon-embedded-consensus-module)，更改为[语义版本](https://erigon.substack.com/p/big-release-and-renaming-of-erigon)

- Nethermind [v1.14.3](https://github.com/NethermindEth/nethermind/releases/tag/1.14.3): 减少错过证明；及即[将到来的新特性](https://twitter.com/m25marek/status/1578057260248465408)

- 建议在 eth_getLogs 返回的日志中[添加区块时间戳](https://ethereum-magicians.org/t/proposal-for-adding-blocktimestamp-to-logs-object-returned-by-eth-getlogs-and-related-requests/11183)

- 利用边缘执行成本估算gas成本的[第二阶段研究成果](https://github.com/imapp-pl/gas-cost-estimator/blob/master/docs/gas-cost-estimator.md)

- 通过[限制构建者的权力](https://ethresear.ch/t/how-much-can-we-constrain-builders-without-bringing-back-heavy-burdens-to-proposers/13808)来降低中心化风险，并通过部分块拍卖(包含列表或后提议者)最小化区块提议者的责任。

- MEV-Boost 开发过程[讨论](https://collective.flashbots.net/t/toward-an-open-research-and-development-process-for-mev-boost/464)

- Flashbots [计划实现去中心化的区块构建](https://twitter.com/bertcmiller/status/1577482296629739520)

- Jon Charbonneau: Flashbot 可以通过开源它们的生成器或添加一个上限来[减少审查](https://joncharbonneau.substack.com/p/censorship-wat-do)

- Nimbus [v22.10.0](https://github.com/status-im/nimbus-eth2/releases/tag/v22.10.0): 出块更快，增加了指标

- ethStaker [Goerli 测试网验证器押金](https://goerli.launchpad.ethstaker.cc/en/)更新为只需要 0.0001 GoETH

- [Prysmatic Labs（Prysm 团队） 被 Offchain Labs（Arbitrum 团队） 收购](https://offchain.medium.com/the-merge-2-0-offchain-labs-acquires-prysmatic-labs-the-leading-ethereum-consensus-client-team-9eab169c5fb6)

- [MEV-Boost 发展理念](https://collective.flashbots.net/t/mev-boost-development-philosophy/505)

- [MEV Watch](https://www.mevwatch.info/) 的US OFAC审查增加了最后100个区块的可视化

- 轻客户端代理; 将钱包连到本地 RPC, 对最新的区块头使用证明来验证调用:

  - [Nimbus](https://twitter.com/jcksie/status/1580201162846146565) 轻代理
  - [Kevlar](https://github.com/shresthagrawal/kevlar#readme) CLI 工具运行一个基于客户端的轻RPC代理，用 Lodestar API

- Nimbus [v22.10.1](https://github.com/status-im/nimbus-eth2/releases/tag/v22.10.1): 支持轻客户端 REST API ，提高使用外部块构建时的稳定性

- Teku [v22.10.1](https://github.com/ConsenSys/teku/releases/tag/22.10.1): 修bug, 优化 ， `voluntary-exit` 命令改进

- Ben Edgington 的升级以太坊电子书，带注释的 [Bellatrix 版](https://eth2book.info/bellatrix/)

- Barnabe: [protocol-enforced proposer commitments](https://ethresear.ch/t/unbundling-pbs-towards-protocol-enforced-proposer-commitments-pepc/13879?u=barnabe) (PEPC) 可能替代 PBS

- [EIP4844（proto-danksharding）](https://github.com/ethereum/pm/blob/master/Breakout-Room/4844-readiness-checklist.md)准备就绪的清单

- [EVM Object Format (EOF)](https://twitter.com/lightclients/status/1593270266909450241) EIP 解释

- 最新共识层

  - [MEV-Boost 更新](https://hackmd.io/kJQguDvTRXGY4qK0z_j1gA)：Flashbot不再是[Top Builder](https://www.relayscan.io/) 
  - 取款：关于设置一个约束避免扫描整个验证者集的讨论
  - EIP4844 blob 待验证，将在测试网上检查
  - 提议将 getCapabilities 添加到 Engine API 并改进规范结构

- 共识规范[v1.3.0-alpha.1](https://github.com/ethereum/consensus-specs/releases/tag/v1.3.0-alpha.1)：Capella 和 EIP4844 改进，为开发测试网做好准备

- Flashbot [区块构建器开源](https://writings.flashbots.net/open-sourcing-the-flashbots-builder/)

- MEV-Boost 中继[v0.14.0](https://github.com/flashbots/mev-boost-relay/releases/tag/v0.14.0)：修复 DoS 漏洞

- Etherscan（测试版） 显示 [每个区块的](https://twitter.com/etherscan/status/1593204861969264640) MEV 信息：包括 proposer fee recipient 和 MEV reward，

- [整个信标链历史的ERA 文件](https://mainnet.era.nimbus.team/)（区块和共识数据的平面存储格式）

- [提款](https://twitter.com/terencechain/status/1599781623687700482)对构建层的影响：中继和构建者只需要运行正确的客户端版本

- 统一的[EOF 规范](https://notes.ethereum.org/@ipsilon/eof1-unified-specification)

- [EOF 的好处](https://twitter.com/leonardoalt/status/1600845724618326016)：节省gas、提高了安全性、更简单和更容易调试

- EIP4844 实施者[视频会议](https://www.youtube.com/watch?v=2lSGS9weOv0)和[笔记](https://twitter.com/terencechain/status/1600195265700339713)

- SELFDESTRUCT[删除分析](https://hackmd.io/X-waAY49SrW9i36SKOVuGQ)

- [Reth](https://www.paradigm.xyz/2022/12/reth)（用 Rust 写的 EL 客户端）：开源，预计第一季度支持完全同步

- Endpoint提案，以便[DVT 验证器客户端可以支持委员会聚合](https://blog.obol.tech/committee-aggregations-with-distributed-validators/)

- Verkle 树[PoS 测试网](https://twitter.com/gballet/status/1600088800943710209)（Beverly Hills)）

- Protolambda 的[PoS 规范发布和测试网历史](https://twitter.com/protolambda/status/1599437869994496000)

- 用 Geth + Prysm[运行 EIP4844 本地开发网](https://hackmd.io/q1SLCaubTIWw_1zsEjW_Vg?view)

- EIP1153 临时存储（ transient storage ）[解释](https://etherworld.co/2022/12/13/transient-storage-for-beginners/)

- [Lido 计划将 MEV-Boost 中继列表委托](https://research.lido.fi/t/lido-on-ethereum-identify-and-constitute-relay-maintenance-committee/3386)给一个委员会

- [Reconnaissance](https://github.com/Will-Smith11/reconnaissance#readme)：用 Reth 客户端的代理节点

- Vitalik 的 [EOF 提案：禁止 EOF 账户代码自省](https://ethereum-magicians.org/t/eof-proposal-ban-code-introspection-of-eof-accounts/12113) : 自动将代码转换到最新的 EVM 版本并使添加 EVM 功能更容易

- [
  TX-Fuzz](https://github.com/MariusVanDerWijden/tx-fuzz#readme) v1.0.0: 创建随机交易，用于破解 ”提款开发测试网 devnet-0

  Afri: [配置一个网络从创世开始使用PoS](https://dev.to/q9/how-to-merge-an-ethereum-network-right-from-the-genesis-block-3454), 使用了 Geth + Lodestar

  Lido: 在测试网上 [试验与Obol网络进行分布式验证器技术（DVT）](https://blog.lido.fi/dvt-pilot-with-obol-network/)

  [Gasper（以太坊权益证明协议）的演变](https://github.com/ethereum/pos-evolution/blob/master/pos-evolution.md)

  - [ERC4337 更新](https://twitter.com/yoavw/status/1608637361570877440)（使用 alt mempool 的帐户抽象），捆绑参考实现、兼容性测试套件
  - Remco：[交易内存上限](https://xn--2-umb.com/22/eth-max-mem/index.html)，32MB 3000 万 gas

- [核心开发者和共识层会议重命名](https://github.com/ethereum/pm#ethereum-project-management-repository)为 core devs call execution (ACDE)和 consensus (ACDC)

- [EOF v2](https://notes.ethereum.org/@ipsilon/eof2-proposal) 提案，包括 Vitalik 关于禁止代码自省的想法

- [Mevboost.pics](https://twitter.com/nero_eth/status/1609491027043225600) 随着时间的推移增加区块生成器插槽共享

- ERC4337（账户抽象）去中心化内存池[工作组笔记](https://github.com/JohnRising/4337-bundler-working-group/blob/main/20230104 Meeting Notes.md)

- EF Research AMA

  - 各种主题：[不仅仅是 4844 的需求](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3lsbja/)，技术解决方案[阻止了审查的尝试](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3m2z23/)，以太坊[有什么用](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3lz3iv/)？[就路线图优先级](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3lppce/)达成共识，[为 zkEVM 的最小化 EVM 变化](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3t8ab5/)，以太坊应该[死板点吗](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3pqz7i/)？跨 zkrollups的[同步可组合性](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3uge1d/)以及[Robust Incentives Group 正在做什么？](https://old.reddit.com/r/ethereum/comments/107cqi8/ama_we_are_ef_research_pt_9_11_january_2023/j3visx8/)

- Lido [benchmarks DVT](https://twitter.com/irina_everstake/status/1612536005135044609)（DVT 去中心化质押池）

- [Flashbots privacy roast（隐私出块）](https://youtu.be/Di5fO99lCPo?t=643)

- Prestwich：[MEV 的未来 5 年](https://medium.com/@Prestwich/mev-the-next-five-years-63f84fffdf36)

- [Crazy frontrun](https://twitter.com/bertcmiller/status/1613566397954621442)：抢跑者（frontrunner）将套利者（exploiter）的 遍布在超 50 个区块 的 4 笔交易进行了复制

- 

- - 独特的熵生成器：[猫证明(Proof-of-Cat)](https://proofof.cat/)和[弹珠机（marble run）](https://twitter.com/Xofee3/status/1618406659659038720)
  - [用 Go 实现的 Towers-of-pau](https://github.com/dknopik/towers-of-pau/tree/proper-client#readme)

- Nimbus 中[EIP4844 解耦 gossip（分发网络层） ](https://notes.status.im/decoupled-eip4844#)的设计笔记

- 提案[交易 SSZ 重构](https://notes.ethereum.org/@vbuterin/transaction_ssz_refactoring)

- [EOF v2 设计](https://notes.ethereum.org/@ipsilon/eof2-design-space)，基于上个月的讨论

- 
  通过中继从构建器到验证器的[MEV-boost 区块流](https://nerolation.github.io/mevflow.html)的可视化

  Flashbots：[为 MEV-Share 设计](https://collective.flashbots.net/t/mev-share-programmably-private-orderflow-to-share-mev-with-users/1264)，用户将交易发送给 matchmaker，用来匹配为使用其交易而付费的搜索者

  - [取款凭证的可视化](https://twitter.com/Data_Always/status/1629210968025714690)（0x00 与 0x01）
  - [Beverly Hills](https://twitter.com/jasoriatanishq/status/1627692040015478785)（Verkle 树）测试网现在是多客户端，Nethermind 与 Geth 同步
  - Reth (执行客户端)[模块化 p2p 架构](https://fiber.chainbound.io/blog/reth-p2p/)可作为独立组件使用

  - 执行层客户端多样性：[Nethermind 客户端超过 20% 的同步节点](https://twitter.com/ant_sabado/status/1630732757012717570)
  - EF [账户抽象赞助](https://esp.ethereum.foundation/account-abstraction-grants)，高达 30 万美元，截止日期为 3 月 31 日
  - Flashbot：
    - [用多方计算尾随隐私交易](https://writings.flashbots.net/backrunning-private-txs-MPC)，概念证明
    - 在 Sepolia 测试网上，[SGX enclave 内运行的区块生成器](https://writings.flashbots.net/block-building-inside-sgx)
  - 提议[将 blob 从执行负载中解耦](https://ethereum-magicians.org/t/uncouple-blobs-from-the-execution-payload/13059)