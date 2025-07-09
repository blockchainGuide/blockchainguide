1. 他的多链架构可以让一个应用程序在不同的IBC协调的链上并行运行，无论是否由相同的验证器集运行。这种跨链的水平可扩展性理论上允许无限的类似垂直的可扩展性，减去协调开销。

2. *Hub zone,我们如何将我们的链连接到非 Tendermint 链？*IBC 连接不限于基于 Tendermint 的链。如果另一个非 Tendermint 区块链使用快速最终共识算法，则可以通过使 IBC 适应非 Tendermint 共识机制来建立连接。

   如果另一条链是**概率确定性链**，则 IBC 的简单改编是不够的。称为**peg-zone**的代理链有助于建立互操作性。Peg-zones 是快速终结性区块链，它跟踪链状态以建立终结性。peg-zone 链本身是 IBC 兼容的，并充当IBC 网络其余链和概率最终链之间的**桥梁。**

3. CLI命令构建区块链

4. cosmos wasm 一个用于开发和测试智能合约的多链平台

5. Ethemint将EVM作为SDK模块

6. 质押atom 获取收益

7. [Althea Testnet 存储库：Gravity Bridge](https://github.com/gravity-bridge/gravity-bridge) （cosmos上面的跨链桥）， 将 Cosmos 与以太坊连接起来，并允许在基于 Cosmos 的区块链之间传输 ERC-20 代币。

8. 针对私有链之间的跨链通信https://www.hyperledger.org/blog/2021/06/09/meet-yui-one-the-new-hyperledger-labs-projects-taking-on-cross-chain-and-off-chain-operations

9. cosmos的相关所有库文件 https://github.com/cosmos/awesome-cosmos

10. IBC支持同构链和异构链，支持非最终一致性和最终一致性链互相跨链

11. https://youtu.be/OSMH5uwTssk 允许无数主权区块链之间的互操作性的方法以及如何构建与 IBC 兼容的应用程序进行了演讲。

12. https://youtu.be/zUVPkEzGJzA 他解释了如何使用区块链间通信 (IBC) 协议连接不同的区块链，特别关注轻客户端、连接、通道, 和数据包承诺

13. 跨链账户概念

14. https://youtu.be/X5mPQrCLLWE  他研究了 IBC 数据包生命周期和轻客户端的安全属性