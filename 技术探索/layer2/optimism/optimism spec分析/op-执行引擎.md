# L2执行引擎

**目录**

- 存入交易处理
  - [存入交易边界](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#deposited-transaction-boundaries)
- 费用
  - [费用金库](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#fee-vaults)
  - [优先费（Sequencer Fee Vault）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#priority-fees-sequencer-fee-vault)
  - [基本费用（基本费用保险库）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#base-fees-base-fee-vault)
  - [L1-成本费（L1 Fee Vault）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#l1-cost-fees-l1-fee-vault)
- 引擎API
  - `engine_forkchoiceUpdatedV1`
    - [扩展有效负载属性V1](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#extended-payloadattributesv1)
  - [`engine_newPayloadV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine_newpayloadv1)
  - [`engine_getPayloadV1`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#engine_getpayloadv1)
- [联网](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#networking)
- 同步
  - [快乐路径同步](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#happy-path-sync)
  - [最坏情况同步](https://github.com/ethereum-optimism/optimism/blob/develop/specs/exec-engine.md#worst-case-sync)

本文档概述了 L2 的 L1 执行引擎的修改、配置和使用。

## 存入交易处理

引擎接口使用[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)抽象出事务类型。

为了支持汇总功能，新存款的处理[`TransactionType`](https://eips.ethereum.org/EIPS/eip-2718#transactions) 由引擎实现，请参阅[存款规范](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md)。

此类交易可以铸造L2 ETH，运行EVM，并在执行状态将L1信息引入到铭记的合约中。

### 存入交易边界

交易不能盲目信任，信任是通过身份验证建立的。与其他交易类型不同，存款不通过签名进行身份验证：汇总节点在引擎外部对它们进行身份验证。

为了安全地处理存款交易，必须首先对存款进行身份验证：

- 通过可信引擎 API 直接摄取
- 部分同步到受信任的块哈希（通过之前的引擎 API 指令受信任）

存入的交易绝不能从交易池中消耗。 *可以在仅存款汇总中禁用交易池*

## 费用

顺序交易（即不适用于存款）收取 3 种费用：优先费、基本费和 L1 成本费。

### 费用金库

出于会计目的，这三种类型的费用在 3 个不同的 L2 费用库部署中收集：费用支付不会注册为内部 EVM 调用，因此通过这种方式可以更好地区分。

这些是硬编码地址，指向预先部署的代理合约。这些代理由金库合约部署支持，基于`FeeVault`，将金库资金安全地路由到 L1。

| 仓库名称     | 预部署                                                       |
| ------------ | ------------------------------------------------------------ |
| 定序器费用库 | [`SequencerFeeVault`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#SequencerFeeVault) |
| 基本费用金库 | [`BaseFeeVault`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#BaseFeeVault) |
| L1 费用金库  | [`L1FeeVault`](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md#L1FeeVault) |

### 优先费（Sequencer Fee Vault）

优先费遵循[eip-1559](https://eips.ethereum.org/EIPS/eip-1559)规范，由 L2 区块的费用接收方收取。区块费用接收者（又名 coinbase 地址）设置为 Sequencer Fee Vault 地址。

### 基本费用（基本费用保险库）

基本费用很大程度上遵循[eip-1559](https://eips.ethereum.org/EIPS/eip-1559)规范，但基本费用不会被销毁，而是会累加到基本费用 Vault ETH 账户余额中。

### L1-成本费（L1 Fee Vault）

该协议通过根据估计的批量提交成本向 L2 用户收取额外费用，为排序的 L2 交易的批量提交提供资金。该费用从 L2 交易发送者 ETH 余额中收取，并收集到 L1 费用库中。

用于确定 L2 交易的 L1 成本费用部分的确切 L1 成本函数计算如下： `(rollupDataGas + l1FeeOverhead) * l1Basefee * l1FeeScalar / 1000000` （big-int 计算，结果以 Wei 和`uint256`范围表示）其中：

- ```
  rollupDataGas
  ```

  

  由完整

  编码交易确定（标准 EIP-2718 交易编码，包括签名字段）：

  - 在 Regolith 分叉之前：

    ```
    rollupDataGas = zeroes * 4 + (ones + 68) * 16
    ```

    - `68`非零字节的添加是 Bedrock 之前的 L1 成本核算功能的残余，它考虑了最坏情况的非零字节添加以补充未签名的交易，与 Bedrock 不同。

  - 使用 Regolith 叉子：`rollupDataGas = zeroes * 4 + ones * 16`

- `l1FeeOverhead`是 Gas Price Oracle`overhead`值。

- `l1FeeScalar`是 Gas Price Oracle`scalar`值。

- `l1Basefee`是在L2链上注册的最新L1源的L1基本费用。

请注意，它使用与[eip-2028](https://eips.ethereum.org/EIPS/eip-2028)`rollupDataGas`中定义的相同的字节成本核算，但完整的 L2 事务现在计入 L1 呼叫数据中收取的字节数。此行为与 Bedrock 之前对 L2 交易的 L1 成本估计相匹配。

`overhead`批量交易的压缩、批量和内在 Gas 成本由具有 Gas Price Oracle和参数的协议计算`scalar`。

Gas Price Oracle`l1FeeOverhead`和`l1FeeScalar`以及`l1Basefee`L1 来源的 可以通过两种可互换的方式访问：

- 从当前 L2 块的存放的 L1 属性 ( `l1FeeOverhead`, `l1FeeScalar`, ) 中读取`basefee`

- 从 L1 区块信息合约中读取 ( 

  ```
  0x4200000000000000000000000000000000000015
  ```

  )

  - 使用相应的 Solidity `uint256`-getter 函数 ( `l1FeeOverhead`, `l1FeeScalar`, `basefee`)
  - 使用直接存储读取：
    - `uint256`L1 基本费用作为槽中的大尾数法`1`
    - `uint256`槽中作为大端的开销`5`
    - `uint256`标量作为槽中的大尾数法`6`

## 引擎API

### `engine_forkchoiceUpdatedV1`

这会更新引擎认为规范的 L2 块（`forkchoiceState`参数），并可选择启动块生成（`payloadAttributes`参数）。

在汇总中，forkchoice 更新的类型转换为：

- `headBlockHash`：规范链头部的区块哈希。在用户 JSON-RPC 中标记`"unsafe"`。节点可以提前在带外应用 L2 块，然后在 L1 数据冲突时重新组织。
- `safeBlockHash`：规范链的区块哈希，源自L1数据，不太可能重组。
- `finalizedBlockHash`：不可逆的区块哈希，匹配争议期的下边界。

为了支持汇总功能，引入了一项向后兼容的更改[`engine_forkchoiceUpdatedV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#engine_forkchoiceupdatedv1)：扩展`PayloadAttributesV1`

#### 扩展有效负载属性V1

[`PayloadAttributesV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#PayloadAttributesV1)扩展为：

```
PayloadAttributesV1: {
    timestamp: QUANTITY
    random: DATA (32 bytes)
    suggestedFeeRecipient: DATA (20 bytes)
    transactions: array of DATA
    noTxPool: bool
    gasLimit: QUANTITY or null
}
```



这里使用的类型表示法是指[以太坊 JSON-RPC API 规范](https://github.com/ethereum/execution-apis)使用的[十六进制值编码](https://eth.wiki/json-rpc/API#hex-value-encoding)，因为该结构需要通过 JSON-RPC 发送。指的是 JSON 数组。`array`

数组的每一项`transactions`都是编码交易的字节列表：`TransactionType || TransactionPayload`或`LegacyTransaction`，如[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)中定义。这相当于`transactions`中的字段[`ExecutionPayloadV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#ExecutionPayloadV1)

该`transactions`字段是可选的：

- 如果为空或缺失：引擎行为不会发生变化。排序器（如果启用）将通过消耗交易池中的交易来构建区块。
- 如果存在且非空：必须从这个确切的事务列表开始生成有效负载。[Rollup 驱动程序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md)根据确定性 L1 输入确定事务列表。

也是`noTxPool`可选的，并扩展了`transactions`含义：

- 如果`false`，执行引擎可以在任何`transactions`. 这是 L1 节点实现的默认行为。
- 如果`true`，则执行引擎不得更改有关给定列表的任何内容`transactions`。

如果该`transactions`字段存在，引擎必须按顺序执行事务，`STATUS_INVALID` 如果处理事务出错则返回。`STATUS_VALID`如果所有事务都可以无错误地执行，则它必须返回。**注意**：状态转换规则已修改，存款永远不会失败，因此如果`engine_forkchoiceUpdatedV1`返回`STATUS_INVALID`，则说明批量交易无效。

`gasLimit`与 L1 的兼容性是可选的，但在用作汇总时是必需的。该字段覆盖块构建期间使用的气体限制。如果未指定为汇总，`STATUS_INVALID`则返回 a。

### `engine_newPayloadV1`

没有对[`engine_newPayloadV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#engine_newPayloadV1). 将 L2 块应用于发动机状态。

### `engine_getPayloadV1`

没有对[`engine_getPayloadV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#engine_getPayloadV1). 按 ID 检索有效负载，由`engine_forkchoiceUpdatedV1`调用时准备`payloadAttributes`。

## 联网

执行引擎可以通过rollup节点获取所有数据，正如源自L1： *P2P网络是严格可选的。*

然而，为了不成为 L1 数据检索速度的瓶颈，应该启用 P2P 网络功能，服务于：

- 对等发现（[光盘 v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md)）

- `eth/66`

  ：

  - 交易池（由排序器节点消耗）
  - 状态同步（快速无信任数据库复制的快乐路径）
  - 历史区块头和区块体检索
  - *新区块通过共识层获取（rollup 节点）*

除了配置之外，无需修改 L1 网络功能：

- [`networkID`](https://github.com/ethereum/devp2p/blob/master/caps/eth.md#status-0x00)：区分 L2 网络与 L1 和测试网。等于[`chainID`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)rollup 网络的 。
- 激活合并分叉：启用引擎 API 并禁用块的传播，因为没有共识层就无法对块头进行身份验证。
- Bootnode列表：DiscV5是共享网络， 通过先连接L2节点[引导速度更快。](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-rationale.md)

## 同步

执行引擎可以通过不同的方式操作同步：

- Happy-path：rollup节点将L1确定的所需链头告知引擎，通过引擎P2P完成。
- 最坏情况：汇总节点检测到停滞的引擎，纯粹从 L1 数据完成同步，不需要对等点。

happy-path 更适合让新节点快速上线，因为引擎实现可以通过[snap-sync](https://github.com/ethereum/devp2p/blob/master/caps/snap.md)等方法更快地同步状态。

### 快乐路径同步

1. rollup 节点无条件地通知 L2 链头的引擎（常规节点操作的一部分）：
   - [`engine_newPayloadV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#engine_newPayloadV1)使用从 L1 派生的最新 L2 块进行调用。
   - [`engine_forkchoiceUpdatedV1`](https://github.com/ethereum/execution-apis/blob/769c53c94c4e487337ad0edea9ee0dce49c79bfa/src/engine/specification.md#engine_forkchoiceupdatedv1)// 使用当前L2 块哈希`unsafe`值进行调用 。`safe``finalized`
2. 引擎向同级请求标头，反向请求直到父哈希与本地链匹配
3. 引擎赶上：a）一种形式的状态同步被激活以实现最终或头块哈希b）一种形式的块同步将块体和进程拉向头块哈希

精确的基于 P2P 的同步超出了 L2 规范的范围：引擎内的操作与 L1 完全相同（尽管使用支持存款的 EVM）。

### 最坏情况同步

1. 引擎不同步、未对等和/或由于其他原因而停止。
2. 汇总节点维护来自引擎的最新头（轮询`eth_getBlockByNumber`和/或维护头订阅）
3. 如果引擎不同步但未通过 P2P 同步，则汇总节点会激活同步 ( `eth_syncing`)
4. Rollup 节点逐一插入从 L1 派生的块，可能适应 L1 重组，如[Rollup 节点规范](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md)中所述( `engine_forkchoiceUpdatedV1`, `engine_newPayloadV1`)