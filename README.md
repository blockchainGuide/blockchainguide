# 🚀 死磕区块链 - 区块链技术学习导航

![GitHub stars](https://img.shields.io/github/stars/blockchainGuide/blockchainguide?style=social) ![GitHub forks](https://img.shields.io/github/forks/blockchainGuide/blockchainguide?style=social) ![文档数量](https://img.shields.io/badge/文档数量-280+-blue)

欢迎来到"死磕区块链"知识库！本仓库旨在构建一个**全面、系统化**的区块链技术学习体系，内容涵盖从入门到精通的各个阶段。无论你是初学者还是资深开发者，都可以在这里找到有价值的学习资源。

> 📢 **关注公众号 `链上无名`，获取最新资料。**
> 💬 **加入技术讨论群，请添加微信 `mindcarver`。**

---

## 📖 目录

- [📚 区块链基础](#-区块链基础-blockchain-basics)
- [⛓️ 公链开发](#️-公链开发-public-chain-development)
- [🎨 DApp 开发](#-dapp-开发-dapp-development)
- [⚡ Layer2 解决方案](#-layer2-解决方案-layer2-solutions)
- [🔐 隐私计算](#-隐私计算-privacy-computing)
- [🔗 跨链技术](#-跨链技术-cross-chain-technology)
- [📦 去中心化存储](#-去中心化存储-decentralized-storage)
- [🧭 学习路线与资源](#-学习路线与资源-learning-roadmaps--resources)
- [🤝 如何贡献](#-如何贡献)

---

## 📚 区块链基础 (Blockchain Basics)

万丈高楼平地起，这里是您进入区块链世界的第一站。

| 主题 | 描述 |
|------|------|
| [纠删码](./Blockchain_Basics/纠删码.md) | 分布式存储中的数据冗余与恢复技术 |

---

## ⛓️ 公链开发 (Public Chain Development)

深入探索公链的底层技术，包括共识机制、密码学、P2P网络等核心组件。

### 🔧 共识机制 (Consensus Mechanisms)

| 文档 | 描述 |
|------|------|
| [死磕共识算法_POW算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_pow算法-1.md) | 工作量证明机制详解 |
| [死磕共识算法_POS算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_POS算法-2.md) | 权益证明机制详解 |
| [死磕共识算法_DPOS算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_DPOS算法-4.md) | 委托权益证明机制 |
| [死磕共识算法_EOS_DPOS_BFT算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_EOS_DPOS_BFT算法-5.md) | EOS的混合共识机制 |
| [死磕共识算法_Paxos算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_Paxos算法-5.md) | 分布式一致性算法 |
| [死磕共识算法_Raft算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_Raft算法-6.md) | 易于理解的一致性算法 |
| [死磕共识算法_拜占庭将军问题](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_拜占庭将军问题-7.md) | 拜占庭容错基础理论 |
| [死磕共识算法_PBFT算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_PBFT算法-8.md) | 实用拜占庭容错算法 |
| [死磕共识算法_Istanbul BFT算法](./Public_Chain_Development/Consensus_Mechanisms/死磕共识算法_Istanbul%20BFT算法-9.md) | 以太坊的IBFT共识 |

### 🔐 密码学 (Cryptography)

<details>
<summary>点击展开详细内容</summary>

**基础理论**
- 密码学基础
- 数论基础 / 代数基础

**椭圆曲线密码学**
- 椭圆曲线理论与应用

**签名算法**
- BLS签名算法
- 多种数字签名方案

**隐私计算**
- VRF (可验证随机函数)
- VDF (可验证延迟函数)

</details>

### 🌐 P2P 网络 (P2P Network)

- libp2p 源码分析
- P2P 网络协议详解

### 🔬 公链研究 (Public Chain Research)

<details>
<summary>点击展开详细内容</summary>

**以太坊 (Ethereum)**
| 分类 | 内容 |
|------|------|
| 源码分析 | 以太坊核心模块源码解读、P2P网络分析 |
| 基础理论 | 以太坊基础概念、钱包系列 |
| 开发资料 | ERC标准整理、开发者资源汇总 |
| 安全性 | 以太坊安全机制研究 |
| 生态应用 | 应用整理、生态分析 |

**Cosmos 生态**
- Cosmos SDK 源码分析
- CometBFT 共识引擎
- IBC 跨链通信协议
- Tendermint 核心

**TON (The Open Network)**
- TON 开发指南
- TON 技术资料

**其他公链**
- Hyperledger Fabric (联盟链)

</details>

### 📊 其他专题

- 布隆过滤器
- RawDB 数据库
- 数据可用性 (DA) 探索

---

## 🎨 DApp 开发 (DApp Development)

学习如何构建去中心化应用，从智能合约开发到安全审计的完整流程。

### 📝 智能合约基础

| 主题 | 描述 |
|------|------|
| [Solidity call](./DApp_Development/合约基础/solidity_call.md) | call 函数详解 |
| [Solidity delegatecall](./DApp_Development/合约基础/solidity_delegatecall.md) | 委托调用机制 |
| [Solidity staticcall](./DApp_Development/合约基础/solidity_staticcall.md) | 静态调用 |
| [Solidity callcode](./DApp_Development/合约基础/solidity_callcode.md) | callcode 用法 |
| [CREATE2](./DApp_Development/合约基础/solidity_create2.md) | 可预测地址的合约部署 |
| [Selector](./DApp_Development/合约基础/solidity_selector.md) | 函数选择器 |
| [Fallback 和 Receive](./DApp_Development/合约基础/solidity_fallback和receive.md) | 回退函数机制 |
| [元交易](./DApp_Development/合约基础/元交易.md) | 无 Gas 交易实现 |

### 🚀 合约高级技巧

<details>
<summary>点击展开详细内容</summary>

**核心技巧**
- [死磕 Solidity 之内存布局](./DApp_Development/合约高级技巧/死磕solidity之内存布局.md)
- [死磕 Solidity 之如何有效节省 Gas](./DApp_Development/合约高级技巧/死磕solidity之如何有效的节省gas.md)
- [死磕 Solidity 之可迭代映射](./DApp_Development/合约高级技巧/死磕solidity之可迭代映射.md)
- [死磕 Solidity 之编写可升级合约](./DApp_Development/合约高级技巧/死磕solidity之编写可升级合约.md)
- [死磕 CREATE2 控制合约地址](./DApp_Development/合约高级技巧/死磕CREATE2控制合约地址.md)
- [合约升级实战](./DApp_Development/合约高级技巧/合约升级实战.md)
- [事件高级用法](./DApp_Development/合约高级技巧/事件高级用法.md)

**设计模式**
| 类别 | 模式 |
|------|------|
| 可维护性 | Contract Registry、Contract Relay、数据逻辑分离 |
| 安全性 | Mutex、Rate Limit、Speed Bump、安全转账 |
| 生命周期 | 允许合约自动停止、允许合约自毁 |

</details>

### 📋 EIP/ERC 标准

| 标准 | 描述 |
|------|------|
| [EIP-712](./DApp_Development/eip/EIP712.md) | 类型化结构化数据签名 |
| [EIP-1559](./DApp_Development/eip/EIP1559详解.md) | 交易费用市场改革 |
| [EIP-2718](./DApp_Development/eip/EIP2718详解.md) | 类型化交易信封 |
| [ERC-1167](./DApp_Development/eip/ERC1167详解.md) | 最小代理合约 |
| [ERC-1967](./DApp_Development/eip/ERC1967详解.md) | 代理存储槽标准 |
| [ERC-2535](./DApp_Development/eip/ERC2535详解.md) | 钻石标准 (多面代理) |
| [ERC-4337](./DApp_Development/EVM/账户抽象/ERC4337协议.md) | 账户抽象 |

### 🛡️ 安全审计

<details>
<summary>点击展开详细内容</summary>

**常见漏洞**
| 漏洞类型 | 描述 |
|----------|------|
| [重入攻击](./DApp_Development/安全审计/合约漏洞/重入攻击手段.md) | 递归调用攻击 |
| [重放攻击](./DApp_Development/安全审计/合约漏洞/重放攻击.md) | 交易重复执行 |
| [算法上下溢出](./DApp_Development/安全审计/合约漏洞/算法上下溢出.md) | 整数溢出问题 |
| [权限控制漏洞](./DApp_Development/安全审计/合约漏洞/权限控制漏洞.md) | 访问控制缺陷 |
| [短地址攻击](./DApp_Development/安全审计/合约漏洞/短地址参数攻击.md) | 参数截断攻击 |
| [拒绝服务攻击](./DApp_Development/安全审计/合约漏洞/拒绝服务攻击.md) | DoS 攻击 |
| [提案攻击](./DApp_Development/安全审计/合约漏洞/提案攻击.md) | 治理攻击 |
| [隐私数据泄露](./DApp_Development/安全审计/合约漏洞/不要在合约中存隐私数据.md) | 链上数据可见性 |

</details>

### 💰 DeFi 应用场景

<details>
<summary>点击展开详细内容</summary>

**Uniswap V2**
- [概念介绍](./DApp_Development/应用场景/defi/uniswap/v2/uniswap_v2_概念.md)
- [Factory 合约](./DApp_Development/应用场景/defi/uniswap/v2/uniswap_v2_Factory.md)
- [Pair 合约](./DApp_Development/应用场景/defi/uniswap/v2/uniswap_v2_Pair.md)
- [Router02 合约](./DApp_Development/应用场景/defi/uniswap/v2/uniswap_v2_Router02.md)
- [部署指南](./DApp_Development/应用场景/defi/uniswap/v2/uniswap_v2_部署.md)

**AAVE**
- [AAVE 原理](./DApp_Development/应用场景/defi/aave/Avae原理.md)

**其他**
- 永续期货合约
- 动态 NFT
- 数字版权保护

</details>

---

## ⚡ Layer2 解决方案 (Layer2 Solutions)

Layer2 是解决区块链扩展性问题的关键技术方案。

### Optimistic Rollup

| 项目 | 内容 |
|------|------|
| **Optimism** | Spec 分析、源码分析 (op-node) |
| **Arbitrum** | 技术架构与实现 |

### ZK Rollup

| 项目 | 内容 |
|------|------|
| **Polygon zkEVM** | ZK 证明系统与 EVM 兼容 |
| **ZK Rollup 通用** | 零知识证明在 Rollup 中的应用 |

### 其他方案

- **Celestia** - 模块化数据可用性层
- **Rollkit** - 主权 Rollup 框架
- **B2 Network** - 比特币 Layer2
- **BTC Layer2** - 比特币二层方案研究

---

## 🔐 隐私计算 (Privacy Computing)

隐私保护是区块链技术的重要组成部分。

### 零知识证明 (Zero-Knowledge Proofs)

- ZK 资料整理与学习路径

### 安全多方计算 (Secure Multi-Party Computation)

- MPC 协议与应用场景

---

## 🔗 跨链技术 (Cross-Chain Technology)

打破链与链之间的孤岛，实现价值和信息的自由流通。

### 跨链方案概览

| 文档 | 描述 |
|------|------|
| [跨链主流技术概览](./Cross_Chain_Technology/跨链方案/跨链主流技术概览.md) | 主流跨链技术对比分析 |
| [跨链 HTLC](./Cross_Chain_Technology/跨链方案/跨链_HTLC.md) | 哈希时间锁定合约 |
| [BLS 算法](./Cross_Chain_Technology/跨链方案/bls算法.doc) | BLS 签名在跨链中的应用 |

### 跨链桥项目

| 项目 | 描述 |
|------|------|
| [桥概念](./Cross_Chain_Technology/跨链方案/跨链桥/桥概念.md) | 跨链桥基础原理 |
| [Arbitrum Bridge](./Cross_Chain_Technology/跨链方案/跨链桥/abitrum.md) | Arbitrum 官方桥 |
| [Optimism Gateway](./Cross_Chain_Technology/跨链方案/跨链桥/optimism%20gateway%20.md) | Optimism 官方桥 |
| [Polygon Bridge](./Cross_Chain_Technology/跨链方案/跨链桥/polygon%20bridge.md) | Polygon 官方桥 |
| [cBridge](./Cross_Chain_Technology/跨链方案/跨链桥/cbridge.md) | Celer 跨链桥 |
| [Hop Protocol](./Cross_Chain_Technology/跨链方案/跨链桥/hop协议.md) | Hop 协议 |
| [Multichain](./Cross_Chain_Technology/跨链方案/跨链桥/multichain.md) | Multichain 跨链 |
| [Connext Protocol](./Cross_Chain_Technology/跨链方案/跨链桥/context协议.md) | Connext 跨链协议 |

---

## 📦 去中心化存储 (Decentralized Storage)

探索去中心化存储方案，构建真正去中心化的互联网。

### IPFS

- [IPFS 学习要点](./Decentralized_Storage/IPFS/IPFS学习点.md)
- libp2p 协议详解

### FileCoin

| 文档 | 描述 |
|------|------|
| [FileCoin 技术文档](./Decentralized_Storage/FileCoin/FileCoin技术文档.md) | 技术架构与实现 |
| [死磕 FileCoin 经济模型](./Decentralized_Storage/FileCoin/死磕FileCoin_经济模型.md) | 经济激励机制分析 |

---

## 🧭 学习路线与资源 (Learning Roadmaps & Resources)

我们为您精心规划了从初阶到高阶的学习路线图。

### 📍 学习路线图

| 阶段 | 文件 |
|------|------|
| 初阶 | `死磕区块链_初阶学习路线图.xmind` |
| 中阶 | `死磕区块链_中阶学习路线图.xmind` |
| 高阶 | `死磕区块链_高阶学习路线图.xmind` |

### 📚 学习资料

- [相关学习资料汇总](./Learning_Roadmaps_And_Resources/相关学习资料（路线图和资料持续更新，建议关注）.md) *(持续更新)*
- [智能合约学习路线与资源汇总](./DApp_Development/智能合约学习路线与资源汇总.md)

---

## 🗂️ 项目结构

```
blockchainguide/
├── Blockchain_Basics/          # 区块链基础
├── Public_Chain_Development/   # 公链开发
│   ├── Consensus_Mechanisms/   # 共识机制
│   ├── Cryptography/           # 密码学
│   ├── P2P_Network/            # P2P 网络
│   └── Public_Chain_Research/  # 公链研究 (ETH/Cosmos/TON)
├── DApp_Development/           # DApp 开发
│   ├── 合约基础/               # Solidity 基础
│   ├── 合约高级技巧/           # 高级技巧与设计模式
│   ├── eip/                    # EIP/ERC 标准
│   ├── 安全审计/               # 安全审计与漏洞
│   └── 应用场景/               # DeFi/NFT 应用
├── Layer2_Solutions/           # Layer2 方案
├── Privacy_Computing/          # 隐私计算 (ZK/MPC)
├── Cross_Chain_Technology/     # 跨链技术
├── Decentralized_Storage/      # 去中心化存储 (IPFS/FileCoin)
└── Learning_Roadmaps_And_Resources/  # 学习路线与资源
```

---

## 🤝 如何贡献

非常欢迎您为本知识库做出贡献！您可以通过以下方式参与：

1. **🐛 修正错误**：如果您发现任何错误或过时的内容，请随时提交 Pull Request
2. **📝 补充内容**：如果您有好的学习资料或文章，欢迎分享给我们
3. **💡 提出建议**：如果您对本知识库有任何建议，请通过 Issue 告诉我们
4. **⭐ Star 支持**：如果觉得有帮助，请给我们一个 Star

### 贡献指南

```bash
# 1. Fork 本仓库
# 2. 创建您的特性分支
git checkout -b feature/AmazingFeature

# 3. 提交您的更改
git commit -m 'Add some AmazingFeature'

# 4. 推送到分支
git push origin feature/AmazingFeature

# 5. 打开一个 Pull Request
```

---

## 📜 License

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情

---

<div align="center">

**让我们一起，死磕到底，共同打造最顶级的区块链技术学习资源！**

[![公众号](https://img.shields.io/badge/公众号-链上无名-green)](链上无名)
[![微信](https://img.shields.io/badge/微信-mindcarver-blue)](mindcarver)

</div>
