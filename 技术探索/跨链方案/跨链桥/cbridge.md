## SGN

pos 链，对L1<->L2 交易进行共识，生成多重签名，表明跨链交易可行。

SGN作为cBridge节点网关和服务水平协议（SLA）仲裁器

![image-20230506100319074](/Users/carver/Library/Application Support/typora-user-images/image-20230506100319074.png)

SGN会监控cbride节点的状态和性能以及跨链陈功率等等指标，来将用户请求分发到合适的节点上， 对不能完成承诺的节点，进行惩罚

### SGN作为共享流动性池管理者

![image-20230506103544494](/Users/carver/Library/Application Support/typora-user-images/image-20230506103544494.png)



## SLA债券

你买的多，你作为节点被选中的概率大

## 节点选择规则

根据 SLA债券，节点响应时间，成功率 ，等作为因子来计算概率。

## cBridge

a: 在不跑节点的情况下提供流动性

流动性池合约

b:高流动性

流动性提供商需要放入规范练代币和另一个协议代币到链上AMM池

## 费用结构

xAsset 模型中桥接代币的费用计算如下：

 费用 = 基本费用 + 协议费

基本**费用**以代币转移的形式支付，包括将代币发送给用户的目标链 gas 成本。

协议**费用**与转账金额成正比，并支付给 State Guardian Network (SGN) 验证者和质押者以换取他们的服务。协议**费的**范围为总转账金额的 0% 至 0.5%

## 安全

两种安全模型：

根据延迟和安全假设不同

1. 来自optimism rollup的灵感
2. L1-PoS-blockchainL1-PoS-blockchain

## 端到端的工作流

https://im-docs.celer.network/developer/architecture-walkthrough/end-to-end-workflow