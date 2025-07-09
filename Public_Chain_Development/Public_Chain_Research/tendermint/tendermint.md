# What is Tendermint

组成： blockchain consensus engine + ABCI 接口

- consensus engine : 确保在每台机器上以相同的顺序记录相同的事务
- ABCI(Application Blockchain interface)：使交易能够用任何编程语言进行处理

## ABCI

application做以下事情：

- 维护数据库
- 验证交易签名
- 阻止双花

ABCI通过3种消息协调tendermint core 和 上层应用程序。

- **DeliverTx**：每个交易都必须通过DeliverTx消息进行传递。应用程序需要对接收到的每个带有DeliverTx消息的交易进行验证
- **CheckTx**：CheckTx消息与DeliverTx类似，但仅用于验证事务。Tendermint Core的内存池首先使用CheckTx检查事务的有效性，并仅将有效事务中继到其对等端。例如，应用程序可以检查事务中递增的序列号，如果序列号旧，则在CheckTx时返回错误
- **Commit**: commit hash 放在了下一个区块头（如何做到帮助捕获编程错误和简化轻量级客户端的开发？）


>- `CheckTx` 阶段会在交易广播到 Tendermint 节点时立即执行，用于快速验证交易的基本属性，比如交易的格式、签名、交易费用等是否正确。在这个阶段，交易不会对应用程序状态做出任何更改，只是对交易的基本属性进行验证。
>- `DeliverTx` 阶段会在交易被 Tendermint 打包成区块之后执行，用于对交易进行更加深入的验证，并将交易应用到应用程序状态中。在这个阶段，应用程序会检查交易是否符合应用程序协议和状态，并根据交易内容更新应用程序状态。



应用程序可以有多个 ABCI Socket 连接。Tendermint Core 会创建三个 ABCI 连接给应用程序：一个用于在 mempool 中广播交易时的验证，一个用于共识引擎运行区块提案，还有一个用于查询应用程序状态。

![image-20230516172751043](/Users/carver/Library/Application Support/typora-user-images/image-20230516172751043.png)

## Consensus Overview

Tendermint是一个易于理解的、主要是异步的BFT共识协议。该协议遵循一个简单的状态机，看起来如下：

![image-20230516173116158](/Users/carver/Library/Application Support/typora-user-images/image-20230516173116158.png)

## technical nutshell

Todo: 理一遍

![tx-flow](https://docs.tendermint.com/v0.34/assets/img/tm-transaction-flow.258ca020.png)

## tendermint stack

![cosmos-tendermint-stack](https://docs.tendermint.com/v0.34/assets/img/cosmos-tendermint-stack-4k.6aa56af6.jpg)

## tendermint core

### validator 

成为validator :

- genesis 
- ABCI 

## 代码架构

- reactor编程模型



```go
git checkout master
git tag -a v0.13.2 -m "upgrade to v0.13.2"
git push --tags ## 之后git action会自动触发 
```

