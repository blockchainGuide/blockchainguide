# 预部署

**目录**

- [概述](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#overview)
- [遗留消息传递器](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#legacymessagepasser)
- [L2ToL1消息传递器](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l2tol1messagepasser)
- [部署者白名单](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#deployerwhitelist)
- [旧版ERC20ETH](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#legacyerc20eth)
- [韦斯9](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#weth9)
- [L2跨域信使](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l2crossdomainmessenger)
- [L2标准桥接器](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l2standardbridge)
- [L1区块编号](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l1blocknumber)
- [天然气价格甲骨文](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#gaspriceoracle)
- [L1区块](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l1block)
- [代理管理](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#proxyadmin)
- [排序器费用库](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#sequencerfeevault)
- [OptimismMintableERC20Factory](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#optimismmintableerc20factory)
- [OptimismMintableERC721Factory](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#optimismmintableerc721factory)
- [基本费用保险库](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#basefeevault)
- [L1收费金库](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#l1feevault)

## 概述

[预部署的智能合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#predeployed-contract-predeploy)存在于创世状态下预先确定的地址的 Optimism 上。它们与预编译类似，但直接在 EVM 中运行，而不是在 EVM 外部运行本机代码。

使用预部署而不是预编译，可以更轻松地实现多客户端，并允许与安全帽/铸造厂网络分叉进行更多集成。

预部署地址存在于 1 字节命名空间中`0x42000000000000000000000000000000000000xx`。代理在每个可能的预部署地址处设置，除了 `GovernanceToken`和`ProxyAdmin`。

预`LegacyERC20ETH`部署位于一个特殊地址`0xDeadDeAddeAddEAddeadDEaDDEAdDeaDDeAD0000` ，并且该帐户没有部署代理。

下表包括每个预部署。系统版本指示何时引入预部署。可能的值为`Legacy` 或`Bedrock`。不应使用已弃用的合同。

| 姓名                          | 地址                                        | 介绍 | 已弃用 | 代理 |
| ----------------------------- | ------------------------------------------- | ---- | ------ | ---- |
| 遗留消息传递器                | 0x4200000000000000000000000000000000000000  | 遗产 | 是的   | 是的 |
| 部署者白名单                  | 0x4200000000000000000000000000000000000002  | 遗产 | 是的   | 是的 |
| 旧版ERC20ETH                  | 0xDeadDeAddeAddEAddeadDEaDDDEAdDeaDDeAD0000 | 遗产 | 是的   | 不   |
| 韦斯9                         | 0x4200000000000000000000000000000000000006  | 遗产 | 不     | 不   |
| L2跨域信使                    | 0x4200000000000000000000000000000000000007  | 遗产 | 不     | 是的 |
| L2标准桥接器                  | 0x4200000000000000000000000000000000000010  | 遗产 | 不     | 是的 |
| 排序器费用库                  | 0x4200000000000000000000000000000000000011  | 遗产 | 不     | 是的 |
| OptimismMintableERC20Factory  | 0x4200000000000000000000000000000000000012  | 遗产 | 不     | 是的 |
| L1区块编号                    | 0x4200000000000000000000000000000000000013  | 遗产 | 是的   | 是的 |
| 天然气价格甲骨文              | 0x420000000000000000000000000000000000000F  | 遗产 | 不     | 是的 |
| 治理代币                      | 0x4200000000000000000000000000000000000042  | 遗产 | 不     | 不   |
| L1区块                        | 0x4200000000000000000000000000000000000015  | 基岩 | 不     | 是的 |
| L2ToL1消息传递器              | 0x4200000000000000000000000000000000000016  | 基岩 | 不     | 是的 |
| L2ERC721桥                    | 0x4200000000000000000000000000000000000014  | 遗产 | 不     | 是的 |
| OptimismMintableERC721Factory | 0x4200000000000000000000000000000000000017  | 基岩 | 不     | 是的 |
| 代理管理                      | 0x4200000000000000000000000000000000000018  | 基岩 | 不     | 是的 |
| 基本费用保险库                | 0x4200000000000000000000000000000000000019  | 基岩 | 不     | 是的 |
| L1收费金库                    | 0x420000000000000000000000000000000000001a  | 基岩 | 不     | 是的 |

## 遗留消息传递器

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/legacy/LegacyMessagePasser.sol)

地址：`0x4200000000000000000000000000000000000000`

该`LegacyMessagePasser`合约存储了基岩升级之前提款交易的承诺。提交提款交易的特定存储槽的默克尔证明被用作 L1 上提款交易的一部分。包含存储槽的预期帐户被硬编码到 L1 逻辑中。基岩升级后，`L2ToL1MessagePasser`改为使用。在基岩之后，将不再支持最终从该合同中撤回，并且仅允许可能依赖于它的替代桥梁。该合约不会将调用转发到 ，`L2ToL1MessagePasser`并且在通过系统进行提款的情况下调用它被视为无操作`CrossDomainMessenger` 。

任何尚未最终确定的待处理提款都会作为 `L2ToL1MessagePasser`升级的一部分迁移到 ，以便仍可以最终确定。

## L2ToL1消息传递器

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L2ToL1MessagePasser.sol)

地址：`0x4200000000000000000000000000000000000016`

本店`L2ToL1MessagePasser`承诺提现交易。当用户在 L1 上提交提款交易时，他们提供了一个证明，证明他们在 L2 上提款的交易位于`sentMessages` 该合约的映射中。

任何提取的 ETH 都会累积到 L2 上的此合约中，并且可以通过调用该`burn()`函数而无需许可地从 L2 供应中删除。

## 部署者白名单

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/legacy/DeployerWhitelist.sol)

地址：`0x4200000000000000000000000000000000000002`

这`DeployerWhitelist`是一种预部署，用于在 Optimism 的初始阶段提供额外的安全性。它之前定义了允许将合约部署到网络的帐户。

随后启用了任意合约部署，并且无法关闭。在遗留系统中，该合同被挂钩`CREATE`并 `CREATE2`确保部署者被列入允许名单。

在基岩系统中，该合约将不再用作 `CREATE`代码路径的一部分。

该合约已被弃用，应避免使用。

## 旧版ERC20ETH

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/legacy/LegacyERC20ETH.sol)

地址：`0xDeadDeAddeAddEAddeadDEaDDEAdDeaDDeAD0000`

预`LegacyERC20ETH`部署代表基岩升级之前系统中的所有以太币。所有 ETH 均表示为 ERC20 代币，用户可以选择使用 ERC20 接口或原生 ETH 接口。

升级到基岩版会将所有以太币从该合约中迁移出来，并将其移至其本机表示形式。该合约中的所有有状态方法都将在基岩升级后恢复。

该合约已被弃用，应避免使用。

## 韦斯9

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/vendor/WETH9.sol)

地址：`0x4200000000000000000000000000000000000006`

`WETH9`是 Wrapped Ether on Optimism 的标准实现。它是一种常用的合约，并被放置为预部署，以便它位于基于 Optimism 的网络中的确定性地址。

## L2跨域信使

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L2CrossDomainMessenger.sol)

地址：`0x4200000000000000000000000000000000000007`

与`L2CrossDomainMessenger`直接调用`L2ToL1MessagePasser`. 它维护已中继到 L2 的 L1 消息的映射，以防止重放攻击，并且如果 L1 到 L2 事务在 L2 上恢复，则还允许重放。

对 L1 上的任何调用`L1CrossDomainMessenger`都会进行序列化，以便它们通过`L2CrossDomainMessenger`L2 上的。

该`relayMessage`函数从远程域执行事务，同时该`sendMessage`函数通过远程域的函数发送要在远程域上执行的事务`relayMessage`。

## L2标准桥接器

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L2StandardBridge.sol)

地址：`0x4200000000000000000000000000000000000010`

这`L2StandardBridge`是一个建立在 之上的更高级别的 API `L2CrossDomainMessenger`，它提供了跨域发送 ETH 或 ERC20 代币的标准接口。

要将代币从 L1 存入 L2，需要`L1StandardBridge`锁定代币并向其发送跨域消息，`L2StandardBridge`然后将代币铸造到指定帐户。

要将代币从 L2 提取到 L1，用户将在 L2 上销毁代币，并向 L2 `L2StandardBridge`发送一条消息，该消息`L1StandardBridge`将解锁底层代币并将其转移到指定账户。

可`OptimismMintableERC20Factory`用于在远程域上创建 ERC20 代币合约，该合约映射到本地域上的 ERC20 代币合约，其中代币可以存入远程域。它部署了一个 `OptimismMintableERC20`具有与 `StandardBridge`.

该合约还可以部署在 L1 上，以允许将 L2 原生代币提取到 L1。

## L1区块编号

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/legacy/L1BlockNumber.sol)

地址：`0x4200000000000000000000000000000000000013`

返回`L1BlockNumber`最后一个已知的 L1 块号。`L1Block`该合约是在遗留系统中引入的，并且应该通过在后台调用该合约来向后兼容。

建议使用`L1Block`合约在L2上获取L1的信息。

## 天然气价格甲骨文

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/GasPriceOracle.sol)

地址：`0x420000000000000000000000000000000000000F`

在遗留系统中，这`GasPriceOracle`是一个经过许可的合约，由链外参与者推动 L1 基本费和 L2 天然气价格。链下参与者观察 L1 区块头以获得 L1 基本费用以及 L2 上的 Gas 使用情况，以根据拥塞控制算法计算 L2 Gas 价格。

在 Bedrock 之后，它`GasPriceOracle`不再是一个许可合约，其存在只是为了保留用于链下 Gas 估算的 API。该函数`getL1Fee(bytes)`接受未签名的 RLP 交易，并将返回费用的 L1 部分。该费用用于支付使用 L1 作为数据可用性层的费用，并且应添加到费用的 L2 部分（用于支付执行费用），以计算总交易费用。

用于计算费用的 L2 部分的值为：

- 标量
- 高架
- 小数点

基岩升级后，这些值改为由 `SystemConfig`L2 上的合约管理。和`scalar`值每个块`overhead`都会发送到合约，并且该值已硬编码为 6。`L1Block``decimals`

## L1区块

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L1Block.sol)

地址：`0x4200000000000000000000000000000000000015`

L1Block是在 Bedrock 中引入的[，](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-attributes-predeployed-contract)负责维护 L2 中的 L1 上下文。这允许在 L2 中访问 L1 状态。

## 代理管理

[代理管理](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/universal/ProxyAdmin.sol) 地址：`0x4200000000000000000000000000000000000018`

是`ProxyAdmin`预部署时设置的所有代理合约的所有者。它本身位于代理后面。所有者`ProxyAdmin`将能够升级任何其他预部署合同。

## 排序器费用库

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/SequencerFeeVault.sol)

地址：`0x4200000000000000000000000000000000000011`

累积`SequencerFeeVault`所有交易优先权费用，其值为 `block.coinbase`。当该账户中积累了足够的费用时，可以将其提取到不可变的 L1 地址。

要更改提取费用的 L1 地址，必须通过更改其代理的实现密钥来升级合约。

## OptimismMintableERC20Factory

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/universal/OptimismMintableERC20Factory.sol)

地址：`0x4200000000000000000000000000000000000012`

负责`OptimismMintableERC20Factory`在 L2 上创建 ERC20 合约，可用于存入原生 L1 代币。`StandardBridge`这些 ERC20 合约可以无需许可地创建，并实现仅处理存款和取款所需的接口。

创建的每个 ERC20 合约都`OptimismMintableERC20Factory`允许铸造`L2StandardBridge`和销毁代币，具体取决于用户是从 L1 存款到 L2 还是从 L2 取款到 L1。

## OptimismMintableERC721Factory

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/universal/OptimismMintableERC721Factory.sol)

地址：`0x4200000000000000000000000000000000000017`

负责`OptimismMintableERC721Factory`在 L2 上创建 ERC721 合约，可用于将原生 L1 NFT 存入其中。

## 基本费用保险库

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/BaseFeeVault.sol)

地址：`0x4200000000000000000000000000000000000019`

预`BaseFeeVault`部署在 L2 上接收基本费用。L2 上的基本费用不会像 L1 上那样被销毁。一旦合约收到一定数量的费用，ETH 就可以提取到 L1 上的不可变地址。

## L1收费金库

[执行](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L1FeeVault.sol)

地址：`0x420000000000000000000000000000000000001a`

预`L1FeeVault`部署接收交易费用的 L1 部分。一旦合约收到一定数量的费用，ETH 就可以提取到 L1 上的不可变地址。